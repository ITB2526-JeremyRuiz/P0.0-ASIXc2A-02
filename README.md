# P0.0 --- Desplegament d'infraestructura

> **Mòdul:** 0379 --- Projecte intermodular d'administració de sistemes
> informàtics en xarxa\
> **Cicle:** ASIX\
> **Estat:** En desenvolupament\
> **Grup:** `Gnn` · **Equip:** `NCC`

## 1. Descripció del projecte

Aquest repositori recull la configuració, els scripts i la documentació
del projecte **P0.0 --- Desplegament d'infraestructura**.

L'objectiu és preparar una infraestructura per a una aplicació multicapa
que integri serveis de xarxa i sistemes, incloent-hi servidor web, accés
SSH, base de dades, DHCP, DNS i FTP. També es desplegaran dos equips
client, un amb Windows i un altre amb Linux.

La configuració concreta, les decisions tècniques i les proves
realitzades s'aniran documentant progressivament en aquest repositori.

## 2. Objectius

-   [ ] Dissenyar i documentar l'arquitectura de la infraestructura.
-   [ ] Configurar un router amb les xarxes DMZ, intranet i NAT.
-   [ ] Desplegar i configurar els serveis requerits.
-   [ ] Crear una base de dades MySQL i importar-hi les dades
    d'equipaments educatius de Barcelona.
-   [ ] Preparar una petita aplicació que mostri les dades
    emmagatzemades.
-   [ ] Configurar l'accés al repositori Git mitjançant claus SSH.
-   [ ] Documentar el flux de treball Git i el desplegament.
-   [ ] Definir i executar proves de connectivitat i funcionament.
-   [ ] Recollir possibles millores per a una futura versió.

## 3. Arquitectura prevista

La infraestructura es dividirà inicialment en tres xarxes:

  -----------------------------------------------------------------------
  Xarxa                   Funció                  Estat
  ----------------------- ----------------------- -----------------------
  **DMZ**                 Ubicació dels serveis   Pendent de definir
                          que hagin d'estar       
                          separats de la xarxa    
                          interna.                

  **Intranet**            Xarxa interna per als   Pendent de definir
                          serveis i clients.      

  **NAT**                 Connexió amb xarxes     Pendent de definir
                          externes quan sigui     
                          necessària.             
  -----------------------------------------------------------------------

### Equips i serveis

  -----------------------------------------------------------------------
  Equip / servei    Hostname previst  Funció            Estat
  ----------------- ----------------- ----------------- -----------------
  Router            `R-NCC`           Interconnexió de  Pendent
                                      les tres xarxes.  

  Servidor web      `W-NCC`           Allotjar          Pendent
                                      l'aplicació web.  

  Servidor de base  `B-NCC`           MySQL i dades     Pendent
  de dades                            d'equipaments     
                                      educatius.        

  Servidor FTP      `F-NCC`           Transferència de  Pendent
                                      fitxers.          

  Servei SSH        Per definir       Administració     Pendent
                                      remota dels       
                                      equips.           

  Servei DHCP       Per definir       Assignació        Pendent
                                      automàtica de     
                                      configuració IP.  

  Servei DNS        Per definir       Resolució dels    Pendent
                                      noms requerits    
                                      pel projecte.     

  Client Windows    Per definir       Proves des d'un   Pendent
                                      client Windows.   

  Client Linux      Per definir       Proves des d'un   Pendent
                                      client Linux.     
  -----------------------------------------------------------------------

> **Nota:** els hostnames, les adreces IP, les màscares, les
> passarel·les i la ubicació definitiva dels serveis s'afegiran quan es
> concreti el disseny de xarxa.

## 4. Serveis i funcionalitats

### Xarxa

-   Router amb segments DMZ, intranet i NAT.
-   Configuració de DHCP i DNS.
-   Proves de connectivitat entre xarxes i equips.

### Sistemes i serveis

-   Servidor web.
-   Accés remot mitjançant SSH.
-   Servidor FTP.
-   Base de dades MySQL.
-   Aplicació de consulta de les dades importades.

### Dades

Es preveu importar a MySQL el conjunt de dades d'equipaments d'educació
de la ciutat de Barcelona.

-   [Portal de dades obertes de Barcelona --- equipaments
    d'educació](https://opendata-ajuntament.barcelona.cat/data/es/dataset/equipament-educacio/)
-   [Recurs CSV del conjunt de
    dades](https://opendata-ajuntament.barcelona.cat/data/dataset/f36b60f2-9541-4d08-b0f9-b0a9313fab3d/resource/29d9ff10-6892-4f16-9012-d5c4997857e7/download)

## 5. Organització del repositori

L'estructura documental es definirà i s'ampliarà a mesura que avancem.
Com a punt de partida, es proposa aquesta organització:

``` text
P0.0-ASIXc2gC-Gnn/
├── README.md
├── docs/                 # Documentació general i decisions tècniques
├── network/              # Configuració i documentació de xarxa
├── services/             # Configuració dels serveis desplegats
├── database/             # Esquema, importació i consultes de la BBDD
├── application/          # Codi de l'aplicació web
├── scripts/              # Scripts d'ajuda i automatització
└── tests/                # Proves i evidències de funcionament
```

> Aquesta estructura és **provisional**. Les carpetes i els fitxers es
> crearan quan definim el contingut de cada apartat; no s'ha de
> considerar encara una estructura implementada.

## 6. Planificació i seguiment

El projecte es planteja en tres sprints quinzenals. Les tasques,
prioritats i responsables s'han de concretar al backlog del projecte.

  ------------------------------------------------------------------------
  Sprint            Període           Objectiu           Estat
  ----------------- ----------------- ------------------ -----------------
  Sprint 1          Per definir       Disseny i          Pendent
                                      preparació de la   
                                      infraestructura.   

  Sprint 2          Per definir       Desplegament i     Pendent
                                      configuració dels  
                                      serveis.           

  Sprint 3          Per definir       Integració, proves Pendent
                                      i documentació     
                                      final.             
  ------------------------------------------------------------------------

## 7. Estat actual

Aquest README és el document inicial del repositori. Encara falta
incorporar-hi la configuració real, els diagrames, les evidències de
proves i la documentació detallada de cada servei.

-   [ ] Confirmar el nom definitiu del grup i l'equip.
-   [ ] Afegir el diagrama de l'arquitectura.
-   [ ] Definir el pla d'adreçament IP.
-   [ ] Documentar la configuració de cada servei.
-   [ ] Afegir les proves i els resultats obtinguts.
-   [ ] Actualitzar l'estat de les tasques a mesura que es completin.

## 8. Seguretat i control de versions

-   L'accés del servidor al repositori Git es configurarà mitjançant una
    parella de claus SSH.
-   Les operacions `clone`, `add`, `commit`, `push` i `pull` es
    documentaran a la carpeta de documentació quan es prepari aquest
    apartat.
-   **No s'han de pujar contrasenyes, claus privades, tokens ni altres
    credencials al repositori.** La informació d'accés de les màquines
    de pràctiques s'ha de gestionar de manera segura i d'acord amb les
    indicacions del professorat.

## 9. Propostes de millora futures

Aquest apartat recollirà possibles millores per a una versió posterior
de la infraestructura. Entre les propostes inicials hi ha:

-   Optimitzar el model de dades i les relacions de la base de dades.
-   Implementar operacions CRUD mitjançant rutes d'una API REST.
-   Configurar DNS i un domini públic, si escau.
-   Estudiar la incorporació d'un honeypot a la DMZ.
-   Automatitzar el desplegament amb scripts.
-   Estudiar mesures addicionals per protegir l'accés SSH, com ara *port
    knocking*.
-   Configurar certificats i noms de domini per als serveis web i FTP,
    segons la viabilitat i els requisits del projecte.

Aquestes propostes no impliquen que les funcionalitats ja estiguin
implementades.

## 10. Historial de canvis

  Versió   Canvis
  -------- ---------------------------------------------
  `0.1`    Creació de l'estructura inicial del README.

------------------------------------------------------------------------

*Document en construcció. S'anirà ampliant a mesura que es desplegui i
es validi la infraestructura.*
