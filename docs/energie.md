# Énergie : Tempo, batterie, chauffe-eau, EMHASS

Fichier : [`homeassistant/packages/energie_tempo.yaml`](../homeassistant/packages/energie_tempo.yaml)

L'installation de référence comprend :

- un abonnement **EDF Tempo** ;
- des panneaux solaires avec un onduleur **Huawei** et sa batterie ;
- un **chauffe-eau électrique**, ajouté pour avoir une eau plus chaude que la production gaz et pour la chauffer au solaire ou la nuit ;
- une **chaudière gaz** (Saunier Duval Thema F25E) pilotée par Tado ;
- les clims décrites dans le reste du dépôt.

## Tarif Tempo

- Couleurs : intégration **RTE Tempo**.
- Prix officiels : intégration **Tarif EDF**. Ses capteurs `sensor.tarif_<couleur>_tempo_heures_<creuses|pleines>_ttc` remplacent avantageusement des `input_number` tenus à la main.
- Un « jour Tempo » va de **6 h à 6 h** ; les heures creuses vont de **22 h à 6 h**.
- `EDF Tempo Prix Actuel` donne le prix en cours, `EDF Tempo Prochain Prix` celui de la période suivante.

Grille au 1er août 2026, en €/kWh TTC (heures creuses / heures pleines) :

| Bleu | Blanc | Rouge |
|---|---|---|
| 0,1356 / 0,1654 | 0,1536 / 0,1921 | 0,1615 / **0,7295** |

Les heures pleines d'un jour rouge coûtent environ 4,4 fois plus que les heures pleines d'un jour bleu. C'est là que tout se joue : le chauffage bascule sur le gaz, la batterie est pleine et les gros appareils attendent.

## Batterie

- **Veille d'un jour rouge** : charge forcée à 100 % pendant les heures creuses.
- **Journée rouge** : si la batterie descend sous 6,5 %, petite recharge de sauvetage jusqu'à 13 %.
- **6 h** : retour en mode autoconsommation.

## Chauffe-eau

La décision se prend à 22 h : chauffer la nuit (en heures creuses) ou attendre le solaire du lendemain. Elle repose sur :

- le **stock d'énergie** du ballon, estimé ailleurs (par exemple avec des débitmètres ESPHome) ;
- la **météo** et la **production solaire prévue** (Solcast) pour le lendemain ;
- la **couleur Tempo** du lendemain.

En journée, le ballon chauffe dès que l'injection dépasse 1 kW pendant 2 min, si la batterie est au-dessus de 50 %. Plusieurs sécurités s'ajoutent :

- relance la nuit si le stock passe sous 10,5 kWh, par exemple après un bain tardif ;
- coupure si la batterie passe sous 40 % en journée ;
- coupure quand le ballon est plein (14,5 kWh ou 58 °C) ;
- 4 h de chauffe maximum par jour.

## EMHASS

[EMHASS](https://emhass.readthedocs.io) calcule toutes les 30 min un plan optimal sur 24 h (48 demi-heures) à partir :

- des **prix Tempo réels** : le `load_cost_forecast` est reconstruit à chaque appel, couleur du jour et du lendemain, heures creuses ou pleines ;
- des prévisions solaires, rafraîchies à 5 h 30 et 13 h ;
- de l'état de charge de la batterie.

Il est utilisé **sans pilotage** : il publie ses capteurs (plan de charge, coût prévu) comme aide à la décision, et les automatisations ci-dessus restent maîtresses.

Points à surveiller pour qu'il donne des résultats crédibles :

- `load_negative` doit valoir `false` si le capteur de consommation est positif ;
- il faut **plusieurs jours d'historique** de consommation pour la prévision de charge ;
- le capteur de production solaire doit être une **puissance en W**, pas un cumul.

## Enregistreur

[`homeassistant/recorder_exemple.yaml`](../homeassistant/recorder_exemple.yaml) exclut les diagnostics et les capteurs très bavards (RSSI, uptime, radars de présence, distances…). Moins d'écritures sur le SSD, une base plus petite, et des sauvegardes plus rapides et plus sûres. Voir [depannage.md](depannage.md) pour la récupération d'une base corrompue.
