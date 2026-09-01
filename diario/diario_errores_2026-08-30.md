# Diario de Errores — Arch Linux Lab

## Sesión: 30 de agosto 2026

---

### Contexto de la sesión

Cierre del Módulo 04 (Bash & dotfiles) y arranque completo del Módulo 05 (Redes, SSH y firewall). Además, reorganización de repos de GitHub y decisión sobre uso de un servidor Oracle Cloud recién adquirido.

---

### 1. Decisión: distro base

**Situación:** Se intentó usar Hyprland sobre Arch, con fallos recurrentes de login y sistema roto. Se consideró migrar a CachyOS u Omarchy con Hyprland preinstalado.

**Resolución:** Se decidió volver a Arch + KDE Plasma (instalado vía `archinstall` esta vez, no manual, dado que el proceso manual ya estaba dominado). Razón: el problema no era la distro base sino la combinación Hyprland + NVIDIA híbrida + Wayland, que se replicaría igual en cualquier distro. Además, usar una distro con dotfiles preconfigurados iría en contra del objetivo de aprendizaje del proyecto.

**Pendiente a futuro:** Retomar Hyprland como experimento aparte (VM o instalación paralela), no como daily driver del lab.

---

### 2. Reorganización de repos de GitHub

**Situación:** El repo `dotfiles` mezclaba configs de Hyprland con scripts generales (`update-dotfiles.sh`), sin separación de propósitos.

**Decisión:** Separar en repos por propósito:
- `dotfiles` → queda exclusivo para configs de Hyprland
- `Scripts` (nuevo) → scripts generales de bash
- Pendiente crear un tercer repo (`arch-lab` o similar) para config específica del sistema (`.bashrc`, futuras configs de módulo 05+)

**Proceso ejecutado (sin errores):**
1. Clon de prueba en carpeta separada (`dotfiles-split-test`), nunca sobre el repo real.
2. `mkdir scripts` + `git mv update-dotfiles.sh scripts/` + commit.
3. `git subtree split --prefix=scripts --branch=scripts-only` — generó rama con historial filtrado.
4. Creación de repo vacío `scripts` en GitHub (sin README).
5. `git remote add` + `git push scripts-repo scripts-only:main` — exitoso.
   - Nota: GitHub normalizó el nombre a `Scripts` (mayúscula) y avisó del cambio de ubicación vía mensaje `remote: This repository moved`. El push funcionó igual por redirección automática.
6. Limpieza: se clonó `dotfiles` de nuevo, se eliminó `update-dotfiles.sh` del original con `git rm` + commit + push.
7. Borrado de carpetas de trabajo temporales (`dotfiles-split-test`, `dotfiles-real`).

**Resultado:** Repos separados correctamente, historial conservado, sin duplicados.

---

### 3. Configuración de autenticación con GitHub (sistema nuevo)

**Situación:** Sistema Arch reinstalado desde cero, sin credenciales de git configuradas.

**Pasos ejecutados:**
- `git config --global user.name` / `user.email`
- `git config --global credential.helper store`
- Generación de un Personal Access Token en GitHub con **todos los permisos marcados** (de más para el caso de uso).

**Nota de seguridad pendiente:** Se identificó que el token generado tiene permisos excesivos (borrado de repos, config de organizaciones, Actions, etc.) para lo que realmente se necesita. **Pendiente:** regenerar token con scope mínimo (`repo` únicamente en token clásico, o permisos específicos en fine-grained) y revocar el actual. Charly se comprometió a hacerlo esta semana.

---

### 4. Módulo 05 — Instalación y hardening de SSH

**Situación inicial:** `openssh` ya estaba instalado (probablemente parte de la instalación base), pero el servicio no estaba activo.

**Errores encontrados y resueltos:**

| Error | Causa | Resolución |
|---|---|---|
| `pacman -Qg openssh` → "group not found" | `openssh` es un paquete, no un grupo | Usar `pacman -Qi` o simplemente `-S` para confirmar/instalar |
| `pacman -y openssh` / `sudo pacman -Y` | Flags inválidos, error de sintaxis | Uso correcto: `sudo pacman -S openssh` |
| `ssh enable-now`, `ssh start` | Confusión entre comando `ssh` (cliente) y gestión de servicios | Se aclaró que la gestión de servicios va por `systemctl`, no por `ssh` |
| `systemctl start ssh` / `systemctl start openssh` → "Unit not found" | Nombre incorrecto del servicio | El servicio correcto es **`sshd.service`**, no `ssh` ni `openssh` |
| `systemctl start sshd.service` sin `sudo` | Posible fallo silencioso por permisos | Se resolvió con `sudo systemctl enable --now sshd.service` |

**Confusión conceptual aclarada:** Diferencia entre paquete `openssh` (incluye cliente y servidor) vs. herramientas mencionadas erróneamente (`dropbear`/`dbclient`, `tinyssh`, PuTTY) que no aplican al caso de uso (Linux-a-Linux).

**Verificación exitosa:** `systemctl status sshd.service` → `active (running)`, escuchando en puerto 22 (antes del cambio de puerto).

---

### 5. Generación de llave SSH y autenticación por clave

**Proceso ejecutado (sin errores):**
- `ssh-keygen -t ed25519 -C "bypass"` — llave generada inicialmente **sin passphrase**.
- `ssh-copy-id iusearch@127.0.0.1` — clave copiada exitosamente a `authorized_keys`.
- Verificado login por clave pública en los logs: `Accepted publickey for iusearch`.

**Corrección aplicada después:** Se identificó el riesgo de no tener passphrase en la llave privada (si roban el archivo, acceso directo sin fricción). Se corrigió con:
```
ssh-keygen -p -f ~/.ssh/id_ed25519
```

**Confusión conceptual aclarada:** La passphrase protege la **llave privada**, no la pública (la pública está diseñada para compartirse sin riesgo).

---

### 6. Hardening de `sshd_config` — Error de sintaxis

**Cambios buscados:** cambiar puerto a 2222, `PermitRootLogin no`, `PasswordAuthentication no`.

**Error 1 — Servicio no reiniciaba:**
```
Job for sshd.service failed because the control process exited with error code.
```
**Diagnóstico:** `sudo sshd -t` reveló:
```
/etc/ssh/sshd_config line 34: keyword PermitRootLogin extra arguments at end of line
```
**Causa:** Argumentos sobrantes al final de la línea 34 (probablemente error de edición/dictado).
**Resolución:** Se corrigió la línea manualmente con `nano`, verificado con `sudo sshd -t` (sin output = sintaxis válida).

**Error 2 — `PasswordAuthentication` seguía en `yes` pese a "haberlo cambiado":**
**Diagnóstico:** `sudo sshd -T | grep -i passwordauthentication` mostró `yes` a pesar de edición previa.
**Causa raíz:** La línea 59 (`#PasswordAuthentication yes`) seguía **comentada** — nunca se quitó el `#`, por lo que `sshd` usaba el valor default.
**Resolución:** Se descomentó y cambió a `PasswordAuthentication no`.

**Error 3 — `start-limit-hit` al reiniciar repetidamente:**
```
sshd.service: Start request repeated too quickly.
sshd.service: Failed with result 'start-limit-hit'.
```
**Causa:** Múltiples intentos de restart en poco tiempo activaron la protección de systemd contra reinicios en loop.
**Resolución:**
```
sudo systemctl reset-failed sshd.service
sudo systemctl restart sshd.service
```

**Verificación final exitosa:**
```
Port 2222
PermitRootLogin no
PasswordAuthentication no
GatewayPorts no
```
Login confirmado solo por llave (con prompt de passphrase local, no de password del sistema — distinción aclarada).

---

### Estado al cierre de la sesión

**Completado:**
- Arch reinstalado vía `archinstall` + KDE Plasma (decisión tomada, pendiente ejecución confirmada en sesión)
- Repos de GitHub reorganizados (`dotfiles`, `Scripts`)
- SSH instalado, endurecido: puerto 2222, sin root, sin password auth, llave con passphrase

**Pendiente para continuar (Módulo 05, parte 2):**
- Configurar firewall con `nftables`:
  - Ubicar archivo de configuración principal y entender tablas/cadenas/reglas
  - Política default `drop` en entrada
  - Permitir SSH en puerto 2222 explícitamente
  - Permitir loopback y conexiones establecidas
  - Dejar salida abierta

**Otros pendientes generales:**
- Regenerar Personal Access Token de GitHub con permisos mínimos (esta semana)
- Decidir uso del servidor Oracle Cloud — reservado conceptualmente para práctica de SSH/hardening contra tráfico real de internet (Módulo 05) o para honeypot/Blue Team (Fase 2). Ojo: umbral de inactividad de Oracle es de 7 días, no un mes — mantener actividad mínima si se deja sin usar por ahora.
- Crear repo `arch-lab` (o nombre similar) para configs específicas del sistema

---

### Nota sobre el día anterior (sesión no documentada)

Charly indicó que la sesión de ayer no se cerró con diario. No fue posible reconstruir esa sesión aquí porque su contenido no está disponible en esta conversación — cada sesión de chat es independiente y este diario solo puede documentar lo que ocurrió en la conversación actual. Si Charly quiere dejar constancia de lo de ayer, puede pegar aquí un resumen de lo trabajado (comandos, errores, decisiones) y se agrega como sección aparte.

