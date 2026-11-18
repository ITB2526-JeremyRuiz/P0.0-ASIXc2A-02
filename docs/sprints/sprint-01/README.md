# Sprint 1 — desplegament inicial i entorn de proves

## Objectiu

Posar en marxa l'entorn de treball del projecte, desplegar Proxmox i els clients de prova, i validar una primera aproximació a l'aplicació web connectada a MySQL mitjançant Docker.

> Aquest registre s'ha elaborat a partir del diari del grup. Quan el document original no concreta una dada tècnica o un resultat, s'indica com a pendent de verificar.

## Registre cronològic

| Data | Treball registrat | Resultat indicat al diari | Estat |
|---|---|---|---|
| 07/10/2026 | Pluja d'idees i definició inicial de les màquines i la seva distribució. Desplegament de Proxmox i dels clients Windows i Ubuntu. | Es va aconseguir desplegar Proxmox i els dos clients per accedir a la interfície de gestió de màquines virtuals. | Fet segons el diari |
| 13/10/2026 | Posada en comú del treball anterior i revisió del desplegament. Es decideix canviar de plataforma de desenvolupament cap a Isard NuVulèt perquè l'entorn anterior donava problemes i es buscava més fiabilitat i recursos. | Es va recuperar el punt de treball i es va plantejar preparar un entorn de proves abans del desenvolupament complet. | Decisió registrada; motiu tècnic pendent de detallar |
| 14/10/2026 | Es preveia desplegar l'entorn de proves en un contenidor. | Una xerrada educativa va impedir avançar en les tasques previstes. | Sense avenç registrat |
| 20/10/2026 | Posada en comú de l'estat del projecte i desplegament d'un contenidor Docker de proves amb una estructura semblant a la prevista. | Es va avançar en l'entorn de proves. Va quedar pendent desenvolupar la web per consultar i modificar la base de dades. | Entorn de proves fet segons el diari; detalls de configuració pendents |
| 21/10/2026 | Desenvolupament de la web, instal·lació del servidor web i preparació del HTML. | El diari indica que la web va quedar acabada i funcional i que es va avançar la documentació. | Resultat declarat al diari; proves concretes pendents d'annexar |
| 28/10/2026 | Revisió del primer sprint. | El grup identifica que s'ha dedicat més temps a Proxmox i als serveis que a documentar-los. Es decideix prioritzar la documentació a GitHub i preparar el segon sprint. | Retrospectiva registrada |
| 28/10/2026 (final) | Organització i esquema de la documentació. | Es deixa preparada l'estructura per començar a documentar. | Fet segons el diari |
| 03/11/2026 | Inici de la documentació a GitHub i creació de carpetes per documentar els passos realitzats amb imatges. | El grup indica que la documentació i les imatges ja s'han començat a ordenar. | En curs |
| 04/11/2026 | El diari només conté l'inici de la jornada. | No hi ha resultat final descrit. | Informació incompleta |
| 10/11/2026 | El diari conté l'inici de la jornada. | No hi ha tasques ni resultat descrits. | Informació incompleta |
| 17/11/2026 | El diari conté l'inici de la jornada. | No hi ha tasques ni resultat descrits. | Informació incompleta |

## Resultats del sprint

### Resultats que consten al diari
- Desplegament inicial de Proxmox.
- Preparació de clients Windows i Ubuntu per accedir a l'entorn.
- Decisió de treballar amb Isard NuVulèt després dels problemes detectats a l'entorn anterior.
- Preparació d'un entorn de proves amb Docker.
- Desenvolupament d'una primera web que, segons el diari, va quedar funcional.
- Inici de la documentació del projecte a GitHub.

### Informació tècnica que falta completar
- Inventari verificat de màquines, tipus de recurs, sistema operatiu i funció.
- Configuració de xarxa de cada client i servidor, amb IP, màscara/prefix, gateway i DNS.
- Fitxers Docker utilitzats i serveis desplegats.
- Tecnologia i codi de la web, connexió a MySQL i operacions que es van provar.
- Proves d'acceptació amb resultat esperat, resultat obtingut i captura.
- Incidències concretes de la migració de plataforma i com es van resoldre.

## Incidències i decisions

| Incidència o decisió | Impacte | Acció registrada | Seguiment necessari |
|---|---|---|---|
| Problemes amb la plataforma de desenvolupament inicial | Es considera necessari canviar l'entorn de treball | Migració a Isard NuVulèt | Afegir errors observats i comprovacions que van motivar el canvi |
| Jornada del 14/10 interrompuda per una xerrada | No es va avançar en el desplegament previst | Continuar en la següent sessió | Cap acció tècnica pendent derivada |
| Documentació insuficient després del primer sprint | Dificulta el seguiment i la reproducció dels passos | Crear estructura documental a GitHub | Completar procediments i evidències |

## Planificació i control

El diari disponible no identifica responsables individuals, estimacions d'hores, dependències ni criteris d'acabament per a cada tasca. Aquests camps s'han d'afegir a partir del tauler de seguiment o dels acords reals del grup; no s'han d'assignar retrospectivament sense confirmació.

## Revisió i retrospectiva

**Què ha funcionat:** s'ha arribat a un primer entorn de proves, s'han desplegat els clients i s'ha desenvolupat una primera web funcional segons el diari.

**Què cal millorar:** el grup identifica explícitament que la documentació no ha avançat al mateix ritme que la infraestructura i els serveis.

**Acció de millora:** documentar cada tasca mentre es realitza, adjuntar captures contextualitzades i registrar les proves i incidències abans de donar una tasca per tancada.

## Evidències

Les captures extretes del document original s'han d'incorporar a `evidence/sprint-01/` i referenciar aquí amb número i peu de figura. No es consideren proves suficients les captures sense explicació del pas, l'objectiu i el resultat.

Consulteu [l'índex d'evidències](../../../evidence/README.md).
