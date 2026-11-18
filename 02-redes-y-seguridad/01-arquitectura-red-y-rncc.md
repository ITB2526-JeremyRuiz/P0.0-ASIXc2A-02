# Arquitectura de xarxa i evolució de R-NCC

## Topologia desplegada
La infraestructura utilitza Proxmox VE (`pvexx`) com a node de virtualització i punt central d'encaminament. La topologia documentada separa tres segments:

| Segment | Xarxa | Interfície/bridge | Funció |
|---|---|---|---|
| Clients i VPN | 192.168.2.0/24 | `vmbr2` / `enp3s0` | Accés dels clients |
| DMZ / servidors | 10.10.10.0/24 | `vmbr1` | WEB-NCC, F-NCC i LDAP-NCC |
| Intranet | 10.10.20.0/24 | Xarxa interna | B-NCC i MySQL |

Proxmox té l'adreça 10.10.10.1/24 a `vmbr1` i 192.168.2.1/24 a `vmbr2`. La sortida externa es fa mitjançant `vmbr0`.

## Serveis i adreces
| Equip | Adreça | Funció |
|---|---|---|
| WEB-NCC | 10.10.10.20 | Portal web |
| F-NCC / Servidor-FTP | 10.10.10.10 | Servei FTP |
| LDAP-NCC | 10.10.10.30 | OpenLDAP i LAM |
| B-NCC | 10.10.20.10 | MySQL i serveis de dades |

## Evolució de R-NCC
La primera planificació contemplava un router independent identificat com R-NCC. La configuració posterior descriu el trasllat de les funcions d'encaminament, gateway, DHCP i DNS al mateix hipervisor Proxmox. La documentació tècnica atribueix aquesta decisió a les restriccions de xarxa d'IsardVDI, especialment el filtratge anti-MAC-spoofing que dificultava servir DHCP des d'un contenidor.

Per tant, en la topologia descrita, Proxmox assumeix les funcions que inicialment s'havien assignat a R-NCC.

## Encaminament a Proxmox
La configuració de xarxa del node utilitza:
- `vmbr0` per a la sortida externa.
- `enp2s0` com a interfície VPN, amb MTU 1446.
- `vmbr1` amb 10.10.10.1/24 per a la xarxa de servidors.
- `vmbr2` amb 192.168.2.1/24 i `enp3s0` per als clients.

Les regles d'`iptables` descrites permeten el forwarding entre la VPN i els servidors, apliquen MASQUERADE i ajusten el MSS TCP per evitar problemes relacionats amb la MTU.

## DNS i DHCP
El servei `dnsmasq` s'executa al node Proxmox:
- Domini local: `educacion.local`.
- `educacion.local` i `www.educacion.local` resolen a 10.10.10.20.
- Rang DHCP de clients: 192.168.2.50–192.168.2.150.
- Gateway i DNS lliurats als clients: 192.168.2.1.
- Opció DHCP 121 per anunciar la ruta 10.10.10.0/24 via 192.168.2.1.
- Reenviadors DNS externs documentats: 1.1.1.1 i 8.8.8.8.

## VPN
La VPN entra al node per `enp2s0`. Proxmox reenvia el trànsit cap a `vmbr1`; els servidors de la DMZ tenen rutes de retorn cap a la xarxa de clients/VPN a través de Proxmox. El DNS local permet resoldre el domini del portal sense editar manualment `/etc/hosts`.

## Fitxers de configuració
- Proxmox: `/etc/network/interfaces`
- DNS/DHCP: `/etc/dnsmasq.conf`
- Clients i servidors Linux: `/etc/netplan/50-cloud-init.yaml` o el fitxer Netplan específic de cada equip.

Les adreces i regles s'han de contrastar amb la configuració activa abans d'aplicar canvis en un entorn en producció.
