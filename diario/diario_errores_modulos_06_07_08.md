# Diario de Errores — Arch Linux Lab

## Sesión: Módulos 06, 07 y 08 (Podman, PostgreSQL/MariaDB, QEMU/KVM)

---

### Contexto de la sesión

Sesión larga y productiva: decisión de contenedores (Podman vs Docker), instalación y validación de Podman, despliegue de PostgreSQL y MariaDB en contenedores rootless, instalación completa del stack QEMU/KVM/libvirt, y primer intento de VM (Kali Linux) con troubleshooting real de red.

---

### 1. Decisión: Podman vs Docker

**Análisis realizado:**
- Podman: rootless por default, sin daemon, mejor integración systemd (quadlets), mantenido por Red Hat.
- Docker: mayor adopción general (71.1% desarrolladores vs 11.1% Podman según encuestas recientes), mejor documentación/comunidad, pero requiere daemon con privilegios root.
- Contexto de mercado: Podman gana terreno en sectores regulados (finanzas, salud, gobierno) por requisitos de compliance (SOC 2, HIPAA) alineados con el principio de menor privilegio.

**Decisión:** Podman, alineado con la filosofía de seguridad/control que ha guiado el proyecto completo (KDE sobre Hyprland, nftables sobre iptables, llaves sobre password).

**Kubernetes:** Se descartó como prioridad actual — no está en el plan de estudios, y es sobre-ingeniería para un lab de un solo nodo. Queda como posible tema si la Fase 2 (tour de especialidades) deriva hacia Cloud/DevSecOps.

---

### 2. Instalación y validación de Podman

**Instalación:** Sin errores, vía repos oficiales de Arch.

**Verificación de modo rootless — confirmada exitosamente:**
- `podman info` → `security.rootless: true`
- `idMappings` correctos (UID 1000 del host mapeado a rango 100000-165536 dentro del contenedor)
- `/etc/subuid` y `/etc/subgid` ya preconfigurados con el rango correspondiente para `iusearch`
- Networking rootless funcionando vía `pasta` + `netavark`
- Prueba con `podman run --rm hello-world`: exitosa, sin necesidad de `sudo` en ningún momento

**Sin errores en esta sección.**

---

### 3. PostgreSQL en contenedor (Módulo 07)

**Error 1 — Bind mount sin carpeta previa:**
```
Error: statfs /home/iusearch/pgdata: no such file or directory
```
**Causa:** A diferencia de Docker, Podman no crea automáticamente el directorio del bind mount si no existe.
**Resolución:** `mkdir -p /home/iusearch/pgdata` antes de correr el contenedor.

**Error 2 — Ruta de datos incompatible con la versión de la imagen:**
```
Error: in 18+, these Docker images are configured to store database data in a
       format which is compatible with "pg_ctlcluster"...
```
**Causa:** La imagen `postgres:latest` (v18+) cambió la convención de montaje: ya no espera el volumen en `/var/lib/postgresql/data`, sino en `/var/lib/postgresql` directo (crea su propia subcarpeta versionada adentro).
**Resolución:** Ajustar el `-v` a `/home/iusearch/pgdata:/var/lib/postgresql` (sin el `/data` final).

**Verificación exitosa:** Contenedor corriendo (`Up`), conexión con `psql`, creación de tabla de prueba, inserción y lectura de datos, y **persistencia confirmada tras `podman restart`** — los datos sobrevivieron el reinicio gracias al bind mount.

---

### 4. MariaDB en contenedor (Módulo 07, continuación)

**Sin errores** — proceso análogo a PostgreSQL, ajustando nombres de variables (`MARIADB_ROOT_PASSWORD`, `MARIADB_USER`, `MARIADB_PASSWORD`, `MARIADB_DATABASE`), ruta de datos (`/var/lib/mysql`) y puerto (3306→3307). Corrió a la primera. Prueba de tabla + persistencia tras `restart` confirmada igual que con PostgreSQL.

**Módulo 07 cerrado por completo:** ambas bases de datos funcionando en paralelo, rootless, con volúmenes persistentes validados.

---

### 5. Instalación de QEMU/KVM/libvirt (Módulo 08)

**Verificación previa de hardware:**
- `grep -E 'vmx|svm' /proc/cpuinfo` → `vmx` presente (Intel VT-x soportado)
- `lsmod | grep kvm` → `kvm_intel` y `kvm` cargados correctamente

**Instalación:** `bridge-utils` no encontrado (paquete deprecado/removido de Arch, reemplazado por `iproute2`, ya incluido en el sistema base). Se instaló el resto del stack sin problema (151 paquetes vía `qemu-full`, `virt-manager`, `libvirt`, `dnsmasq`, `vde2`, `openbsd-netcat`, `dmidecode`).

**Post-instalación:**
- Usuario agregado al grupo `libvirt`: `sudo usermod -aG libvirt iusearch`
- Servicio habilitado: `sudo systemctl enable --now libvirtd.service`
- `sudo virt-host-validate`: todos los checks en `PASS`, salvo un `WARN` esperado por ausencia de SEV/SEV-ES/SEV-SNP/TDX (confidential computing) — irrelevante para el caso de uso de un lab local propio.

---

### 6. Primer intento de VM — Kali Linux ("kali-attacker")

**Error 1 — Ruta de ISO inexistente:**
```
ERROR Validating install media '/home/iusearch/isos/kali.iso' failed: ... No such file or directory
```
**Causa:** La carpeta `~/isos/` nunca se creó; el ISO había quedado descargado en `~/vms/` por configuración del navegador.
**Resolución:** `mkdir -p ~/isos` + `mv` del archivo a la ruta correcta.

**Error 2 — Red "default" no encontrada:**
```
ERROR Network not found: no network with matching name 'default'
```
**Causa raíz:** `libvirt` maneja dos conexiones separadas y aisladas entre sí — `qemu:///system` (con privilegios, red `default` gestionada ahí) y `qemu:///session` (sin privilegios, universo vacío de redes/VMs). El primer intento de `virt-install` usó `qemu:///session` por default, donde no existía ninguna red.
**Resolución:** Forzar `--connect qemu:///system` en todos los comandos de `virsh` y `virt-install`, y activar la red ahí: `sudo virsh net-start default` + `sudo virsh net-autostart default`.

**Advertencia adicional resuelta:** `virt-viewer` no estaba instalado (necesario para ver la consola gráfica de la VM) → `sudo pacman -S virt-viewer`.

**Error 3 — DHCP fallido dentro de la VM (troubleshooting más largo de la sesión):**

La VM arrancó correctamente y llegó al instalador gráfico de Kali, pero falló la autoconfiguración de red por DHCP.

**Diagnóstico paso a paso:**
1. Confirmado que `dnsmasq` corría correctamente y que `virbr0` tenía IP (`192.168.122.1/24`) — la red virtual del lado del host estaba sana.
2. **Causa real encontrada:** el firewall `nftables` propio (configurado en el Módulo 05) tiene `policy drop` tanto en `input` como en `forward`, sin ninguna excepción para el tráfico del bridge `virbr0`. `libvirt` crea su propia tabla `nftables` (`libvirt_network`) con sus reglas de NAT/forward para las VMs, pero **en nftables todas las tablas enganchadas al mismo hook deben ser satisfechas** — la tabla propia (`inet filter`), vacía de reglas para `virbr0`, bloqueaba el tráfico antes de que la tabla de `libvirt` pudiera actuar.
3. Se agregaron reglas en la cadena `forward` de la tabla propia: `iifname "virbr0" accept` y `oifname "virbr0" accept`.
4. Tras reintentar el DHCP, siguió fallando — el paquete DHCP no necesita "forward" (atravesar la máquina), sino llegar directo al proceso `dnsmasq` que escucha en el propio host, por lo que el paquete entra por la cadena **`input`**, no `forward`.
5. Se agregó también en la cadena `input`: `iifname "virbr0" accept comment "allow traffic from libvirt bridge to host"`.

**Resultado:** Pendiente de confirmación final — el fix se aplicó y se dejó reintentando el DHCP en la VM. **Nota importante guardada en memoria del asistente** para futuras VMs en cualquier máquina con esta misma configuración (nftables + libvirt), de modo que estas dos reglas se apliquen preventivamente antes de crear la próxima VM.

**Incidente adicional — instalación muy lenta (~1 hora) y contraseña de usuario olvidada:**
- La instalación completa tardó aproximadamente una hora, notablemente lento.
- Causas probables identificadas: (a) disco virtual `qcow2` corriendo sobre un host con `btrfs`, generando doble capa de copy-on-write; (b) solo 2 vCPUs asignadas para el proceso de instalación; (c) posible modo de caché de disco conservador por default en `virt-install`.
- Al llegar a la TTY final, se perdió/olvidó el usuario y contraseña configurados durante la instalación.
- **Decisión tomada:** en lugar de recuperar la contraseña vía modo de recuperación de GRUB, se optó por borrar la VM completa y reintentar desde cero con ajustes de rendimiento.

**Error al borrar la VM — NVRAM:**
```
error: Requested operation is not valid: Cannot undefine domain with NVRAM/varstore
```
**Causa:** Las VMs creadas con `--boot uefi` generan un archivo NVRAM adicional (variables de firmware UEFI) que `libvirt` no elimina por default junto con la definición de la VM.
**Resolución:** `virsh --connect qemu:///system undefine kali-attacker --nvram`.

**Estado al cierre:** VM `kali-attacker` completamente borrada (definición + NVRAM + disco `qcow2`). Pendiente recrearla en la próxima sesión con ajustes: `cache=writeback` en el disco y 4 vCPUs en lugar de 2, para acelerar la instalación.

---

### Estado al cierre de la sesión

**Completado:**
- Módulo 06 (Podman): instalado, validado en modo rootless.
- Módulo 07 (PostgreSQL + MariaDB): ambos corriendo en contenedores, con persistencia de datos confirmada.
- Módulo 08 (QEMU/KVM/libvirt): stack instalado y validado (`virt-host-validate` en verde).
- Fix de firewall para tráfico de VMs (`virbr0`) documentado y aplicado — reutilizable para futuras VMs.

**Pendiente para la próxima sesión:**
- Recrear la VM `kali-attacker` con ajustes de rendimiento (`cache=writeback`, 4 vCPUs) y confirmar que el DHCP funciona con las reglas de firewall ya aplicadas.
- Completar la mini-red de la Fase 1 (Windows víctima + Kali/Parrot + pfSense) una vez validada la primera VM.
- Cerrar formalmente el Módulo 08 y con ello la **Fase 0 completa**.
- Seguir con el **capstone de VPN + Shadowsocks en el servidor Oracle Cloud** (WireGuard + ofuscación), como cierre final de Fase 0.

**Plan a mediano plazo (decisión tomada en esta sesión):**
Tras cerrar Fase 0 (módulos restantes + capstone VPS), Charly planea tomar una pausa del proyecto práctico para enfocarse en **certificaciones** que refuercen y validen el conocimiento adquirido hasta ahora, antes de continuar con la Fase 1 (fundamentos de seguridad) y Fase 2 (tour de especialidades) del plan de estudios.

**Pendientes generales que siguen abiertos (de sesiones anteriores):**
- Limpiar `jail.local` de fail2ban dejando solo las jails relevantes.
- Decidir uso activo del servidor Oracle Cloud (mantener actividad mínima para evitar reclamo por inactividad — umbral de 7 días).

