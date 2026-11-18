# Anàlisi i justificació del projecte

## Necessitat
Centralitzar les dades dels equipaments educatius de Barcelona en una aplicació web que permeti consultar, filtrar i gestionar registres des de clients Windows i Linux, amb accés local i remot.

El fitxer de dades analitzat conté 2.710 registres i 40 columnes. Inclou identificador, nom del centre, dades d'adreça, categoria i coordenades geogràfiques. El fitxer original està codificat en UTF-16.

## Objectius tècnics
- Separar els serveis en segments de xarxa.
- Allotjar les dades en MySQL.
- Proporcionar una aplicació web amb operacions CRUD.
- Centralitzar l'autenticació amb LDAP.
- Facilitar l'accés des de clients locals i mitjançant VPN.

## Alternatives i decisions

### Virtualització
S'han considerat Proxmox VE, VMware ESXi i VirtualBox/KVM.

| Solució | Consideracions del projecte |
|---|---|
| Proxmox VE | Plataforma escollida. Integra KVM i LXC, bridges de xarxa, snapshots i còpies amb eines pròpies. |
| VMware ESXi | Alternativa propietària; la gestió avançada depèn de components addicionals. |
| VirtualBox / KVM | Útil per a entorns de proves, però menys integrat per gestionar la infraestructura del projecte. |

**Elecció:** Proxmox VE. Permet gestionar màquines virtuals i contenidors LXC des d'una interfície central i configurar la xarxa virtual del laboratori.

### Desplegament de serveis
S'utilitza Docker Compose per orquestrar els serveis de base de dades i l'entorn PHP/Nginx documentat a B-NCC. Aquesta opció permet definir serveis i volums en fitxers declaratius.

### Base de dades
S'ha utilitzat MySQL 8.0. L'esquema treballa amb `utf8mb4` per conservar correctament els caràcters del conjunt de dades després del tractament del CSV original.

### Autenticació
S'ha utilitzat OpenLDAP amb LAM per administrar el directori i l'overlay `ppolicy` per aplicar bloqueig després d'intents d'autenticació fallits.

## Abast
El sistema documentat inclou xarxes segmentades, encaminament al node Proxmox, serveis web i FTP, MySQL, LDAP i clients de prova. Les configuracions concretes es detallen als documents de xarxa i serveis.
