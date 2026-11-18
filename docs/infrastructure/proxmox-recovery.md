# Procediment inicial de recuperació de Proxmox

> Aquest document és una guia de diagnòstic inicial, no un procediment complet de restauració. Adapteu-lo a la configuració real del centre i seguiu les instruccions del professorat o administrador.

## Abans de començar

- Confirmar quin node i quin recurs estan afectats.
- Registrar l'ID, el nom, l'hora i el símptoma.
- No reiniciar el node ni restaurar còpies sense autorització.
- No copiar ni publicar credencials o informació sensible.

## 1. Comprovar l'accés a la interfície

1. Obrir la URL de Proxmox facilitada pel centre.
2. Comprovar si carrega la pàgina d'inici de sessió.
3. Si no carrega, registrar el missatge del navegador i comprovar la connectivitat amb els mitjans autoritzats.
4. Si la interfície és accessible però una VM/LXC no ho és, continuar amb la comprovació del recurs.

No inclogueu aquí una adreça real del centre si la documentació es farà pública; utilitzeu un marcador com `https://PROXMOX_HOST:8006`.

## 2. Comprovar el recurs afectat

Des de la interfície, inspeccionar:
- Estat del recurs (en execució, aturat o amb error).
- Tasques recents i missatges d'error.
- Ús de CPU, memòria i emmagatzematge.
- Configuració de xarxa associada.

Anotar el resultat abans de canviar la configuració.

## 3. Comprovacions dins del sistema convidat

Només si es disposa d'accés autoritzat a la consola del sistema convidat, comprovar:
- Que el sistema ha acabat d'arrencar.
- Que la interfície de xarxa està activa.
- Que l'adreçament coincideix amb el pla aprovat.
- Que els serveis necessaris estan actius.
- Que els logs no indiquen errors.

Els comandos exactes s'afegiran per sistema operatiu i servei quan es verifiquin en l'entorn del projecte.

## 4. Recuperació i restauració

- No esborrar ni recrear el recurs com a primera mesura.
- Verificar si existeix una còpia de seguretat vàlida i autoritzada.
- Confirmar el punt de restauració i l'impacte abans de restaurar.
- Després de recuperar, repetir les proves de connectivitat i servei i registrar-ne el resultat.

## 5. Registre de la incidència

| Camp | Valor |
|---|---|
| Data i hora | Per completar |
| ID / nom del recurs | Per completar |
| Símptoma | Per completar |
| Comprovacions fetes | Per completar |
| Causa confirmada | Per completar |
| Acció aplicada | Per completar |
| Resultat de la prova posterior | Per completar |
| Referència de l'evidència | Per completar |

## Informació pendent per fer aquest procediment executable

- Identificar qui pot reiniciar o restaurar recursos.
- Documentar el sistema de còpies de seguretat disponible.
- Afegir els comandos concrets per als sistemes utilitzats.
- Definir les proves d'acceptació després de la recuperació.
