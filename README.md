# P0.0 — Desplegament d'infraestructura

> **Mòdul:** 0379 — Projecte intermodular d'administració de sistemes informàtics en xarxa  
> **Cicle:** ASIX  
> **Grup:** G02  
> **Estat:** En desenvolupament

## Descripció

Aquest repositori centralitza el codi, la configuració, les proves i la documentació del projecte **P0.0 — Desplegament d'infraestructura**. L'objectiu és dissenyar, desplegar i validar una infraestructura per a una aplicació multicapa amb serveis de xarxa i sistemes, una base de dades i una aplicació web.

La documentació es manté progressivament a partir de les tasques realitzades i les evidències disponibles. Els elements marcats com a **pendent**, **previst** o **per verificar** no s'han de considerar implementats.

## Navegació

- [Documentació general](docs/README.md)
- [Objectiu i abast del projecte](docs/project/overview.md)
- [Sprints](docs/sprints/README.md)
- [Inventari inicial de màquines](docs/infrastructure/machines.md)
- [Procediment inicial de recuperació de Proxmox](docs/infrastructure/proxmox-recovery.md)
- [Relació provisional amb els resultats d'aprenentatge](docs/evaluation/ra-mapping.md)
- [Índex d'evidències](evidence/README.md)

## Objectius del projecte

- [ ] Dissenyar i documentar l'arquitectura de la infraestructura.
- [ ] Definir les xarxes, l'adreçament IP i l'encaminament.
- [ ] Desplegar i configurar els serveis requerits (web, SSH, base de dades, DHCP, DNS i FTP, segons el disseny aprovat).
- [ ] Importar a MySQL les dades d'equipaments educatius de Barcelona.
- [ ] Preparar una aplicació que consulti les dades emmagatzemades.
- [ ] Documentar el desplegament, les proves i la recuperació de les màquines i serveis.
- [ ] Mantenir el seguiment dels sprints, les incidències i les decisions tècniques.
- [ ] Relacionar les evidències amb els requisits i els resultats d'aprenentatge.

## Arquitectura prevista

La proposta inicial contempla tres segments de xarxa: **DMZ**, **intranet** i **NAT**. L'adreçament, les interfícies, les regles i la ubicació definitiva de cada servei s'han de confirmar a partir de la configuració real.

| Component | Identificador observat / previst | Funció prevista | Estat documental |
|---|---|---|---|
| Router | 103 — R-NCC | Interconnexió de xarxes | Identificat; configuració pendent de documentar |
| Servidor de base de dades | 101 — BasedeDatos | MySQL i dades del projecte | Identificat; configuració pendent de verificar |
| Servidor FTP | 104 — Servidor-FTP | Transferència de fitxers | Identificat; configuració pendent de documentar |
| Servidor web | 105 — WEB-NCC | Aplicació web | Identificat; configuració pendent de documentar |
| Client / equip B-NCC | 106 — B-NCC | Funció per confirmar | Identificat; funció pendent de verificar |
| Contenidor o entorn de desplegament | 107 — despliegue | Proves de desplegament | Identificat; tecnologia i funció exactes per confirmar |
| Client Ubuntu | 102 — clienteubuntu | Client Linux | Identificat |
| Entorn de prova | 100 — prueba | Proves | Identificat; funció exacta per confirmar |

> **Important:** els noms i IDs anteriors provenen de la vista de Proxmox disponible. Cal verificar les funcions, el tipus de recurs (VM o contenidor), el sistema operatiu i els paràmetres reals abans de donar-los per definitius.

## Dades obertes

Es preveu treballar amb el conjunt de dades d'equipaments d'educació de Barcelona:

- [Portal de dades obertes de Barcelona — equipaments d'educació](https://opendata-ajuntament.barcelona.cat/data/es/dataset/equipament-educacio/)

La font, la data de descàrrega, el format, el procés d'importació i les comprovacions de qualitat s'han de registrar quan es documenti la importació.

## Organització del repositori

```text
.
├── README.md
├── docs/
│   ├── project/
│   ├── sprints/
│   ├── infrastructure/
│   ├── services/
│   ├── testing/
│   └── evaluation/
├── infrastructure/
│   ├── diagrams/
│   └── inventory/
├── services/
├── application/
├── database/
├── scripts/
├── tests/
└── evidence/
    ├── sprint-01/
    └── sprint-02/
```

Aquesta estructura s'ampliarà a mesura que hi hagi contingut real per afegir. No cal crear fitxers buits ni documentar com a implementat cap servei que encara no s'hagi verificat.

## Sprints i seguiment

El seguiment de cada sprint hauria d'incloure objectius, tasques i responsables, treball realitzat, incidències, proves i resultats, revisió i retrospectiva. Consulteu [l'índex de sprints](docs/sprints/README.md).

## Seguretat

- No s'han de pujar contrasenyes, tokens, claus privades, fitxers `.env` reals ni dades personals.
- Els fitxers d'exemple han de contenir valors ficticis, com ara `.env.example`.
- Abans de publicar una captura, cal revisar que no mostri credencials, tokens o informació sensible.
- Les còpies de seguretat s'han d'emmagatzemar en una ubicació autoritzada i no s'han de pujar al repositori si contenen dades o secrets.

## Estat actual i pròxims passos

Aquesta documentació és incremental. El primer objectiu és consolidar l'inventari, descriure la infraestructura existent i documentar el procediment de recuperació. Després s'afegiran els detalls verificats de xarxa, serveis, aplicació i proves.

- [ ] Verificar inventari, tipus de recurs i funció de cada màquina.
- [ ] Afegir el diagrama de xarxa amb valors reals.
- [ ] Registrar IP, prefix, gateway, DNS i VLAN quan correspongui.
- [ ] Documentar instal·lació i verificació de cada servei.
- [ ] Afegir evidències seleccionades del Sprint 1.
- [ ] Completar les proves i el registre d'incidències.
