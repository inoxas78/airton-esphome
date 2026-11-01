# Monosplit Airton : protocole Tuya MCU (55 AA)

Le module Wi-Fi d'origine des monosplits est un module Tuya branché sur le port USB de l'unité intérieure. La carte de la clim (le « MCU ») parle le **protocole série Tuya MCU** standard, que le composant [`tuya`](https://esphome.io/components/tuya.html) d'ESPHome sait gérer.

- Liaison : UART **9600 bauds, 8N1**
- Produit annoncé par la carte : `{"p":"pjnywabwywopufvb","v":"1.0.0","m":0}`

## Format d'une trame

```
55 AA | version | commande | longueur (2 octets) | données | somme de contrôle
```

- La version vaut `00` dans le sens module → carte, et `03` dans le sens carte → module.
- La somme de contrôle est la somme de tous les octets précédents, modulo 256.

Commandes observées :

| Commande | Sens | Rôle |
|---|---|---|
| `00` | module → carte, toutes les 15 s | battement de cœur (réponse `55 AA 03 00 00 01 01 04`) |
| `08` | module → carte | demande de l'état complet (`55 AA 00 08 00 00 07`) |
| `07` | carte → module | rapport d'un datapoint (DP) |
| `06` | module → carte | écriture d'un DP |

Un DP est codé ainsi : `id | type | longueur (2 octets) | valeur`. Types : `01` booléen, `02` entier sur 4 octets, `04` liste (enum), `05` masque de bits.

## Datapoints

Valeurs relevées le 1er novembre 2026, clim en chauffage, consigne 22 °C, pièce à 24,0 °C :

| DP | Type | Valeur | Signification |
|---|---|---|---|
| 1 | bool | 1 | Marche / arrêt |
| 2 | int | 22 | Consigne. **Lue en °C entiers, écrite en dixièmes** (210 = 21 °C) |
| 3 | int | 240 | Température ambiante ×10 (sonde du split). Alterne parfois avec une seconde valeur (voir plus bas) |
| 4 | enum | 3 | Mode : 0 Auto, 1 Froid, 2 Déshumidification, 3 Chauffage, 4 Ventilation |
| 5 | enum | 0 | Ventilation : 0 Auto, 1 à 5 = 20 à 100 %, 6 Silencieux, 7 Turbo |
| 8 | bool | 0 | Inconnu, sans effet visible (probablement Eco) |
| 12 | bool | 1 | Inconnu, **jamais reçu par ESPHome** (voir le bug plus bas) |
| 13 | bool | — | Écran LED, **en écriture seule** (jamais renvoyé) |
| 15 | enum | 0 | Inconnu, **jamais reçu par ESPHome** |
| 20 | bitmask | 0 | Défauts |
| 21 | enum | 1 | Inconnu |
| 22 | int | 1 | Inconnu |
| 101 à 104 | int | 1 | Bouche-trous : **aucune consommation n'est remontée** |
| 105 | enum | 0 | Inconnu |
| 106 | enum | 1 | Balayage horizontal (0 arrêt, 1 marche) |
| 107 | enum | 6 | Balayage vertical (0 arrêt, 1 marche, 2 à 6 = positions 1 à 5) |
| 108 | bool | 0 | Inconnu, sans effet visible |
| 109 | bool | 0 | Mode nuit |
| 110 | bool | 1 | Purificateur |
| 111 | bool | 0 | **Hors-gel.** À 1, la clim passe en hors-gel (vue comme Auto à 16 °C). Le remettre à 0 **éteint** la clim |
| 112 | enum | 1 | Inconnu |
| 113 | bitmask | 0 | Défauts / alertes |
| 114 | enum | 1 | Inconnu |
| 115 | bool | 1 | À 0, la clim affiche « LA » à l'écran |

Testés sans succès pour couper le bip : DP8, 12, 108, 111 et 115. **Le bip n'est pas pilotable** par le module Wi-Fi.

## Réponse complète à la commande 08

Trames brutes, toutes avec une somme de contrôle valide :

```
55 AA 03 07 00 05 01 01 00 01 01 12
55 AA 03 07 00 08 02 02 00 04 00 00 00 16 2F
55 AA 03 07 00 08 03 02 00 04 00 00 00 F0 0A
55 AA 03 07 00 05 04 04 00 01 03 1A
55 AA 03 07 00 05 05 04 00 01 00 18
55 AA 03 07 00 05 08 01 00 01 00 18 55        <- 0x55 en trop
55 AA 03 07 00 05 0C 01 00 01 01 1D 55        <- 0x55 en trop
55 AA 03 07 00 05 0F 04 00 01 00 22
55 AA 03 07 00 05 14 05 00 01 00 28
55 AA 03 07 00 05 15 04 00 01 01 29
55 AA 03 07 00 08 16 02 00 04 00 00 00 01 2E
55 AA 03 07 00 08 65 02 00 04 00 00 00 01 7D
55 AA 03 07 00 08 66 02 00 04 00 00 00 01 7E
55 AA 03 07 00 08 67 02 00 04 00 00 00 01 7F
55 AA 03 07 00 08 68 02 00 04 00 00 00 01 80
55 AA 03 07 00 05 69 04 00 01 00 7C
55 AA 03 07 00 05 6A 04 00 01 01 7E
55 AA 03 07 00 05 6B 04 00 01 06 84
55 AA 03 07 00 05 6C 01 00 01 00 7C
55 AA 03 07 00 05 6D 01 00 01 00 7D
55 AA 03 07 00 05 6E 01 00 01 01 7F
55 AA 03 07 00 05 6F 01 00 01 00 7F
55 AA 03 07 00 05 70 04 00 01 01 84
55 AA 03 07 00 05 71 05 00 01 00 85
55 AA 03 07 00 05 72 04 00 01 01 86
55 AA 03 07 00 05 73 01 00 01 01 84
```

## Bug de la carte : un octet 0x55 en trop

Après les trames DP8 et DP12, la carte envoie un `0x55` supplémentaire : `… 00 18 55 55 AA 03 07 …`.

Le parseur Tuya d'ESPHome lit alors `55 55`. Il juge la trame invalide et vide son tampon, en jetant au passage le `55` qui commençait la vraie trame suivante. **DP12 et DP15 ne sont donc jamais reçus**, et la liste des DP qu'ESPHome affiche au démarrage confirme qu'ils manquent. L'écriture de ces DP fonctionne normalement.

## La seconde valeur du DP3

Le DP3 alterne parfois très vite entre la température (×10) et une petite valeur : 4, ou 4 et 14 en alternance. Mélangées sans filtre, les deux donnent des moyennes absurdes (12,7 °C, 0,4 °C…). Dans la config :

- le capteur de température ne garde que les valeurs entre 50 et 450 (5 à 45 °C), puis fait une moyenne sur 30 s ;
- les valeurs inférieures à 50 sont publiées à part, dans « DP3 valeur secondaire (brut) ».

Cette seconde valeur n'est pas une température. On l'observe surtout quand la télécommande est en mode I-Feel, mais elle ne contient **pas** la température de la télécommande : celle-ci n'est transmise au module sur aucun DP.

## Capturer soi-même les trames

On peut ajouter temporairement dans la config :

```yaml
uart:
  id: uart_bus
  debug:
    direction: BOTH
    dummy_receiver: false
    after:
      timeout: 8ms
    sequence:
      - lambda: UARTDebug::log_hex(direction, bytes, ' ');

button:
  - platform: template
    name: "Capture - etat complet Tuya"
    on_press:
      - uart.write:
          id: uart_bus
          data: [0x55, 0xAA, 0x00, 0x08, 0x00, 0x00, 0x07]
```

Les journaux sont consultables sans Home Assistant, sur `http://<ip>/` (web_server), ou avec un `EventSource('/events')` dans la console du navigateur.
