# Guia d'administració tècnica

> Document de treball. Els procediments s'han de completar amb les ordres, rutes i valors de l'entorn real abans d'utilitzar-los en producció o lliurar-los com a procediments verificats.

## 1. Abast

Aquesta guia recull les operacions que l'equip administrador ha de documentar per mantenir l'entorn del projecte: plataforma de virtualització, màquines, xarxa, contenidors, servidor web i base de dades.

## 2. Inventari i accés

Abans d'intervenir:
1. Identificar la màquina o servei afectat a l'inventari.
2. Confirmar que es disposa d'autorització.
3. Registrar la data, l'hora i el motiu de la intervenció.
4. Consultar els canvis recents i les incidències obertes.

Les credencials s'han de gestionar fora del repositori. No s'han d'incloure contrasenyes, tokens, claus privades ni fitxers .env reals en aquesta guia.

## 3. Operació de màquines i contenidors

Per cada màquina o contenidor, completar i validar:
- Identificador i nom.
- Tipus de recurs (VM o contenidor).
- Sistema operatiu i versió.
- CPU, memòria i emmagatzematge assignats.
- Interfícies, xarxa i adreçament.
- Servei que proporciona.
- Responsable i dependències.

No s'han d'aplicar canvis de recursos o reinicis sense seguir les normes de l'entorn de pràctiques.

## 4. Serveis

Per a cada servei desplegat, registrar:
- Paquets o imatge utilitzada i versió.
- Fitxer de configuració i ruta.
- Dependències.
- Ordre o procediment d'inici i aturada autoritzat.
- Ports i xarxes utilitzats.
- Logs i ubicació.
- Prova per comprovar que funciona.

### Docker

Abans de documentar ordres concretes, identificar el fitxer Compose real i els noms dels serveis. Com a comprovacions habituals, quan estiguin disponibles i autoritzades, es poden registrar l'estat dels contenidors i els logs. No copiar sortides que continguin secrets.

### Web i base de dades

Cal documentar la relació entre l'aplicació, el servidor web i MySQL. Les credencials de connexió han d'estar protegides. La guia ha d'indicar com es comprova la connexió i com es valida una consulta de lectura sense modificar dades.

## 5. Còpies de seguretat i restauració

El grup ha de confirmar quin mecanisme de còpia existeix realment. Per cada còpia cal registrar:
- Dades incloses i excloses.
- Ubicació protegida.
- Periodicitat i retenció.
- Responsable.
- Data de l'última còpia correcta.
- Resultat de l'última restauració de prova.

No declarar que hi ha còpies vàlides fins que s'hagi comprovat una restauració en un entorn autoritzat.

## 6. Diagnòstic d'incidències

1. Registrar símptoma, hora i recurs afectat.
2. Revisar estat del recurs i logs.
3. Comprovar connectivitat i dependències.
4. Comparar amb l'últim canvi conegut.
5. Aplicar una correcció autoritzada.
6. Repetir la prova funcional.
7. Registrar causa, solució i evidència.

Per a incidències de Proxmox, consulteu [el procediment inicial de recuperació](../infrastructure/proxmox-recovery.md).

## 7. Registre de manteniment

| Data | Recurs | Operació | Resultat | Responsable | Evidència |
|---|---|---|---|---|---|
| Per completar | Per completar | Per completar | Per completar | Per assignar | Per afegir |

## 8. Pendents per validar

- Completar les versions i configuracions reals.
- Afegir les ordres exactes utilitzades a l'entorn.
- Confirmar els permisos i responsables d'administració.
- Documentar el mecanisme de còpies i fer una restauració de prova.
- Afegir captures de l'estat correcte i dels tests.
