# Airton → ESPHome → Home Assistant

Piloter des climatisations **Airton** (monosplits et bi-splits) en local depuis Home Assistant, **sans les dongles Wi-Fi Airton** (≈ 99 € pièce) et sans cloud, avec un simple ESP8266 / ESP32 sous ESPHome.

Le dépôt contient aussi un **orchestrateur de chauffage** complet : clims (pompe à chaleur) + chaudière gaz pilotée par Tado, en fonction du tarif **EDF Tempo**, de la **présence** et du **surplus solaire**. Ainsi que la gestion énergie associée : batterie, chauffe-eau, EMHASS. L'histoire du projet est racontée dans le [journal](docs/journal.md).

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

## Contenu

```
esphome/
  airton-monosplit.yaml      package monosplit (Tuya MCU 55 AA, ESP8266)
  airton-bisplit.yaml        package bi-split (protocole 7A 7A, ESP32 + composant ACW02)
  clim-salon.yaml ...        un petit fichier par appareil (nom + broches)
  secrets.example.yaml
homeassistant/
  packages/clims_orchestrateur.yaml   helpers, capteurs, automatisations, script
  packages/energie_tempo.yaml         Tempo, batterie, chauffe-eau, EMHASS, alertes
  dashboard/thermostats.yaml          vue "Thermostats" (Mushroom)
  recorder_exemple.yaml               enregistreur allégé
docs/
  journal.md                    l'histoire du projet, étape par étape
  protocole-tuya-monosplit.md   datapoints, trames capturées, bugs de la carte
  protocole-7a7a-bisplit.md     format des trames, octets décodés, compteur d'énergie
  orchestrateur.md              logique Tempo / gaz / présence / été
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
   puis copier `homeassistant/packages/clims_orchestrateur.yaml` et adapter les entités en tête de fichier (Tado, Tempo, personnes, compteur).
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
