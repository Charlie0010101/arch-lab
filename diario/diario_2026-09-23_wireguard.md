# Diario de laboratorio — 2026-09-23

**Proyecto:** Lab VPN + Ofuscación en VPS (Oracle Cloud, Ampere A1, Ubuntu 24.04 aarch64)
**Fase del plan:** Fase 1 (WireGuard) — implementación completada y verificada sin fugas. Pendiente: kill switch antes de Fase 2 (Shadowsocks).

---

## Objetivo de la sesión

Implementar WireGuard de extremo a extremo: llaves, configuración de servidor y cliente, y verificar que el túnel funciona en la práctica (no solo en teoría).

---

## Lo que se hizo

### 1. Confirmación de soporte nativo
- `modprobe wireguard` sin errores, módulo cargado con sus dependencias (`libcurve25519_generic`, `ip6_udp_tunnel`, `udp_tunnel`).
- `wg-tools` v1.0.20210914 ya disponible tras el `apt install` de la sesión anterior.

### 2. Generación de llaves
- Par de llaves del servidor y del cliente generados con `wg genkey | tee <archivo> | wg pubkey | tee <archivo>`.
- Error de sintaxis corregido en el camino: `tee` necesita una ruta de **archivo**, no solo el directorio; y cada llave necesita su propio nombre de archivo (si no, la segunda sobreescribe a la primera).
- Permisos verificados: `tee` heredó automáticamente `600` del `umask` del directorio padre (`/etc/wireguard` con `700`) — no hizo falta `chmod` manual.
- Aprendizaje de buenas prácticas: idealmente la llave privada del **cliente** se genera directamente en el dispositivo cliente, no en el servidor, para que nunca tenga que "viajar". Para este lab (aprendizaje, no producción) se generó en el servidor por simplicidad y luego se transfirió.
- Transferencia de la llave privada del cliente al equipo cliente vía `scp`, con paso intermedio obligatorio porque el usuario SSH permitido no es root (`AllowUsers` en `sshd_config`): copiar a `/tmp` con el dueño correcto, transferir, y `shred -u` en ambas copias del servidor tras confirmar que llegó bien.

### 3. Configuración del servidor (`/etc/wireguard/wg0.conf`)
- `Address`, `ListenPort` (UDP, puerto personalizado — ver nota de privacidad), bloque `[Peer]` con la llave pública del cliente y `AllowedIPs` como IP única (`/32`) — significado de ACL + ruta hacia ese peer específico.
- `PostUp`/`PostDown` con reglas nftables para MASQUERADE (tabla `nat`, cadena `postrouting`).
- Validado con `wg-quick strip wg0` antes de levantar.
- Interfaz levantada con `wg-quick up wg0` sin errores; confirmado con `wg show`, `ip addr show wg0`, y `nft list ruleset`.

### 4. Apertura de puertos (patrón ya conocido de la sesión anterior, aplicado a un puerto UDP nuevo)
- `ufw allow <puerto>/udp` en el servidor.
- Regla de ingress UDP nueva en Oracle Cloud Security List (mismo flujo que con SSH, cambiando el protocolo a UDP).

### 5. Configuración del cliente
- Instalación de `wireguard-tools` vía `pacman` (CachyOS, no `apt`).
- `wg0.conf` del cliente con `AllowedIPs = 0.0.0.0/0, ::/0` (túnel completo) y `PersistentKeepalive = 25` — repaso de por qué existe cada uno:
  - `AllowedIPs` del lado cliente decide qué tráfico va por el túnel (túnel completo vs. dividido); del lado servidor es una ACL + ruta hacia ese peer específico.
  - `PersistentKeepalive` evita que el NAT del router del cliente olvide la conexión activa (UDP no tiene estado persistente como TCP).
- Interfaz levantada con `wg-quick up wg0`; primer handshake confirmado con `wg show`.

### 6. Diagnóstico y corrección de fuga de tráfico (después del cierre inicial de la sesión)
El handshake funcionaba, pero verificar la IP pública real mostraba que el tráfico seguía saliendo por la red local, no por el túnel. Dos causas raíz encontradas y corregidas:

**Causa 1 — Cliente:** en algún punto la interfaz quedó levantada sin pasar por `wg-quick` (confirmado con `ps aux | grep wg-quick` vacío), por lo que nunca se crearon las reglas de enrutamiento por política (`ip rule`, tabla de fwmark) que traducen `AllowedIPs = 0.0.0.0/0` en tráfico real por el túnel. Solución: `ip link delete wg0` + `wg-quick up wg0` de nuevo. Verificado con `ip route get <ip-de-prueba>` mostrando la ruta vía `wg0`.

**Causa 2 — Servidor:** ufw tiene una cadena `FORWARD` con política propia (`drop` por defecto), **separada** de `INPUT`. Aunque el NAT/MASQUERADE ya estaba bien configurado, los paquetes se descartaban en `FORWARD` antes de llegar a esa regla. Solución:
- `DEFAULT_FORWARD_POLICY="ACCEPT"` en `/etc/default/ufw`.
- Reglas explícitas: `ufw route allow in on wg0 out on <interfaz-wan>` y la inversa (más seguro que un ACCEPT global).
- Confirmado `net.ipv4.ip_forward=1` en `/etc/ufw/sysctl.conf`.
- Recarga de ufw (`disable` + `enable`).

**Verificación final:** herramienta de verificación de IP pública mostrando la IP y ubicación del servidor VPS, no las del ISP local. Fuga resuelta, sin cortes de internet con la VPN activa.

---

## Pendiente para la próxima sesión

- **Kill switch en nftables**: si el túnel se cae inesperadamente, bloquear todo el tráfico en vez de caer de vuelta a la red local sin protección.
- Después del kill switch: arrancar Fase 2 del lab (Shadowsocks + posible plugin de ofuscación) para comparar tráfico con/sin ofuscación ante inspección de paquetes.
- Evaluar agregar un segundo peer (celular) una vez que el kill switch esté probado — requiere su propio par de llaves y su propia entrada `[Peer]`.

---

## Nota de privacidad
La IP pública del VPS, el puerto UDP de WireGuard, la ubicación real, el ISP y el nombre del controlador de red local se omiten deliberadamente de este diario para no dejar esos datos en texto plano en el repositorio.
