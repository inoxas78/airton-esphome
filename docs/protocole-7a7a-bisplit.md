# Bi-split Airton : protocole 7A 7A

Les unités intérieures des bi-splits et multi-splits utilisent un autre protocole que les monosplits, celui du module Wi-Fi « ACW02 ». Le pilotage est assuré par le composant [devildant/acw02_esphome](https://github.com/devildant/acw02_esphome). Ce document complète son décodage avec ce qui a été relevé sur 4 unités intérieures reliées à un même groupe extérieur.

- Liaison : UART **9600 bauds, 8N1**
- Contrôle d'intégrité : **CRC16 Modbus** (valeur initiale 0xFFFF, polynôme 0xA001) calculé sur toute la trame sauf les 2 derniers octets, qui le contiennent, **octet de poids fort en premier**

## En-tête commun

```
7A 7A | destinataire | expéditeur | longueur totale | 00 00 | commande | 0A 0A | données… | CRC (2 octets)
```

Adresses : `21` = module Wi-Fi (l'ESP), `D5` = unité intérieure, `D1` = une seconde source, très probablement le groupe extérieur ou la carte de puissance.

## Trames observées

| Sens | Longueur | Commande | Période | Rôle |
|---|---|---|---|---|
| module → unité | 12 | `AB` | ~1 min | maintien de la liaison |
| module → unité | 12 | `A2` | sur demande | demande d'état |
| module → unité | 24 | — | sur ordre | commande (mode, consigne, options) |
| unité → module | 34 | `A3` | réponse à A2, et régulièrement | **état complet** |
| unité → module | 18 | `AC` | réponse à AB | **compteurs** |
| unité → module | 28 | `A6` | avec A3 | alertes et défauts |
| D1 → module | 13 | `A4` | 10 s | signal de vie (données toujours `00`) |

Exemples bruts, avec CRC vérifiés :

```
TX  7A 7A 21 D5 0C 00 00 AB 0A 0A FC F9
TX  7A 7A 21 D5 0C 00 00 A2 0A 0A FE 29
RX  7A 7A D5 21 22 00 00 A3 0A 0A 15 00 00 0C 03 06 C0 00 00 00 00 04 64 01 52 44 42 31 31 33 37 30 E6 44
RX  7A 7A D5 21 12 00 00 AC 0A 0A 00 89 00 01 52 00 5B 40
RX  7A 7A D5 21 1C 00 00 A6 0A 0A 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 52 0B
RX  7A 7A D1 21 0D 00 00 A4 0A 0A 00 25 25
```

## Trame d'état (34 octets, commande A3)

Décodée pour une unité en chauffage, consigne 19 °C, pièce à 21,0 °C :

| Octet(s) | Exemple | Signification |
|---|---|---|
| 10-11 | `15 00` | Température ambiante : partie entière, puis dixièmes (21,0 °C) |
| 12 | `00` | Toujours 0 |
| 13 | `0C` | bit 3 = marche ; bits 0-2 = mode ; bits 4-7 = ventilation |
| 14 | `03` | Consigne : 16 + valeur (19 °C) ; bit 6 = ventilation silencieuse ; table spéciale en °F |
| 15 | `06` | bits 0-3 = volet vertical ; bits 4-7 = volet horizontal |
| 16 | `C0` | Options : 0x01 eco, 0x02 nuit, 0x04 ordre venant de la télécommande, 0x10 nettoyage, 0x40 purificateur, 0x80 écran |
| 17-20 | `00…` | Toujours 0 |
| 21-22 | `04 64` | **1124 sur les 4 unités** : une constante (modèle ou capacité) |
| 23-24 | `01 52` | **Compteur d'énergie** de l'unité : 338 → 33,8 kWh |
| 25-31 | `44 42 31 31 33 37 30` | Texte ASCII **« DB11370 »** : probablement la référence de la carte |
| 32-33 | `E6 44` | CRC |

## Trame compteurs (18 octets, commande AC)

| Octet(s) | Exemple | Signification |
|---|---|---|
| 10-11 | `00 89` | **Compteur d'heures** : +24 par jour, à ±1 près la même valeur sur toutes les unités. Probablement les heures écoulées depuis la dernière mise sous tension |
| 12 | `00` | Toujours 0 |
| 13-14 | `01 52` | **Compteur d'énergie** de l'unité, par pas de 0,1 kWh (identique aux octets 23-24 de la trame A3) |
| 15 | `00` | Toujours 0 |
| 16-17 | `5B 40` | CRC |

Le composant ACW02 ignore cette trame. Le package `airton-bisplit.yaml` la décode de façon passive (sans rien envoyer) et en tire :

- **Énergie consommée**, en kWh (`total_increasing`), utilisable directement dans le tableau de bord Énergie ;
- **Puissance moyenne** = 100 Wh ÷ temps écoulé entre deux pas du compteur. Si le compteur n'avance plus, cette valeur est plafonnée puis ramenée à 0 au bout de 2 h.

## Trame alertes (28 octets, commande A6)

- Octet 10 = alerte : `0x80` = filtre à nettoyer.
- Octet 12 = défaut : `0x04` = conflit de mode (unités d'un même groupe extérieur en chaud et en froid).

## Ce qui n'est pas transmis

- La température de la télécommande (mode I-Feel).
- Une puissance instantanée : seul le compteur cumulé existe.

## Bip

Le bip se coupe par le bit 0 de l'octet 16 des **trames de commande**, ce que le composant ACW02 propose sous le nom « commande silencieuse ». Certaines unités bipent malgré tout sur la commande d'allumage : le composant prévoit pour ce cas un délai réglable.
