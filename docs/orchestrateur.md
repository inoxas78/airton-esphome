# Orchestrateur chauffage / climatisation

Fichiers : [`chauffage_superviseur.yaml`](../homeassistant/packages/chauffage_superviseur.yaml) (le superviseur et ses dépendances), [`presence.yaml`](../homeassistant/packages/presence.yaml), [`clims_conso.yaml`](../homeassistant/packages/clims_conso.yaml). Liste complète des dépendances : [dependances.md](dependances.md).

Objectif : chauffer avec les clims (pompes à chaleur, nettement moins chères que le gaz la plupart du temps) et basculer sur la chaudière gaz pilotée par **Tado** uniquement quand l'électricité est hors de prix, c'est-à-dire en **heures pleines des jours rouges Tempo**. Le tout en tenant compte de la présence, des horaires et du soleil.

## Les trois capteurs qui décident

| Capteur | Valeurs | Règle |
|---|---|---|
| `sensor.chauffage_source_active` | Clim chauffage / Gaz (Tado) / Clim été / Arrêt | Saison choisie + couleur Tempo. Hiver : gaz si **jour rouge en heures pleines** (6 h – 22 h), sinon clims |
| `sensor.chauffage_consigne_cible` | °C | Jour / nuit / absence. En hiver, le confort démarre avant le réveil (anticipation réglable). +1,5 °C de 4 h à 6 h la veille d'un jour rouge (préchauffage en heures creuses). L'été : −1 °C en cas de surplus solaire |
| `sensor.chauffage_consigne_chambres` | °C | Chambres : consigne propre en journée (ex. 19 °C de 9 h à 17 h) quand la maison est occupée, sinon la consigne cible |

## Le superviseur

L'automatisation `chauffage_superviseur_tempo` parcourt une liste de pièces. Chaque pièce associe une clim, la vanne Tado de la même pièce, un minuteur de dérogation et un type (mono / bi). Pour chacune :

| Source | Clims | Tado |
|---|---|---|
| Clim chauffage | chauffage à la consigne (arrondie au degré supérieur ; chambres : consigne chambres) | arrêté |
| Gaz (Tado) | veille : chauffage à 16 °C (minimum), pas d'arrêt | **auto** : Tado gère seul, avec son anticipation et sa géolocalisation |
| Clim été | monosplits en **Auto**, bi-splits en **Froid** | arrêté |
| Arrêt | éteintes | auto |

Pourquoi les bi-splits en Froid et pas en Auto l'été ? Les unités d'un même groupe extérieur ne peuvent pas chauffer et refroidir en même temps (défaut « conflit de mode »). Le mode Froid évite ce risque.

Le superviseur ne renvoie un ordre que si l'état diffère, avec 2 s entre deux clims, pour ne pas saturer les cartes ni multiplier les bips.

**Boost Tado.** Un boost ou un réglage depuis l'application Tado met la zone en mode `heat`. Le superviseur ne touche jamais une zone dans cet état : Tado reprend son planning tout seul à la fin du boost (30 min). Sans cette règle, la moindre relance du superviseur (fin d'une dérogation, changement de consigne) remettait Tado en `auto` et annulait le boost.

**Clim qui revient.** Quand une clim redevient disponible (redémarrage du module), le superviseur repasse 30 s plus tard pour la remettre dans l'état voulu : certaines unités redémarrent éteintes.

## Dérogation manuelle

Si quelqu'un règle une clim à la main **depuis Home Assistant**, le superviseur laisse cette pièce tranquille pendant **3 h**. Techniquement : un appel de service `climate.*` porteur d'un `user_id` démarre `timer.derogation_<clim>`.

Pour reprendre la main avant la fin : toucher la carte « Réglage manuel » de la pièce (`timer.finish`), ou lancer le script `reprendre_pilotage_clims` qui libère toutes les pièces.

## Notifications des décisions

[`cerveau_notifications.yaml`](../homeassistant/packages/cerveau_notifications.yaml) explique chaque décision sur l'iPhone, dans le centre de notifications de Home Assistant (la dernière par catégorie) et dans le journal :

- **chauffage** : changement de source avec la raison Tempo, de consigne (jour, nuit, absence, réveil anticipé, préchauffage la veille d'un jour rouge), des chambres (école ou week-end), de présence (téléphones ou Tado), boost Tado détecté et sa fin, début et fin des réglages manuels des clims. Les changements simultanés sont regroupés en un seul message (15 s), et rien n'est envoyé si rien n'a vraiment changé ;
- **batterie** : charge de nuit lancée la veille d'un jour rouge, recharge de sauvetage, retour en autoconsommation ;
- **chauffe-eau** : décision de 22 h avec ses chiffres (Tempo et météo du lendemain, solaire prévu, stocks estimés), marche sur surplus, arrêt et sa raison.

Toutes passent par un seul script, `script.cerveau_notifier`. `input_boolean.cerveau_notifications` coupe le téléphone et le centre de notifications ; le journal, lui, est toujours rempli.

Exemple :

> 🔥 **Chauffage : Gaz (Tado)**
> Source : Clim chauffage → Gaz (Tado). Jour rouge en heures pleines (6 h – 22 h) : la chaudière gaz chauffe, Tado suit son planning. Les clims restent en veille à 16 °C.
> Consigne maison : 18 → 20 °C (journée, depuis 05:30 (réveil 06:30 − 60 min d'anticipation)).

## Présence

- `binary_sensor.presence_telephones` : quelqu'un est à la maison d'après l'application compagnon.
- `sensor.maison_mode_tado` : géolocalisation Tado (HOME / AWAY). Tado réagit parfois avec plusieurs minutes de retard.
- `presence_fusion_tado_telephones` : à l'**arrivée**, le premier des deux systèmes qui détecte quelqu'un l'emporte ; au **départ**, la maison ne passe absente que quand les téléphones des deux personnes sont partis. Un passage de Tado en AWAY est ignoré tant qu'un téléphone est à la maison.
- `input_select.mode_presence` : Auto (géolocalisation), Forcé présents ou Forcé absents. Le forçage est aussi envoyé à Tado (préréglage home / away / auto).

## Surplus solaire

`energie_surplus_solaire` active `input_boolean.surplus_solaire` quand la maison injecte plus de 800 W pendant 10 min, et le coupe quand l'injection retombe sous 100 W pendant 10 min. L'été, ce surplus abaisse la consigne de 1 °C pour pré-rafraîchir gratuitement.

## Réglages

Tous les réglages sont des helpers (consignes jour, nuit, absence, chambres, été, préchauffage, anticipation, horaires) exposés dans la vue [Thermostats](../homeassistant/dashboard/thermostats.yaml).

## À adapter

- Les entités Tado (`climate.salon`, `climate.chambre_1`…) et `sensor.maison_mode_tado`.
- `person.personne_1` et `person.personne_2`.
- `sensor.reseau_puissance` : la puissance au compteur, **négative en injection**.
- Tempo : intégration RTE Tempo (`sensor.rte_tempo_couleur_actuelle`, `sensor.rte_tempo_prochaine_couleur`, `binary_sensor.rte_tempo_heures_creuses`).
- La liste `pieces` du superviseur, et les minuteurs `timer.derogation_<object_id de la clim>` correspondants.
