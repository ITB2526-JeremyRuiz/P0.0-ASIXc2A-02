# Inventari de màquines

Aquest inventari és inicial. Els identificadors i noms s'han observat a la interfície de Proxmox compartida a la conversa; la funció exacta i la configuració s'han de verificar dins de cada recurs.

## Inventari inicial observat

| ID | Nom mostrat | Funció provisional | Tipus / sistema | Estat de verificació |
|---:|---|---|---|---|
| 100 | prueba | Entorn de proves; funció exacta pendent | Per verificar | Pendent |
| 101 | BasedeDatos | Servidor de base de dades previst | Per verificar | Pendent |
| 102 | clienteubuntu | Client Ubuntu | Per verificar | Pendent |
| 103 | R-NCC | Router previst | Per verificar | Pendent |
| 104 | Servidor-FTP | Servidor FTP previst | Per verificar | Pendent |
| 105 | WEB-NCC | Servidor web previst | Per verificar | Pendent |
| 106 | B-NCC | Funció per confirmar | Per verificar | Pendent |
| 107 | despliegue | Entorn de desplegament; tecnologia exacta pendent | Per verificar | Pendent |

## Fitxa que cal completar per cada recurs

Copieu aquesta plantilla per cada VM o contenidor quan s'hagi comprovat la informació.

- **ID i nom:**
- **Tipus de recurs:** VM / LXC
- **Funció:**
- **Sistema operatiu i versió:**
- **CPU / RAM / disc:**
- **Interfícies de xarxa:**
- **Adreça IP / prefix:**
- **Gateway / DNS:**
- **Xarxa virtual / VLAN:**
- **Serveis instal·lats:**
- **Ubicació de fitxers de configuració:**
- **Com iniciar i aturar:**
- **Comprovar que funciona:**
- **Logs útils:**
- **Còpia de seguretat i restauració:**
- **Incidències conegudes:**
- **Darrera verificació:**

## Regles

- No publicar contrasenyes, claus privades, tokens ni dades sensibles.
- No inventar IP, credencials, versions o recursos assignats.
- Si una màquina canvia de funció, actualitzar aquesta fitxa i el diagrama.
