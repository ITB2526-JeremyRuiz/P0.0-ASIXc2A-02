# Proyecto Intermodular P0.0 — Infraestructura Multicapa y Portal Educación Barcelona (`educacion.local`)

**Equipo:** Ivan y Jeremy · **Ciclo:** ASIX (Módulo 0379) · **Entorno:** Proxmox VE sobre IsardVDI

Este repositorio contiene la documentación técnica, arquitectónica, normativa y de usuario del proyecto **Educación Barcelona**, una infraestructura segmentada en DMZ, Intranet y Red de Clientes/VPN que da soporte a una aplicación web CRUD con autenticación centralizada OpenLDAP sobre el catálogo oficial de equipamientos educativos de Barcelona (2.710 registros).

---

## Estructura de la Documentación

### 1. Planificación y Estudio Previo (RA1 / RA2)
* [Análisis de Necesidades, Estudio de Mercado y Justificación de Proxmox VE](01-proyecto/01-analisis-y-justificacion.md)
* [Planificación de Sprints, Recursos e Inventario Real de Máquinas](01-proyecto/02-planificacion-e-inventario.md)

### 2. Arquitectura de Red, Seguridad y Contingencia (RA2 / RA3)
* [Arquitectura de Red Centralizada, Reglas Persistentes, DNS/DHCP y Evolución desde `R-NCC`](02-redes-y-seguridad/01-arquitectura-red-y-rncc.md)
* [Procedimientos de Administración, Backups y Juego de Pruebas Integral](02-redes-y-seguridad/02-contingencia-y-pruebas.md)

### 3. Despliegue de Servicios (RA2 / RA3)
* [Servicio Web (`WEB-NCC`) y Servidor de Transferencia de Ficheros (`F-NCC`)](03-servicios/01-servidor-web-y-ftp.md)
* [Servicio de Base de Datos y Limpieza UTF-8 en Docker (`B-NCC` - `10.10.20.10`)](03-servicios/02-base-datos-docker.md) *(archivo `BBDD.md`)*
* [Autenticación Centralizada OpenLDAP, `ppolicy`, LAM y Portal Web (`LDAP-NCC` / `WEB-NCC`)](03-servicios/03-autenticacion-ldap.md) *(archivo `LDAP.md`)*

### 4. Manuales y Prevención de Riesgos Laborales (RA3)
* [Manual de Acceso, Uso y Resolución de Problemas para Clientes (LAN y VPN)](04-manuales/manual-usuario-cliente.md)
* [Plan de Prevención de Riesgos Laborales (PRL) y Matriz de Riesgos Técnicos](05-normativa-y-prl/prevencion-riesgos-laborales.md)

---

## Resumen Rápido de Direccionamiento

| Nodo / Servicio | Rol | Dirección IP | Puerto / Protocolo |
| :--- | :--- | :--- | :--- |
| **`pvexx` (Proxmox)** | Hipervisor, Router, Gateway, NAT, DNS/DHCP | `10.10.10.1` / `10.10.20.1` / `192.168.2.1` | UDP/TCP `53`, UDP `67`, DNAT `80/443` |
| **`F-NCC` (`Servidor-FTP`)** | Transferencia de ficheros (ProFTPD Docker) | `10.10.10.10` (DMZ) | TCP `21`, `60000-60010` |
| **`WEB-NCC`** | Portal Web CRUD (`[http://educacion.local](http://educacion.local)`) | `10.10.10.20` (DMZ) | TCP `80`, `443` |
| **`LDAP-NCC`** | Directorio OpenLDAP + LAM (`/lam`) | `10.10.10.30` (DMZ) | TCP `389` (LDAP), `80` (LAM) |
| **`B-NCC`** | Base de Datos MySQL 8.0 (`bcn_educacio`) | `10.10.20.10` (Intranet) | TCP `3306` |
