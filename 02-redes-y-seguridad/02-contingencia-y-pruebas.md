# Contingència, recuperació i proves

## Recuperació de Proxmox i dels recursos
Abans d'intervenir:
1. Identificar el node o recurs afectat i registrar ID, nom, hora i símptoma.
2. Comprovar l'estat del node i les tasques recents a la interfície Proxmox.
3. Revisar CPU, memòria, emmagatzematge i configuració de xarxa.
4. No reiniciar ni restaurar un recurs sense autorització i sense identificar una còpia vàlida.

Després de recuperar un recurs, comprovar l'arrencada, la interfície de xarxa, l'adreçament i els serveis necessaris.

## Còpies i restauració
La documentació del projecte identifica snapshots i còpies de Proxmox (`vzdump`) com a mecanismes de protecció. No consta en la informació disponible una política completa amb calendari, retenció, destinació ni últim resultat de restauració. Aquests paràmetres no es donen per configurats.

Per a MySQL, el desplegament utilitza el volum Docker `db_data`; també es documenta el fitxer d'importació SQL `inserts.sql`. Abans d'una restauració cal validar la còpia i l'impacte sobre les dades actuals.

## Proves de xarxa
| Prova | Comprovació |
|---|---|
| Client → gateway | El client rep configuració DHCP i arriba a 192.168.2.1 |
| Client → web | Resolució de `educacion.local` i accés a 10.10.10.20 |
| Client → xarxa de servidors | Ruta 10.10.10.0/24 via 192.168.2.1 |
| VPN → web | Accés al portal a través del túnel |
| Web → LDAP | Connexió al servei LDAP de 10.10.10.30:389 |
| Web → MySQL | Connexió a B-NCC 10.10.20.10:3306 |

## Proves de serveis
- MySQL: comprovar l'estat del contenidor i consultar el recompte de registres de `equipaments`.
- LDAP: consultar el directori i comprovar el bloqueig després d'intents fallits.
- Web: provar inici de sessió, consulta, filtratge i operacions CRUD.
- FTP: comprovar autenticació i transferència amb un compte de prova autoritzat.

Els documents disponibles descriuen configuracions i funcionalitats, però no aporten resultats mesurats per a totes les proves d'aquesta llista. Cal registrar el resultat real i l'evidència de cada execució abans de marcar-les com a superades.

## Incidències tècniques identificades
- El servei DHCP en contenidors podia quedar afectat pel filtratge anti-MAC-spoofing d'IsardVDI; es va centralitzar `dnsmasq` a Proxmox.
- La MTU 1446 del túnel IsardVDI requereix tenir en compte l'ajust del MSS TCP.
- El CSV original està codificat en UTF-16; cal normalitzar-lo abans d'importar-lo a MySQL amb UTF-8.

