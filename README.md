# P0.0 — Infraestructura para equipamientos educativos de Barcelona

Projecte intermodular d'ASIX · Grup G02 · Iván Ramírez i Jeremy Ruiz

## Objectiu
Dissenyar, desplegar i documentar una infraestructura segmentada per allotjar el portal web de consulta i gestió dels equipaments educatius de Barcelona, amb base de dades, autenticació centralitzada i accés des de clients locals i remots.

## Documentació

| Apartat | Contingut |
|---|---|
| [01. Anàlisi i justificació](01-proyecto/01-analisis-y-justificacion.md) | Necessitat, alternatives i elecció de Proxmox VE |
| [02. Planificació i inventari](01-proyecto/02-planificacion-e-inventario.md) | Fases, seguiment a ProofHub i inventari de recursos |
| [Xarxes i seguretat](02-redes-y-seguridad/01-arquitectura-red-y-rncc.md) | Topologia, adreçament, Proxmox com a gateway, DNS, DHCP i VPN |
| [Contingència i proves](02-redes-y-seguridad/02-contingencia-y-pruebas.md) | Recuperació, còpies i comprovacions |
| [Servidor web i FTP](03-servicios/01-servidor-web-y-ftp.md) | WEB-NCC i F-NCC |
| [Base de dades Docker](03-servicios/02-base-datos-docker.md) | MySQL 8.0, Compose i importació de dades |
| [Autenticació LDAP](03-servicios/03-autenticacion-ldap.md) | OpenLDAP, ppolicy, LAM i integració PHP |
| [Manual d'usuari](04-manuales/manual-usuario-cliente.md) | Connexió LAN/VPN i ús del portal |
| [PRL i riscos tècnics](05-normativa-y-prl/prevencion-riesgos-laborales.md) | Prevenció laboral i mesures davant riscos tècnics |

## Arquitectura resumida
- **Clients/VPN:** 192.168.2.0/24, gateway Proxmox 192.168.2.1.
- **Xarxa de servidors (DMZ):** 10.10.10.0/24.
- **Intranet:** 10.10.20.0/24.
- **Hipervisor/gateway:** Proxmox VE (`pvexx`), amb bridges `vmbr0`, `vmbr1` i `vmbr2`.
- **Serveis documentats:** WEB-NCC, F-NCC, B-NCC i LDAP-NCC.

## Dades
El projecte treballa amb el conjunt de dades d'equipaments d'educació del portal Open Data BCN: [Equipaments d'educació](https://opendata-ajuntament.barcelona.cat/data/es/dataset/equipament-educacio/). La documentació tècnica registra 2.710 registres i 40 columnes al fitxer original.

## Criteri documental
La documentació descriu configuracions i resultats recollits al projecte. Quan una dada no està acreditada, no es presenta com a verificada. No s'hi publiquen contrasenyes, tokens ni claus privades.
