# Servidor web WEB-NCC i servidor FTP F-NCC

## WEB-NCC
- Hostname: `WEB-NCC`
- Adreça documentada: `10.10.10.20/24`
- Xarxa: DMZ / servidors (`vmbr1`)
- Domini del portal: `educacion.local`
- Accés documentat: HTTPS al port 443.

El servidor allotja el portal Educación Barcelona. La documentació d'integració descriu PHP-FPM, Nginx i autenticació contra OpenLDAP, a més de les operacions de consulta i gestió dels equipaments emmagatzemats a MySQL.

La configuració de Netplan assigna l'adreça 10.10.10.20/24 a la interfície interna i incorpora rutes cap a 10.10.20.0/24 i 192.168.2.0/24 a través de 10.10.10.1.

### Funcionalitats documentades
- Inici de sessió amb autenticació LDAP.
- Consulta i filtratge de registres.
- Alta, edició i eliminació d'equipaments.
- Detecció de comptes bloquejats per `ppolicy`.
- Desbloqueig de comptes per part dels usuaris administradors.

## F-NCC / Servidor-FTP
- Hostname identificat: `Servidor-FTP` / `F-NCC`
- Adreça documentada: `10.10.10.10/24`
- Xarxa: DMZ / servidors.

La configuració de xarxa assigna l'adreça estàtica a la interfície interna i defineix rutes cap a la intranet i la xarxa de clients/VPN a través de 10.10.10.1.

El projecte identifica ProFTPD com a servei FTP previst per a aquest servidor. La informació recuperada no inclou el fitxer de configuració final ni proves de transferència; per tant, no es documenten directives ni resultats com a verificats.
