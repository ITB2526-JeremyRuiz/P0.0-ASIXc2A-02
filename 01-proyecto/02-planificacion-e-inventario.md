# Planificación del Proyecto, Control de Tareas e Inventario Real

## 1. Metodología de Trabajo y Planificación (ProofHub y Git)

La ejecución del proyecto (**`P0.0-ASIXc2gC-Gnn`** en ProofHub / **`P0.0-ASIXc2A-02`** en GitHub) se ha estructurado siguiendo una metodología ágil iterativa dividida en **3 Sprints quincenales** de 10 horas cada uno (**30 horas totales**), combinando **ProofHub** para la gestión visual de tareas (tablero Kanban) y **GitHub** (con autenticación segura mediante pares de claves SSH pública/privada y comandos `git add`, `commit`, `push`, `pull`, `clone`) para el control de versiones del código y la documentación.

### 1.1. Cronograma y Control de Sprints

| Sprint | Duración | Objetivos y Tareas Ejecutadas | Responsables | Entregable / Evidencia | Estado |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Sprint 1** | 10 horas | • Despliegue de Proxmox VE (`pvexx`) en IsardVDI con ajuste `MTU 1446` (VXLAN).<br>• Diseño de red de 3 zonas (WAN, DMZ, Intranet).<br>• Diagnóstico de bloqueo *Anti-MAC Spoofing* en `R-NCC` (`CT 103`) y migración de enrutamiento, NAT (`iptables`) y `dnsmasq` a Proxmox. | Ivan y Jeremy | [02-Redes-y-seguridad/01-arquitectura-red.md](../02-Redes-y-seguridad/01-arquitectura-red.md) | **Completado** |
| **Sprint 2** | 10 horas | • Despliegue de `B-NCC` (`10.10.20.10`) con Docker Compose (`MySQL 8.0`).<br>• Desarrollo de script Python (`generar_sql.py`) para limpiar UTF-16 LE BOM e importar 2.710 registros.<br>• Despliegue de servidor FTP pasivo `F-NCC` (`delfer/proftpd`) y usuario `bchecker`. | Ivan y Jeremy | [03-servicios/02-configuracion-servidor-ftp.md](../03-servicios/02-configuracion-servidor-ftp.md)<br>[03-servicios/configuracion-servidor-BBDD.md](../03-servicios/configuracion-servidor-BBDD.md) | **Completado** |
| **Sprint 3** | 10 horas | • Despliegue de `LDAP-NCC` (`10.10.10.30`) con OpenLDAP, `ppolicy` y LAM.<br>• Desarrollo de portal CRUD en `WEB-NCC` (`10.10.10.20`) con login LDAP y desbloqueo admin.<br>• Redacción de manual de cliente, plan PRL, contingencia y juego de pruebas. | Ivan y Jeremy | [03-servicios/03-configuracion-ldap.md](../03-servicios/03-configuracion-ldap.md)<br>[04-manuales/manual-cliente.md](../04-manuales/manual-cliente.md)<br>[04-manuales/05-Normativa-PRL/PRL.md](../04-manuales/05-Normativa-PRL/PRL.md) | **Completado** |

---

## 2. Estándar de Administración y Auditoría (`bchecker`)

Para facilitar las tareas de evaluación, auditoría y administración transversal en todos los sistemas operativos, servicios y contenedores del proyecto, se ha estandarizado la cuenta administrativa obligatoria:
* **Usuario:** `bchecker`
* **Contraseña de auditoría:** `bchecker121`
* **Ámbito de aplicación:**
  * **Sistemas Linux (Hosts/LXC):** Cuenta local con shell `/bin/bash` y pertenencia al grupo `sudo`.
  * **Servidor FTP (`F-NCC`):** Usuario autenticado en ProFTPD con directorio raíz `/home/bchecker`.
  * **Base de Datos (`B-NCC`):** Usuario MySQL con permisos completos (`ALL PRIVILEGES ON bcn_educacio.*`).
  * **Directorio LDAP (`LDAP-NCC`):** Entrada `uid=bchecker,ou=usuarios,dc=educacion,dc=local` para pruebas de autenticación y políticas de bloqueo `ppolicy`.

```bash
# Creación del usuario estándar de administración en nodos Linux
useradd -m -s /bin/bash bchecker
echo "bchecker:bchecker121" | chpasswd
usermod -aG sudo bchecker
```

---

## 3. Inventario Real Verificado de Infraestructura

| ID | Hostname | Tipo de Recurso | Sistema Operativo | IP / Subred | Puente Proxmox | Servicios Desplegados | Estado |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Host** | `pvexx` | Nodo Hipervisor (VM IsardVDI) | Debian GNU/Linux (Proxmox VE) | `10.2.169.223` (VPN)<br>`10.10.10.1/24` (DMZ)<br>`10.10.20.1/24` (Intranet)<br>`192.168.2.1/24` (Clientes) | `vmbr0`, `vmbr1`, `vmbr1:0`, `vmbr2` | Virtualización KVM/LXC, Router central, NAT/DNAT (`iptables`), DNS y DHCP (`dnsmasq`). | **Activo** |
| **CT 102** | `CLI-LINUX` | Contenedor LXC | Ubuntu Server | `192.168.2.50` – `.150` (DHCP) | `vmbr2` | Cliente Linux para pruebas DHCP (Opción 121), DNS, HTTP, FTP y ping inter-red. | **Activo** |
| **CT 103** | `R-NCC` | Contenedor LXC | Ubuntu Server | `10.10.10.1` / `10.10.20.1` *(histórico)* | `vmbr1`, `vmbr2` | Router inicial retirado tras detectar filtro *Anti-MAC Spoofing* en IsardVDI. | **Detenido (Retirado)** |
| **CT 104** | `Servidor-FTP` (`F-NCC`) | Contenedor LXC + Docker | Ubuntu + Docker Engine | `10.10.10.10/24` (DMZ) | `vmbr1` | Contenedor `ftp_bcn` (`delfer/proftpd`) en puerto `21` y rango pasivo `60000-60010`. | **Activo** |
| **CT 105** | `WEB-NCC` (`W-NCC`) | Contenedor LXC | Ubuntu Server | `10.10.10.20/24` (DMZ) | `vmbr1` | `Nginx` + `PHP 8.2-FPM` (`php-ldap`, `php-mysql`), portal CRUD `educacion.local`. | **Activo** |
| **CT 106** | `B-NCC` | Contenedor LXC + Docker | Ubuntu + Docker Compose | `10.10.20.10/24` (Intranet) | `vmbr1:0` | Stack Compose: `mysql_bcn_db` (`mysql:8.0`), `php_bcn_app`, `nginx_bcn_web`. BD `bcn_educacio`. | **Activo** |
| **CT** | `LDAP-NCC` | Contenedor LXC | Debian / Ubuntu | `10.10.10.30/24` (DMZ) | `vmbr1` | Demonio `slapd` (OpenLDAP `dc=educacion,dc=local`), *overlay* `ppolicy`, Apache2 + LAM (`/lam`). | **Activo** |
| **VM** | `CLI-WIN` | Máquina Virtual / Host | Windows 10/11 | `192.168.2.50` – `.150` (LAN) / `10.2.x.x` (VPN) | `vmbr2` / `enp2s0` | Cliente Windows para pruebas de navegador, `ipconfig`, rutas DHCP 121 y acceso remoto VPN. | **Activo** |
