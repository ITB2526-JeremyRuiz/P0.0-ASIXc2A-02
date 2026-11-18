# Planificación del Proyecto, Control de Tareas e Inventario Real

## 1. Metodología y Gestión de Sprints (ProofHub & GitHub)

El desarrollo del proyecto (**`P0.0-ASIXc2gC-Gnn`**) se ha planificado y ejecutado mediante metodología ágil dividida en **3 Sprints** (10 horas por Sprint / 30 horas totales), gestionando el tablero Kanban en **ProofHub** y el control de versiones documental en **GitHub** (autenticación mediante claves SSH).

### Distribución de Fases y Responsabilidades

| Sprint | Fase / Entregables Principales | Responsables | Estado y Evidencia |
| :--- | :--- | :--- | :--- |
| **Sprint 1** | Despliegue de Proxmox VE en IsardVDI, diseño de topología DMZ/Intranet, pruebas iniciales con `R-NCC` y migración de enrutamiento/DNS/DHCP (`dnsmasq`) a `pvexx`. | Ivan y Jeremy | **Completado** (`02-redes-y-seguridad/01-arquitectura-red-y-rncc.md`) |
| **Sprint 2** | Despliegue de contenedor `B-NCC` con Docker Compose (MySQL 8.0), limpieza de codificación UTF-16 BOM e importación de 2.710 registros, y despliegue de `F-NCC` (ProFTPD). | Ivan y Jeremy | **Completado** (`03-servicios/02-base-datos-docker.md`) |
| **Sprint 3** | Despliegue de `LDAP-NCC` (OpenLDAP + `ppolicy` + LAM), desarrollo de aplicación web CRUD en `WEB-NCC` con autenticación LDAP, acceso VPN y documentación integral. | Ivan y Jeremy | **Completado** (`03-servicios/03-autenticacion-ldap.md`) |

---

## 2. Inventario Real de Infraestructura (Verificado)

| ID | Hostname | Tipo | IP / Subred | Puente Proxmox | Servicios Alojados | Estado |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Host** | `pvexx` | Nodo Proxmox VE | `10.2.169.223` (VPN)<br>`10.10.10.1` (DMZ)<br>`10.10.20.1` (Intranet)<br>`192.168.2.1` (Clientes) | `vmbr0`, `vmbr1`, `vmbr1:0`, `vmbr2` | Hipervisor, Router central, NAT (`iptables`), DNS y DHCP (`dnsmasq`). | **Activo** |
| **CT 102** | `CLI-LINUX` | LXC (Ubuntu) | `192.168.2.50-150` (DHCP) | `vmbr2` | Estación cliente Linux para validación DHCP, DNS, FTP y Web. | **Activo** |
| **CT 103** | `R-NCC` | LXC (Ubuntu) | `10.10.10.1` / `10.10.20.1` *(histórico)* | `vmbr1`, `vmbr2` | Router inicial retirado tras detectar filtro *Anti-MAC Spoofing* en IsardVDI. | **Detenido (Obsoleto)** |
| **CT 104** | `Servidor-FTP` (`F-NCC`) | LXC + Docker | `10.10.10.10/24` (DMZ) | `vmbr1` | Servidor FTP pasivo (`delfer/proftpd`, puertos `21` y `60000-60010`). | **Activo** |
| **CT 105** | `WEB-NCC` | LXC (Ubuntu) | `10.10.10.20/24` (DMZ) | `vmbr1` | Servidor Web (`Nginx` + `PHP 8.2-FPM`), portal CRUD `educacion.local` conectado a LDAP y MySQL. | **Activo** |
| **CT 106** | `B-NCC` | LXC + Docker | `10.10.20.10/24` (Intranet) | `vmbr1:0` | Stack Docker Compose (`mysql:8.0`, `php:8.2-fpm`, `nginx:alpine`), BD `bcn_educacio` (2.710 filas). | **Activo** |
| **CT** | `LDAP-NCC` | LXC (Debian/Ubuntu) | `10.10.10.30/24` (DMZ) | `vmbr1` | Servidor OpenLDAP (`dc=educacion,dc=local`), política `ppolicy` e interfaz web LAM (`/lam`). | **Activo** |
| **VM** | `CLI-WIN` | VM / Host Windows | `192.168.2.50-150` (LAN) / `10.2.x.x` (VPN) | `vmbr2` / `enp2s0` | Estación cliente Windows para pruebas de acceso local y remoto por VPN. | **Activo** |
