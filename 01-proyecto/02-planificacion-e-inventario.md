# Planificació i inventari

## Fases de treball
1. Preparació de l'entorn de virtualització i clients.
2. Disseny de la topologia i configuració de xarxa.
3. Desplegament dels serveis web, FTP, base de dades i LDAP.
4. Integració de l'aplicació amb MySQL i LDAP.
5. Proves de connectivitat, accés i funcionament.
6. Documentació i recopilació d'evidències.

El seguiment de tasques del grup es realitza amb ProofHub. Les dates i responsables s'han de consultar al tauler del projecte; no s'assignen aquí tasques sense aquesta informació.

## Inventari de recursos

### Recursos identificats a la vista de Proxmox

| ID | Nom observat | Funció identificada o prevista |
|---:|---|---|
| 100 | prueba | Entorn de proves |
| 101 | BasedeDatos | Recurs relacionat amb base de dades |
| 102 | clienteubuntu | Client Ubuntu |
| 103 | R-NCC | Router inicial |
| 104 | Servidor-FTP | Servidor FTP |
| 105 | WEB-NCC | Servidor web |
| 106 | B-NCC | Recurs identificat amb aquest nom |
| 107 | despliegue | Entorn de desplegament |

### Adreçament dels serveis documentats

| Sistema | Adreça | Xarxa |
|---|---|---|
| Proxmox, bridge de servidors | 10.10.10.1/24 | Servidors |
| Proxmox, bridge de clients | 192.168.2.1/24 | Clients |
| WEB-NCC | 10.10.10.20/24 | Servidors |
| F-NCC / Servidor-FTP | 10.10.10.10/24 | Servidors |
| LDAP-NCC | 10.10.10.30 | Servidors |
| B-NCC / base de dades | 10.10.20.10/24 | Intranet |

Els identificadors de Proxmox i els noms funcionals dels serveis provenen de documents diferents del projecte; cal mantenir aquesta distinció quan es relacionin VM/LXC i serveis.

## Recursos de programari
- Proxmox VE sobre l'entorn IsardVDI.
- Clients Windows i Ubuntu.
- Docker i Docker Compose.
- MySQL 8.0.
- Nginx i PHP-FPM.
- OpenLDAP i LDAP Account Manager.
- ProofHub per al seguiment de tasques.
- GitHub per al control de versions i la documentació.
