# Proyecto Intermodular P0.0 — Infraestructura Multicapa y Portal Educación Barcelona (`educacion.local`)

* **Repositorio:** `P0.0-ASIXc2A-02`
* **Autores:** Ivan y Jeremy
* **Ciclo Formativo:** Administración de Sistemas Informáticos en Red Y Ciberseguridad (ASIXc — Módulo 0379)
* **Plataforma de Despliegue:** Proxmox VE (`pvexx`) sobre IsardVDI (encapsulación VXLAN · MTU 1446)

Este repositorio contiene la documentación técnica, arquitectónica, normativa y de usuario del proyecto **Educación Barcelona**. La solución despliega una infraestructura segmentada en **DMZ**, **Intranet** y **Red de Clientes/VPN** que da soporte a una aplicación web CRUD con autenticación centralizada **OpenLDAP (`ppolicy`)** sobre el catálogo oficial de equipamientos educativos del Ayuntamiento de Barcelona (**2.710 registros**).

---

## Estructura del Repositorio e Índice de Documentación

### 1. `01-proyecto/` — Estudio Previo, Planificación e Inventario
* [01-analisis-y-justificacion.md](01-proyecto/01-analisis-y-justificacion.md) — Planteamiento del problema, estudio de mercado y justificación de Proxmox VE, Docker, OpenLDAP y MySQL.
* [02-planificacion-e-inventario.md](01-proyecto/02-planificacion-e-inventario.md) — Planificación por Sprints en ProofHub, control de versiones Git, estándar `bchecker` e inventario real de nodos.

### 2. `02-Redes-y-seguridad/` — Arquitectura de Red, Seguridad y Contingencia 
* [01-arquitectura-red.md](02-Redes-y-seguridad/01-arquitectura-red.md) — Topología segmentada, evolución desde `R-NCC` hacia Proxmox `pvexx`, reglas persistentes `iptables`, DNS/DHCP (`dnsmasq`) y Netplan.
* [Contigencia-y-pruebas.md](02-Redes-y-seguridad/Contigencia-y-pruebas.md) — Protocolos de copias de seguridad (`vzdump` y `mysqldump`), recuperación ante desastres y juego de pruebas integral.

### 3. `03-servicios/` — Despliegue de Servicios e Infraestructura 
* [01-configuracion-servidor-web.md](03-servicios/01-configuracion-servidor-web.md) — Despliegue de `WEB-NCC` (`Nginx` + `PHP 8.2-FPM`), arquitectura CRUD, seguridad SQLi/XSS y script `deploy.sh`.
* [02-configuracion-servidor-ftp.md](03-servicios/02-configuracion-servidor-ftp.md) — Despliegue de `F-NCC` (`ProFTPD` en Docker Compose), modo pasivo y usuario global `bchecker`.
* [03-configuracion-ldap.md](03-servicios/03-configuracion-ldap.md) — Despliegue de `LDAP-NCC` (OpenLDAP `dc=educacion,dc=local`), política `ppolicy`, LAM e integración completa en PHP.
* [configuracion-servidor-BBDD.md](03-servicios/configuracion-servidor-BBDD.md) — Despliegue de `B-NCC` (`MySQL 8.0` en Docker), esquema `init.sql` y limpieza UTF-16 LE BOM a `utf8mb4` (2.710 registros).

### 4. `04-manuales/` — Manuales de Usuario y Normativa PRL 
* [manual-cliente.md](04-manuales/manual-cliente.md) — Manual de conexión LAN (DHCP Opción 121) y VPN, guía funcional del portal web, acceso FTP y troubleshooting.
* [05-Normativa-PRL/PRL.md](04-manuales/05-Normativa-PRL/PRL.md) — Plan de Prevención de Riesgos Laborales (PVD, eléctrico, cableado, CPD) y matriz de riesgos técnicos.

---

## Mapa de Direccionamiento Consolidado

| Nodo / Servicio | Rol Principal | Zona / Puente Proxmox | Dirección IP | Puertos / Protocolos |
| :--- | :--- | :--- | :--- | :--- |
| **`pvexx` (Proxmox VE)** | Hipervisor, Router, Gateway, NAT, Firewall, DNS y DHCP (`dnsmasq`) | WAN (`vmbr0`) / VPN (`enp2s0`)<br>DMZ (`vmbr1`)<br>Intranet (`vmbr1:0`)<br>Clientes (`vmbr2`/`enp3s0`) | `10.2.169.223` (VPN)<br>`10.10.10.1/24`<br>`10.10.20.1/24`<br>`192.168.2.1/24` | UDP/TCP `53` (DNS)<br>UDP `67` (DHCP + Op. 121)<br>DNAT TCP `80`, `443` |
| **`F-NCC` (`Servidor-FTP`)** | Transferencia de ficheros en contenedor Docker (`delfer/proftpd`) | DMZ (`vmbr1`) | `10.10.10.10/24` | TCP `21` (Control)<br>TCP `60000-60010` (Pasivo) |
| **`WEB-NCC` (`W-NCC`)** | Portal Web CRUD `[http://educacion.local](http://educacion.local)` (Nginx + PHP 8.2-FPM) | DMZ (`vmbr1`) | `10.10.10.20/24` | TCP `80` (HTTP), `443` (HTTPS) |
| **`LDAP-NCC`** | Directorio OpenLDAP (`dc=educacion,dc=local`), `ppolicy` y LAM (`/lam`) | DMZ (`vmbr1`) | `10.10.10.30/24` | TCP `389` (LDAP), `80` (LAM) |
| **`B-NCC`** | Base de Datos MySQL 8.0 (`bcn_educacio`, 2.710 registros) en Docker | Intranet (`vmbr1:0`) | `10.10.20.10/24` | TCP `3306` (MySQL aislado) |
| **`CLI-LINUX` / `CLI-WIN`** | Estaciones de trabajo cliente (Ubuntu y Windows) | Red Clientes (`vmbr2`) / VPN | `192.168.2.50` – `.150` (DHCP) | HTTP, HTTPS, FTP, DNS, ICMP |
