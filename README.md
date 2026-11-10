# Airton → ESPHome → Home Assistant

Piloter des climatisations **Airton** (monosplits et bi-splits) en local depuis Home Assistant, **sans les dongles Wi-Fi Airton** (≈ 99 € pièce) et sans cloud, avec un simple ESP8266 / ESP32 sous ESPHome.

Le dépôt contient aussi un **superviseur de chauffage** pour Home Assistant (voir [la section dédiée](#le-superviseur-de-chauffage)). Il décide en permanence qui chauffe, les clims (pompes à chaleur) ou la chaudière gaz pilotée par Tado, et à quelle température, selon le tarif **EDF Tempo**, la **présence** et le **surplus solaire**. S'y ajoute la gestion de l'énergie : batterie, chauffe-eau, EMHASS. L'histoire du projet est racontée dans le [journal](docs/journal.md).

> Projet personnel, testé sur 2 monosplits et un groupe extérieur bi-split alimentant 4 unités intérieures. Aucune affiliation avec Airton. Ouvrir une clim et y brancher un montage maison se fait à vos risques.

## Ce que ça apporte

| | Monosplit (module USB Tuya) | Bi-split (module ACW02) |
|---|---|---|
| Marche / modes (dont **Auto**) / consigne | ✅ | ✅ |
| Ventilation, volets vertical et horizontal | ✅ | ✅ |
| Nuit, purificateur, écran | ✅ | ✅ (+ eco, nettoyage) |
| Hors-gel | ✅ (DP111) | — |
| Température ambiante | ✅ (sonde du split) | ✅ (sonde du split) |
| Consommation | ⚠️ **estimée** (la carte ne la fournit pas) | ✅ **compteur réel** de l'unité (pas de 0,1 kWh) |
| Couper le bip | ❌ impossible par le Wi-Fi | ✅ commande silencieuse |

## Le superviseur de chauffage

C'est le cerveau de l'installation : l'automatisation **`chauffage_superviseur_tempo`**, dans [`homeassistant/packages/chauffage_superviseur.yaml`](homeassistant/packages/chauffage_superviseur.yaml). À chaque changement de source, de consigne, de fin de dérogation, et au démarrage de Home Assistant, il repasse sur chaque pièce et applique la règle suivante :

| Situation (`sensor.chauffage_source_active`) | Clims | Chaudière gaz (Tado) |
|---|---|---|
| **Hiver**, sauf heures pleines d'un jour rouge → « Clim chauffage » | chauffage à la consigne cible (chambres : consigne chambres) | **arrêtée** |
| **Hiver, jour rouge de 6 h à 22 h** → « Gaz (Tado) » | **en veille** : chauffage à 16 °C, leur minimum, sans les éteindre (certaines unités coupent l'alimentation du module à l'arrêt) | **auto** : Tado gère seul, avec son anticipation et sa géolocalisation |
| **Été** → « Clim été » | monosplits en **Auto**, bi-splits en **Froid** | arrêtée |
| **Arrêt** (mi-saison) | éteintes | auto |

La consigne (`sensor.chauffage_consigne_cible`) suit ces règles :

- **jour**, **nuit** ou **absence** selon la présence, toutes réglables (ici 22, 18 et 16 °C) ;
- le confort démarre **avant le réveil** : 7 h en semaine, 8 h le week-end, avec une anticipation réglable ;
- **+1,5 °C de 4 h à 6 h** (heures creuses) la veille d'un jour rouge, pour stocker de la chaleur ;
- les **chambres** ont une consigne propre en journée (19 °C de 9 h à 17 h quand on est là) ;
- **l'été**, consignes été (26, 25 ou 28 °C selon jour, nuit ou absence), **−1 °C en cas de surplus solaire**.

Ce qu'il respecte :

- une clim réglée à la main dans Home Assistant passe en **dérogation pendant 3 h** et le superviseur ne la touche plus. Le bouton « Tout en auto » rend la main à toutes les pièces ;
- il n'envoie un ordre **que si l'état diffère**, avec 2 s entre deux clims ;
- un interrupteur général (`input_boolean.superviseur_chauffage`) le met en pause.

### Ses dépendances

**À installer :**

- **ESPHome** (les clims de ce dépôt) ;
- **Tado** (une entité `climate` par pièce, plus `sensor.maison_mode_tado`) ;
- **RTE Tempo** (HACS : couleur du jour, du lendemain, heures creuses) ;
- l'**application compagnon** (`person.*`) ;
- un **compteur électrique** qui donne `sensor.reseau_puissance` en W, négatif en injection.

**Fournies par les packages de ce dépôt, à copier ensemble :**

| Fichier | Contenu |
|---|---|
| [`chauffage_superviseur.yaml`](homeassistant/packages/chauffage_superviseur.yaml) | le superviseur ; l'interrupteur général, la saison, les consignes et les horaires ; les 3 capteurs de décision (source active, consigne cible, consigne chambres) ; les minuteurs et l'automatisation de dérogation ; la détection du surplus solaire ; le script « Tout en auto » |
| [`presence.yaml`](homeassistant/packages/presence.yaml) | `binary_sensor.presence_maison` : fusion Tado + téléphones (le dernier changement l'emporte) et forçage présents / absents |
| [`clims_conso.yaml`](homeassistant/packages/clims_conso.yaml) | consommation du jour par clim (pour le tableau de bord) |

**À adapter :** la **table des pièces**, en tête du superviseur. Une ligne par pièce : la clim, la vanne Tado de la même pièce, le minuteur de dérogation et le type `mono` ou `bi` :

```yaml
pieces:
  - { clim: climate.clim_salon,     tado: climate.salon,     timer: timer.derogation_clim_salon,     type: mono }
  - { clim: climate.clim_chambre_1, tado: climate.chambre_1, timer: timer.derogation_clim_chambre_1, type: bi }
```

Détail de chaque entité et graphe des dépendances : [docs/dependances.md](docs/dependances.md). Logique complète : [docs/orchestrateur.md](docs/orchestrateur.md).

## Contenu

```
esphome/
  airton-monosplit.yaml      package monosplit (Tuya MCU 55 AA, ESP8266)
  airton-bisplit.yaml        package bi-split (protocole 7A 7A, ESP32 + composant ACW02)
  clim-salon.yaml ...        un petit fichier par appareil (nom + broches)
  secrets.example.yaml
homeassistant/
  packages/chauffage_superviseur.yaml le SUPERVISEUR + réglages, capteurs de décision, dérogations
  packages/presence.yaml              présence (fusion Tado + téléphones, forçage)
  packages/clims_conso.yaml           consommation du jour par clim
  packages/energie_tempo.yaml         Tempo, batterie, chauffe-eau, EMHASS, alertes
  dashboard/thermostats.yaml          vue "Thermostats" (Mushroom)
  recorder_exemple.yaml               enregistreur allégé
docs/
  journal.md                    l'histoire du projet, étape par étape
  protocole-tuya-monosplit.md   datapoints, trames capturées, bugs de la carte
  protocole-7a7a-bisplit.md     format des trames, octets décodés, compteur d'énergie
  orchestrateur.md              logique du superviseur : Tempo / gaz / présence / été
  dependances.md                tout ce dont le superviseur a besoin, et la table des pièces
  energie.md                    Tempo, batterie, chauffe-eau, EMHASS
  materiel.md                   matériel, câblage, flashage
  depannage.md                  FAQ, capture de trames, base de données corrompue
```

## Démarrage rapide

1. **Matériel et câblage** : voir [docs/materiel.md](docs/materiel.md).
2. **ESPHome** : copier le dossier `esphome/` dans le répertoire ESPHome de Home Assistant, créer `secrets.yaml` à partir de `secrets.example.yaml`, adapter `clim-*.yaml` (nom, broches), puis installer.
3. **Home Assistant** : activer les packages dans `configuration.yaml`
   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```
   puis copier `chauffage_superviseur.yaml`, `presence.yaml` et `clims_conso.yaml` (dossier `homeassistant/packages/`). Adapter ensuite la table des pièces du superviseur et les entités externes : voir [docs/dependances.md](docs/dependances.md).
4. **Tableau de bord** : coller `homeassistant/dashboard/thermostats.yaml` dans l'éditeur YAML d'un tableau de bord (cartes HACS : Mushroom, mini-graph-card, card-mod).

## Points importants (découverts en route)

- **Monosplit, consigne** : la carte renvoie la consigne en °C entiers mais l'attend **en dixièmes à l'écriture** (210 = 21 °C). Envoyer « 21 » affiche « 02 » sur la clim.
- **Monosplit, mode Auto** : la plateforme `climate: tuya` d'ESPHome ne le propose pas → climate `template` piloté par les datapoints.
- **Monosplit, température** : le DP3 alterne avec une seconde valeur non documentée (4, 14…) → filtre 5–45 °C et capteur séparé.
- **Monosplit, octet parasite** : la carte insère un `0x55` en trop après certaines trames, ce qui fait perdre la trame suivante à ESPHome (DP12 et DP15 n'arrivent jamais). Détails dans [la doc du protocole](docs/protocole-tuya-monosplit.md).
- **Monosplit, hors-gel (DP111)** : le couper **éteint** la clim, elle ne revient pas à son mode précédent.
- **Bi-split, énergie** : chaque unité intérieure envoie un compteur d'énergie cumulé (trames de 18 et 34 octets) que le composant ACW02 ignore : un décodeur passif le récupère.
- **Bi-split, groupe extérieur partagé** : éviter de mettre une unité en chaud et l'autre en froid ; l'été, l'orchestrateur met les bi-splits en Froid (pas en Auto).
- **Température « I-Feel » de la télécommande** : elle n'est jamais transmise au module Wi-Fi, sur aucun des deux modèles.

## Remerciements

- [devildant/acw02_esphome](https://github.com/devildant/acw02_esphome) pour le pilotage des bi-splits (protocole 7A 7A).
- [ESPHome](https://esphome.io) et son composant [Tuya MCU](https://esphome.io/components/tuya.html).

## Licence

MIT
