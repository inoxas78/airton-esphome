# Matériel et branchement

> Avant tout branchement, **mesurer les tensions** sur le connecteur de votre unité. Les brochages ci-dessous sont ceux de cette installation : vérifiez-les sur votre modèle.

## Monosplits (module USB)

Le dongle Wi-Fi Airton des monosplits se branche dans une prise **USB-A** de l'unité intérieure. Ce n'est pas un vrai port USB : on y trouve l'alimentation **5 V** et la masse, et les deux lignes de données transportent un **UART à 9600 bauds** (protocole Tuya MCU).

| Élément | Choix |
|---|---|
| Microcontrôleur | ESP8266, module ESP-12E (`board: esp12e`) |
| Alimentation | 5 V de la prise USB, régulateur 3,3 V |
| UART | matériel : GPIO1 (TX) / GPIO3 (RX), comme sur `clim-salon.yaml` ; ou logiciel : GPIO12 (TX) / GPIO14 (RX), comme sur `clim-bureau.yaml` |

Remarques :

- Avec l'UART matériel, le journal série doit être coupé (`logger: baud_rate: 0`). Les journaux restent lisibles par le Wi-Fi.
- Si rien ne remonte, **inverser TX et RX**.
- Contrôler le niveau logique des lignes de données avant de les relier à l'ESP, qui fonctionne en 3,3 V.

## Bi-splits / multi-splits (module ACW02)

| Élément | Choix |
|---|---|
| Microcontrôleur | ESP32, D1 mini WROOM-32 (`board: esp32dev`) |
| UART | UART2 : GPIO17 (TX) / GPIO16 (RX), 9600 bauds |
| Branchement | à la place du module Wi-Fi ACW02, selon la documentation de [devildant/acw02_esphome](https://github.com/devildant/acw02_esphome) |

Chaque unité intérieure a son propre ESP, même si elles partagent un groupe extérieur.

## Flashage

1. Premier flash par câble, avec un adaptateur USB-série 3,3 V, ou par le navigateur via ESPHome Web.
2. Ensuite, mises à jour **par le Wi-Fi** (OTA) depuis ESPHome Builder dans Home Assistant.
3. Ajouter l'appareil dans Home Assistant (intégration ESPHome, découverte automatique).

## Noms des entités

Les fichiers `clim-*.yaml` produisent des entités du type `climate.clim_salon_consigne_climatisation` ou `sensor.clim_bisplit_1_energie_consommee`. Dans cette installation, elles ont été renommées dans Home Assistant (par exemple en `climate.clim_salon` et `climate.clim_chambre_1`) pour correspondre à l'orchestrateur et au tableau de bord. Adaptez les noms d'un côté ou de l'autre.
