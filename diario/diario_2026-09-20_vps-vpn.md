# Diario de laboratorio — 2026-09-20

**Proyecto:** Lab VPN + Ofuscación en VPS (Oracle Cloud, Ampere A1, Ubuntu 24.04 aarch64)
**Fase del plan:** Fase 0 (preparación del entorno) — completada. Fase 1 (WireGuard) — teoría completada, implementación pendiente.

---

## Objetivo de la sesión

Cerrar la Fase 0 del lab VPN (hardening del VPS) y avanzar en la Fase 1 (WireGuard), priorizando entender la teoría de red antes de tocar la implementación práctica.

---

## Lo que se hizo

### 1. Hardening de acceso root
- Se confirmó que `PermitRootLogin` estaba en su valor por defecto comentado (`prohibit-password`): root solo puede entrar con llave SSH, nunca con contraseña.
- Diferencia clave aprendida: `prohibit-password` (root permitido solo con llave) vs `no` (root deshabilitado por completo, obliga a usar un usuario con `sudo`). Se decidió mantener `prohibit-password`.

### 2. Firewall del sistema operativo (ufw)
- Se descubrió que `ufw` estaba activo con política `deny incoming` por defecto, **sin ninguna regla explícita de excepción**, ni siquiera para SSH.
- La sesión SSH activa seguía funcionando porque ya era una conexión establecida, pero cualquier conexión nueva se habría colgado sin respuesta.
- Se agregó la regla `ufw allow OpenSSH` a tiempo, verificando con una segunda terminal antes de confiar en el cambio.

### 3. Cambio de puerto SSH (22 → puerto personalizado, ver nota de privacidad)
Checklist de las **cinco capas** que había que sincronizar:
1. `sshd_config` (`Port`)
2. `ssh.socket` (vía el generador automático de systemd, `sshd-socket-generator`)
3. `fail2ban` (`jail.local`, línea `port`)
4. `ufw` (regla nueva + borrar la vieja)
5. Oracle Cloud — Security List (regla de ingress)

**Errores encontrados en el camino (y su resolución):**
- Un guion suelto (`-`) en `sshd_config` rompía la sintaxis — detectado con `sshd -t`.
- `/run/sshd` no existía (directorio tmpfs, se limpia en cada boot) → `sshd.service` fallaba al arrancar. Se recreó con `mkdir -p -m 0755 /run/sshd`.
- Se intentó un override manual de `ssh.socket` vía `systemctl edit` sin necesidad — Ubuntu ya regenera automáticamente las direcciones del socket a partir de las líneas `Port` en `sshd_config`. El override manual coexistiendo con el generado automáticamente causó una entrada `ListenStream` duplicada y el socket falló con `Failed with result 'resources'`. Se resolvió eliminando el override manual.
- Trampa de `systemctl edit`: solo lo escrito **arriba** del marcador "Edits below this comment will be discarded" se guarda; lo de abajo es de solo lectura y se descarta en silencio si se edita ahí.
- `systemctl status` mostró información desactualizada (un archivo ya borrado seguía apareciendo como Drop-In) hasta correr `daemon-reload` — lección: verificar con evidencia directa (`ls`, `ss`), no solo con el reporte del comando de estado.

### 4. fail2ban
- Se corrigió `jail.local` para que fuera un archivo mínimo de overrides (no una copia completa de `jail.conf`).
- Se activó el jail `sshd` (`enabled = true`, `backend = systemd`, `maxretry = 5`, `findtime = 10m`, `bantime = 1h`).
- Se verificó funcionamiento con `fail2ban-client status sshd`.

### 5. Verificación final
- Conexión de prueba exitosa al nuevo puerto desde una terminal fresca, sin cerrar la sesión de trabajo, antes de eliminar cualquier acceso al puerto 22.
- Limpieza completa del puerto 22 en las cinco capas, confirmada paso a paso.

### 6. Teoría de WireGuard (Fase 1)
Se investigó y se afianzó con preguntas dirigidas:
- **Generación de llaves:** `wg genkey` / `wg pubkey`, guardadas en `/etc/wireguard`.
- **`AllowedIPs`** — significa cosas distintas según el lado:
  - Del lado del servidor: control de acceso + ruta hacia ese peer.
  - Del lado del cliente: qué tráfico va por el túnel. `0.0.0.0/0, ::/0` = túnel completo (todo el tráfico sale por el VPS); una subred específica = túnel dividido.
- **`net.ipv4.ip_forward=1`** — habilita que el kernel actúe como router entre interfaces; sin esto, nada de lo demás funciona.
- **MASQUERADE (NAT de origen)** — necesario únicamente cuando el paquete cruza hacia una red que no reconoce las IPs privadas de la VPN (internet real). No hace falta para tráfico dentro de la propia subred del túnel, porque ambos extremos ya la reconocen como propia.
- **`PostUp` / `PostDown`** en `wg0.conf` — atan las reglas de NAT/forward al ciclo de vida de la interfaz WireGuard, evitando reglas "fantasma" si la VPN se detiene.

---

## Pendiente para la próxima sesión

- Instalación de WireGuard en el VPS (confirmar si el kernel `6.17.0-1020-oracle` ya trae el módulo nativo).
- Generación de llaves servidor/cliente.
- Armar `wg0.conf` completo (túnel completo, `PostUp`/`PostDown` con nftables).
- Levantar la interfaz, conectar un cliente, y correr la prueba básica de fuga de DNS/IP (Fase 3 del documento del lab).
- Fase 2 (Shadowsocks) queda para más adelante, no se tocó hoy.

---

## Nota de privacidad
La IP pública del VPS y el número de puerto SSH configurado se omiten deliberadamente de este diario (reemplazados por referencias genéricas) para no dejar esos datos en texto plano en el repositorio.
