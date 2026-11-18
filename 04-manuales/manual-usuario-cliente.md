# Manual d'usuari client

## Requisits
- Client Windows o Linux (Ubuntu).
- Navegador web actualitzat.
- Connexió a la xarxa local del laboratori o VPN d'IsardVDI.

## Accés des de la xarxa local
El segment de clients és 192.168.2.0/24. Proxmox actua com a gateway i servidor DNS a 192.168.2.1. El servei DHCP documentat assigna adreces del rang 192.168.2.50–192.168.2.150.

### Windows
1. Obrir les propietats de l'adaptador de xarxa.
2. Entrar a les propietats d'IPv4.
3. Seleccionar l'obtenció automàtica de l'adreça IP i del DNS.
4. En CMD, renovar la configuració si cal:

```cmd
ipconfig /release
ipconfig /renew
ipconfig /flushdns
```

### Ubuntu
El client utilitza DHCP a la interfície de xarxa. La configuració de Netplan documentada activa DHCP i afegeix la ruta cap a 10.10.10.0/24 via 192.168.2.1.

## Accés al portal
El domini intern documentat és `educacion.local`, resolt per DNS a 10.10.10.20. Obrir aquest domini al navegador quan el client estigui connectat a la xarxa del projecte o a la VPN.

## Accés remot
1. Connectar la VPN d'IsardVDI.
2. Comprovar que el client ha rebut connectivitat.
3. Accedir a `educacion.local`.
4. Si no resol el nom, comprovar el DNS rebut i la ruta cap a la xarxa de servidors.

## Problemes habituals
- **No s'obté IP:** comprovar que l'adaptador està connectat a la xarxa de clients i renovar DHCP.
- **No es resol el domini:** comprovar que el DNS és 192.168.2.1.
- **No s'obre el portal:** comprovar la connexió amb 10.10.10.20 i l'estat del servei web.
- **Accés remot fallit:** comprovar que la VPN està connectada i que existeix ruta cap a 10.10.10.0/24.

L'accés al portal i les funcionalitats disponibles depenen de l'estat dels serveis del projecte.
