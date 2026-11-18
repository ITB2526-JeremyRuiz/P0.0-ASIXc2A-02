# Prevenció de riscos laborals i riscos tècnics

## 1. Prevenció de riscos laborals

| Risc | Situació | Mesures preventives |
|---|---|---|
| Trastorns musculoesquelètics | Treball prolongat amb ordinador i postures estàtiques | Cadira regulable, pantalla a l'altura dels ulls, postura neutra i pauses actives |
| Fatiga visual | Ús continuat de pantalles, reflexos o il·luminació inadequada | Il·luminació uniforme, evitar reflexos i aplicar la regla 20-20-20 |
| Contacte elèctric | Manipulació de cablejat, regletes i equips | No manipular fonts obertes, evitar sobrecàrregues i desconnectar abans d'intervenir en maquinari |
| Caigudes i ensopegades | Cablejat a zones de pas | Canalitzar cables i mantenir lliures les vies de pas |
| Calor i soroll | Funcionament continu de servidors | Limitar la permanència a sales tècniques i garantir ventilació |
| Estrès operatiu | Incidències tècniques amb terminis ajustats | Planificar tasques, repartir càrrega i documentar procediments |

Referències normatives indicades al document del projecte: Llei 31/1995 de prevenció de riscos laborals i RD 488/1997 sobre pantalles de visualització de dades.

## 2. Riscos tècnics

| Risc | Mesures preventives i resposta |
|---|---|
| Pèrdua de connectivitat després de reiniciar Proxmox | Persistir les regles d'encaminament/NAT i comprovar forwarding, rutes i connectivitat després de reiniciar |
| DHCP bloquejat en contenidors | Executar DHCP/DNS al node Proxmox amb `dnsmasq`, segons la incidència documentada d'anti-MAC-spoofing |
| Caiguda o corrupció de MySQL | Mantenir còpies de les dades, verificar el volum Docker i validar la importació abans de donar-la per bona |
| Força bruta al portal | Aplicar `ppolicy` amb 3 intents fallits i 900 segons de bloqueig |
| Fallada d'un contenidor LXC | Disposar de snapshots/còpies abans de canvis crítics i validar els serveis després de restaurar |

## 3. Resposta davant incidències
1. Registrar data, hora, sistema afectat i símptoma.
2. Identificar l'abast abans de modificar configuracions.
3. Consultar logs i estat del servei.
4. Aplicar una acció de recuperació autoritzada.
5. Repetir les proves funcionals i de xarxa.
6. Registrar el resultat i l'evidència.

Les còpies programades, la retenció i els responsables de recuperació s'han de concretar segons la configuració real de l'entorn.
