# Matériel et branchement

> Avant tout branchement, **mesurer les tensions** sur le connecteur de votre unité. Les brochages ci-dessous sont ceux de cette installation : vérifiez-les sur votre modèle.

## Monosplits (module USB)

Le dongle Wi-Fi Airton des monosplits se branche dans une prise **USB-A** de l'unité intérieure. Ce n'est pas un vrai port USB : on y trouve l'alimentation **5 V** et la masse, et les deux lignes de données transportent un **UART à 9600 bauds** (protocole Tuya MCU).

| Élément | Choix |
|---|---|
| Microcontrôleur | ESP8266, module ESP-12E (`board: esp12e`) |
| Montage | fils soudés directement entre la carte de la clim (au niveau de la prise USB) et l'ESP |
| Alimentation | 5 V de la prise USB → **régulateur 3,3 V** → VCC de l'ESP |
| UART | matériel : GPIO1 (TX) / GPIO3 (RX), comme sur `clim-salon.yaml` ; ou logiciel : GPIO12 (TX) / GPIO14 (RX), comme sur `clim-bureau.yaml` |

### Schéma

```
 Prise USB de la clim                    ESP8266 (ESP-12E)
 ────────────────────                    ─────────────────
 5 V   ──► régulateur 3,3 V (LDO) ─────► VCC
 GND   ────────────────────────────────► GND
 ligne de données « TX clim »  ────────► RX  (GPIO3 ou GPIO14)
 ligne de données « RX clim »  ◄──────── TX  (GPIO1 ou GPIO12)
                                         EN  ── 10 kΩ ── 3,3 V
                                         GPIO15 ── 10 kΩ ── GND
                                         GPIO0, GPIO2 ── 10 kΩ ── 3,3 V
```

Le module ESP-12E nu a besoin des résistances de démarrage indiquées (EN, GPIO0, GPIO2 et GPIO15). Une carte de développement (Wemos D1 mini, NodeMCU…) les intègre déjà, et sa broche **5V** passe par son propre régulateur.

### Alimentation : pourquoi un régulateur

La puce ESP8266 fonctionne en **3,3 V** : la fiche technique d'Espressif donne 2,5 à 3,6 V. Brancher le 5 V directement sur le VCC d'un ESP-12E nu peut marcher un temps, mais hors des spécifications. On ajoute donc un régulateur basse chute 3,3 V, capable de fournir 500 mA ou plus, car l'ESP tire des pointes d'environ 300 mA à l'émission Wi-Fi. Par exemple un AMS1117-3.3 ou un HT7333, avec un condensateur de 10 µF en entrée et un autre en sortie.

### Niveau des lignes de données

Mesurer la tension au repos de la ligne qui va vers le RX de l'ESP. Si elle est à 5 V, ajouter un pont diviseur (par exemple 1 kΩ en série et 2 kΩ vers la masse) ou un adaptateur de niveau. Le TX de l'ESP, en 3,3 V, est en général bien compris par une carte 5 V.

### Remarques

- Avec l'UART matériel, le journal série doit être coupé (`logger: baud_rate: 0`). Les journaux restent lisibles par le Wi-Fi.
- Si rien ne remonte, **inverser TX et RX**.

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
