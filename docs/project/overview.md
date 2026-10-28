# Visió general del projecte

## Propòsit

El projecte P0.0 consisteix a dissenyar, desplegar i documentar una infraestructura per donar suport a una aplicació multicapa i als serveis de xarxa necessaris.

## Abast previst

Segons el README inicial del repositori, l'abast previst inclou:
- Segmentació de xarxa amb DMZ, intranet i NAT.
- Serveis web, SSH, base de dades, DHCP, DNS i FTP, segons el disseny definitiu.
- Importació a MySQL de dades obertes d'equipaments educatius de Barcelona.
- Aplicació per consultar les dades.
- Clients Windows i Linux per fer comprovacions.
- Documentació de desplegament, proves, incidències i recuperació.

Aquest llistat descriu l'abast previst, no certifica que tots els components estiguin implementats.

## Principis de documentació

1. Documentar només allò que s'ha observat, executat o verificat.
2. Distingir entre **previst**, **en curs**, **implementat** i **verificat**.
3. Per a cada procediment, indicar prerequisits, passos, comprovació i recuperació.
4. Registrar els canvis importants i les incidències.
5. No incloure secrets, contrasenyes ni claus privades.
6. Afegir captures només quan demostrin un estat o resultat important.

## Decisions tècniques

Les decisions i alternatives es registraran aquí o en un document específic quan hi hagi informació confirmada. No s'han d'inferir adreces IP, recursos assignats, versions de sistema ni topologies que encara no s'hagin comprovat.

## Informació pendent

- Confirmar requisits definitius i criteris d'acceptació.
- Confirmar l'arquitectura i el diagrama de xarxa.
- Verificar funció i configuració de cada VM o contenidor.
- Documentar el flux real de desplegament de l'aplicació.
- Registrar proves amb resultats esperats i obtinguts.
