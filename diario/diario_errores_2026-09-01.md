# Diario de Errores — Arch Linux Lab

## Sesión: 01 de septiembre 2026

---

### Contexto de la sesión

Cierre de pendientes de la sesión anterior (token de GitHub, repo `arch-lab`) y continuación del Módulo 05: configuración de firewall con `nftables` y hardening adicional con `fail2ban`.

---

### 1. Regeneración de Personal Access Token de GitHub

**Situación:** El token anterior tenía todos los permisos marcados (de más para el caso de uso).

**Acción:** Se generó un token nuevo (classic) con scope únicamente `repo`, y se revocó el token viejo.

**Error encontrado al usarlo:**
```
remote: Invalid username or token. Password authentication is not supported for Git operations.
fatal: Authentication failed
```
**Causa:** `~/.git-credentials` (con `credential.helper store`) seguía teniendo cacheada la credencial vieja, ya revocada.

**Resolución:**
```bash
rm ~/.git-credentials
git push origin main
```
Al borrar el archivo, git volvió a pedir usuario/token, se ingresó el token nuevo, y quedó guardado correctamente. Push exitoso.

**Aprendizaje:** Regenerar un token no invalida automáticamente las credenciales cacheadas localmente — hay que forzar la reautenticación.

---

### 2. Creación del repo `arch-lab` y primer commit del diario

**Proceso:**
- Clonado `arch-lab` (vacío, creado previamente en GitHub).
- Creada carpeta `diario/` dentro del repo.
- Copiado el diario de la sesión anterior (`diario_errores_2026-08-30.md`) desde `~/Downloads/`.
- Commit y push exitosos tras resolver el problema de autenticación del punto 1.

**Nota:** El diario de sesiones ahora vive versionado en `arch-lab/diario/`, en vez de quedar suelto como archivo descargado. Esta sesión (01 de septiembre) es la primera en seguir ese flujo desde el inicio.

---

### 3. Firewall con `nftables`

**Archivo de configuración:** `/etc/nftables.conf` (Arch ya trae un template base "Simple & Safe" con el paquete `nftables`).

**Revisión del template por default:** Traía una política sólida de base (`policy drop` en `input`, descarte de conexiones inválidas, aceptación de conexiones establecidas/relacionadas, loopback permitido, ICMP permitido, `forward` en drop). Solo requería un ajuste.

**Error encontrado:**
```
tcp dport ssh accept comment "allow sshd"
```
La palabra clave `ssh` en `nftables` resuelve al puerto 22 (vía `/etc/services`), pero el `sshd` de esta máquina ya corre en el puerto 2222 (cambiado en la sesión anterior). La regla original habría dejado el puerto 2222 sin abrir en el firewall.

**Resolución:**
```
tcp dport 2222 accept comment "allow sshd"
```

**Proceso de validación y activación (sin errores tras la corrección):**
```bash
sudo nft -c -f /etc/nftables.conf      # validación de sintaxis
sudo systemctl enable --now nftables.service
sudo nft list ruleset                   # confirmación de reglas cargadas
```

**Verificación de que el firewall no bloqueó el propio acceso:** se probó `ssh iusearch@127.0.0.1 -p 2222` en la misma sesión antes de cerrar la terminal original — conexión exitosa.

**Discusión conceptual:** Se aclaró que "fácil de configurar" no equivale a "firewall robusto" — el template es una base mínima (bloquea puertos no autorizados) pero no filtra por origen, no tiene rate-limiting agresivo, ni distingue tráfico de confianza. La siguiente capa de defensa relevante era `fail2ban`.

---

### 4. Instalación y configuración de `fail2ban`

**Instalación:** Vía AUR con `yay` (no está en repos oficiales de Arch).

**Archivo de configuración:** Se usó correctamente `/etc/fail2ban/jail.local` (no se tocó `jail.conf`, siguiendo la convención de que `.conf` se sobreescribe en actualizaciones).

**Error 1 — Jail `sshd` no se activaba pese a "estar configurada":**

Contenido problemático:
```
[sshd]
enable = true
port    = 2222
```

**Diagnóstico:**
```bash
sudo fail2ban-client status
# Number of jail: 0
```
El dry-run (`fail2ban-client -d`) y la validación de sintaxis (`fail2ban-client -t`) no marcaron ningún error, porque `enable = true` es sintácticamente válida como línea `clave = valor` — el problema es que **`fail2ban` no reconoce la clave `enable`**, la directiva correcta es **`enabled`** (con "d" al final). Al no existir esa clave, se ignoró silenciosamente y prevaleció el `enabled = false` del bloque `[DEFAULT]`.

**Resolución:**
```
[sshd]
enabled = true
port    = 2222
logpath = %(sshd_log)s
backend = %(sshd_backend)s
```

**Verificación final exitosa:**
```bash
sudo systemctl restart fail2ban
sudo fail2ban-client status sshd
```
```
Status for the jail: sshd
|- Filter
|  |- Currently failed: 0
|  |- Total failed:     0
|  `- Journal matches:  _SYSTEMD_UNIT=sshd.service + _COMM=sshd + _COMM=sshd-session
`- Actions
   |- Currently banned: 0
   |- Total banned:     0
   `- Banned IP list:
```

**Aprendizaje clave de la sesión:** Las herramientas de validación de sintaxis (`-t`, `-c`, `-d`) confirman que un archivo está bien *formado*, no que las directivas dentro de él sean *válidas o reconocidas* por el programa. Es la segunda vez en el proyecto que un typo de una sola letra (antes `PasswordAuthentication` comentado, ahora `enable` vs `enabled`) pasa desapercibido por herramientas de validación y solo se detecta verificando el estado real del servicio en ejecución.

**Nota de limpieza pendiente (no urgente):** El `jail.local` actual quedó como copia casi completa de `jail.conf` (incluye jails de Apache, MySQL, DNS, etc. no utilizadas). No representa un problema funcional, pero sería más prolijo dejar solo `[DEFAULT]` con overrides propios y `[sshd]`.

---

### Estado al cierre de la sesión — Módulo 05 completo

- SSH: puerto 2222, sin root, sin password auth, solo llave ed25519 con passphrase.
- Firewall (`nftables`): política `drop` por default, solo tráfico esencial permitido (loopback, conexiones establecidas, ICMP, SSH en 2222).
- `fail2ban`: jail `sshd` activa y monitoreando el journal de `sshd.service` correctamente.
- Repo `arch-lab` creado y en uso para versionar los diarios de sesión.
- Token de GitHub regenerado con scope mínimo (`repo`).

**Pendientes generales que siguen abiertos:**
- Limpiar `jail.local` dejando solo las jails relevantes.
- Decidir uso del servidor Oracle Cloud (pendiente de sesiones anteriores).
- Crear config de firewall/fail2ban específica cuando se active el servidor Oracle (tráfico real de internet, no solo localhost).
- Módulo 06 (Docker/Podman) queda como siguiente paso del roadmap.

