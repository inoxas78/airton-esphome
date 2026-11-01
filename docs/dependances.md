# Dépendances du superviseur

Le superviseur (`chauffage_superviseur_tempo`, dans [`chauffage_superviseur.yaml`](../homeassistant/packages/chauffage_superviseur.yaml)) s'appuie sur trois catégories d'éléments.

## 1. Intégrations à installer

| Intégration | Source | Fournit |
|---|---|---|
| **ESPHome** | officielle | les clims (`climate.clim_*`), voir le dossier `esphome/` |
| **Tado** | officielle | une entité `climate.*` par pièce, avec un préréglage `auto` / `home` / `away` |
| **RTE Tempo** | HACS ([hekmon/rtetempo](https://github.com/hekmon/rtetempo)) | `sensor.rte_tempo_couleur_actuelle`, `sensor.rte_tempo_prochaine_couleur`, `binary_sensor.rte_tempo_heures_creuses` |
| **Application compagnon** | officielle | `person.*` (présence des téléphones) |
| Compteur électrique (Shelly EM, Linky…) | au choix | `sensor.reseau_puissance` en W, **négatif en injection** (surplus solaire) |
| Tado « mode maison » | capteur de l'intégration Tado | `sensor.maison_mode_tado` (HOME / AWAY) |

Pour le tableau de bord seulement : Mushroom, mini-graph-card et card-mod (HACS), ainsi que l'intégration **Tarif EDF** pour la pastille de prix.

## 2. Entités fournies par les packages

| Entité | Fichier | Rôle pour le superviseur |
|---|---|---|
| `input_boolean.superviseur_chauffage` | chauffage_superviseur.yaml | interrupteur général |
| `input_select.saison_clim` | chauffage_superviseur.yaml | Hiver / Été / Arrêt |
| `input_number.chauffage_consigne_*`, `clim_consigne_ete_*`, `chauffage_prechauffage_veille_rouge`, `chauffage_anticipation_reveil` | chauffage_superviseur.yaml | consignes |
| `input_datetime.chauffage_*` | chauffage_superviseur.yaml | horaires (réveil semaine et week-end, nuit, journée des chambres) |
| `sensor.chauffage_source_active` | chauffage_superviseur.yaml | **qui chauffe** : clims ou gaz, selon la saison et Tempo |
| `sensor.chauffage_consigne_cible` | chauffage_superviseur.yaml | **consigne commune** |
| `sensor.chauffage_consigne_chambres` | chauffage_superviseur.yaml | consigne des chambres (bi-splits) |
| `timer.derogation_clim_*` | chauffage_superviseur.yaml | une dérogation de 3 h par clim |
| `automation` dérogation | chauffage_superviseur.yaml | démarre la dérogation quand un utilisateur règle une clim |
| `input_boolean.surplus_solaire` + `automation` | chauffage_superviseur.yaml | −1 °C l'été en cas de surplus |
| `script.reprendre_pilotage_clims` | chauffage_superviseur.yaml | « Tout en auto » |
| `binary_sensor.presence_maison` | presence.yaml | présent ou absent (consignes jour / nuit / absence) |
| `binary_sensor.presence_telephones`, `input_boolean.presence_calculee`, `input_select.mode_presence` | presence.yaml | fusion Tado + téléphones, forçage |
| `sensor.clim_*_conso_jour` | clims_conso.yaml | consommation du jour (tableau de bord uniquement) |

## 3. La table des pièces

Le superviseur ne découvre rien tout seul. La liste `pieces`, en tête de ses actions, associe à chaque pièce :

```yaml
- { clim: climate.clim_salon, tado: climate.salon, timer: timer.derogation_clim_salon, type: mono }
```

- `clim` : l'entité climate ESPHome de la clim ;
- `tado` : la vanne ou le thermostat Tado **de la même pièce** ;
- `timer` : obligatoirement `timer.derogation_<object_id de la clim>`, car l'automatisation de dérogation construit ce nom ;
- `type` : `mono` ou `bi`. Les `bi` reçoivent la consigne chambres l'hiver et le mode Froid l'été (groupe extérieur partagé).

## Graphe des dépendances

```
 RTE Tempo ──┐
 saison ─────┼─► sensor.chauffage_source_active ─────────────┐
             │                                               │
 Tado HOME ──┐                                               ▼
 téléphones ─┼─► presence_calculee ─► presence_maison ─► sensor.chauffage_consigne_cible ─► SUPERVISEUR ─► clims + Tado
 forçage ────┘                            │                  │                                ▲
                                          └─► sensor.chauffage_consigne_chambres ─────────────┤
 consignes, horaires, surplus solaire ───────────────────────┘                                │
 réglage manuel ─► dérogation (timer 3 h) ────────────────────────────────────────────────────┘
```
