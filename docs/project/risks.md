# Registre de riscos i pla de contingència

> Registre inicial per completar amb l'equip. Els riscos següents són escenaris que cal valorar, no incidències que s'hagin produït necessàriament.

## Escala de valoració

- Probabilitat: baixa, mitjana o alta.
- Impacte: baix, mitjà o alt.
- Prioritat: resultat de valorar conjuntament probabilitat i impacte.

## Registre inicial

| ID | Risc | Probabilitat | Impacte | Prevenció | Resposta / contingència | Responsable |
|---|---|---|---|---|---|---|
| R01 | Pèrdua o corrupció de la màquina virtual | Per valorar | Alt | Còpies de seguretat verificades i registre de canvis | Aïllar el recurs afectat i restaurar una còpia validada amb autorització | Per assignar |
| R02 | Fallada de connectivitat entre segments o clients | Per valorar | Mitjà/alt | Documentar IP, gateway, DNS, VLAN i regles | Comprovar enllaç, adreçament i encaminament; revertir l'últim canvi si escau | Per assignar |
| R03 | Servei web o contenidor aturat | Per valorar | Alt | Proves després dels canvis i registre de configuració | Revisar estat i logs, recuperar el servei i repetir les proves | Per assignar |
| R04 | Base de dades inaccessible o dades inconsistents | Per valorar | Alt | Còpies de dades i proves d'importació/restauració | Aturar operacions que puguin agreujar el problema, preservar logs i restaurar segons procediment | Per assignar |
| R05 | Pèrdua de credencials o exposició de secrets | Per valorar | Alt | No pujar secrets a Git; permisos mínims i credencials protegides | Revocar/rotar credencials exposades i revisar l'historial del repositori | Per assignar |
| R06 | Retard per dependències o manca de recursos | Per valorar | Mitjà | Planificació, responsables i revisió periòdica | Reprioritzar el backlog i comunicar l'impacte al grup | Per assignar |

## Pla de contingència

1. Detectar i registrar el símptoma, la data, l'hora i els recursos afectats.
2. Determinar l'abast de l'afectació sense fer canvis destructius.
3. Consultar els logs i les últimes modificacions.
4. Aplicar només accions autoritzades i documentades.
5. Verificar el servei amb proves definides.
6. Registrar la causa confirmada, la solució i les mesures per evitar que es repeteixi.

## Dades que ha de completar el grup

- Probabilitat real de cada risc.
- Impacte sobre el projecte i els terminis.
- Responsable de prevenció i resposta.
- Ubicació i periodicitat de les còpies de seguretat.
- Temps objectiu de recuperació, si el projecte l'exigeix.
- Evidència d'una restauració de prova.

No s'ha de declarar que hi ha un sistema de còpies o una alta disponibilitat operativa fins que s'hagi comprovat.
