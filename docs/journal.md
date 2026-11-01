# Journal du projet

Comment on est passé de 4 clims pilotées par l'application Airton (et des dongles à 99 € pièce) à une installation 100 % locale, intégrée au chauffage de la maison. Fin octobre et début novembre 2026.

## 1. Monosplits : remplacer le dongle USB

- Un ESP8266 (ESP-12E) est soudé à la place du module Wi-Fi, sur la prise USB de l'unité intérieure, et flashé avec ESPHome et le composant `tuya`.
- La carte répond au protocole Tuya MCU standard (produit `pjnywabwywopufvb`). Marche, mode, consigne, ventilation, volets, nuit, purificateur et écran fonctionnent dès le départ.

## 2. Bi-splits : composant ACW02 et compteur d'énergie caché

- Un ESP32 par unité intérieure, avec le composant [devildant/acw02_esphome](https://github.com/devildant/acw02_esphome).
- Les journaux montraient des trames de 18 et 34 octets **ignorées** par le composant. En les comparant dans le temps : les octets 13-14 de la trame de 18 octets augmentent pendant que la clim tourne. C'est un **compteur d'énergie** par pas de 0,1 kWh.
- Un décodeur passif (`uart.debug`) le publie en kWh `total_increasing`, directement utilisable dans le tableau de bord Énergie, et en déduit une puissance moyenne.

## 3. Noms cohérents et vue « Thermostats »

- Renommage de toutes les entités (`climate.clim_salon`, `climate.clim_chambre_…`).
- Une vue unique : en haut le contrôle des clims, en bas le pilotage. Style épuré avec Mushroom et des cartes arrondies.

## 4. Mode Auto absent sur les monosplits

La plateforme `climate: tuya` d'ESPHome ne gère pas le mode Auto (DP4 = 0). Elle est remplacée par un climate `template` piloté par les datapoints, avec un script qui publie l'état renvoyé par la carte.

## 5. Consommation des monosplits : estimation

La carte ne renvoie aucune consommation : les DP101 à 104 valent toujours 1. Un **modèle d'onduleur** les remplace :

- veille : 3 W ;
- ventilation seule : 30 W ;
- compresseur : de 250 W à la consigne jusqu'à 1 200 W à 3 °C d'écart ;
- ventilation seule quand la consigne est dépassée de plus de 1 °C.

Les paramètres sont réglables depuis Home Assistant, et une intégration donne les kWh.

## 6. « 02 » sur l'afficheur

Régler 21 °C depuis Home Assistant affichait « 02 » sur la clim. La carte **attend la consigne en dixièmes** à l'écriture (210) alors qu'elle la renvoie en °C entiers. Correction : ×10 à l'écriture, et une lecture qui accepte les deux formats.

## 7. Orchestrateur chauffage

À partir de l'abonnement Tempo, de la production solaire, de la facture de gaz (qui inclut l'eau chaude sanitaire) et du plan de la maison :

- clims en chauffage sauf **heures pleines des jours rouges** → gaz avec Tado en auto ;
- Tado arrêté quand les clims chauffent ;
- 20 °C le jour quand on est là, 18 °C la nuit, confort dès 7 h en semaine et 8 h le week-end ;
- préchauffage de 1,5 °C en heures creuses la veille d'un jour rouge ;
- dérogation de 3 h en cas de réglage manuel ;
- le mode été et le mode Arrêt sont venus ensuite.

Voir [orchestrateur.md](orchestrateur.md).

## 8. Ménage Tempo

84 `input_number` de prix saisis à la main ont été supprimés et remplacés par les capteurs officiels de l'intégration Tarif EDF (24 références mises à jour).

## 9. Base de données corrompue

Plusieurs corruptions de `home-assistant_v2.db` se sont produites. La base a été récupérée avec `sqlite3 .recover`, puis les statistiques long terme ont été fusionnées en recalculant les cumuls `sum`. On a ensuite :

- décalé les sauvegardes (Home Assistant à 1 h 30, envoi Google Drive à 2 h 30) ;
- allégé l'enregistreur ;
- libéré 18 Go.

Voir [depannage.md](depannage.md).

## 10. Mode été

Monosplits en Auto, bi-splits en Froid : un groupe extérieur partagé ne peut pas chauffer et refroidir en même temps. −1 °C quand il y a du surplus solaire.

## 11. Présence

La géolocalisation Tado réagit parfois avec plusieurs minutes de retard. Elle est fusionnée avec les téléphones : **le dernier changement l'emporte**, à l'arrivée comme au départ. Un sélecteur permet de forcer « présents » ou « absents ».

## 12. Chambres à 19 °C en journée

Consigne spécifique pour les chambres de 9 h à 17 h quand la maison est occupée.

## 13. Température à 0 ou 12,7 °C sur les monosplits

Le DP3 alterne très vite entre la vraie température (×10) et une seconde valeur (4, 14…). Correction :

- filtre 5–45 °C et moyenne sur 30 s ;
- capteur séparé pour la valeur parasite ;
- unité °C ajoutée (l'historique s'affichait en bandes colorées) et arrondi au dixième.

## 14. Capture et analyse des trames

- Monosplit : demande d'état complet (commande `0x08`), décodage de tous les DP, découverte de l'**octet 0x55 parasite** qui fait perdre DP12 et DP15 à ESPHome.
- Bi-split : décodage octet par octet. On trouve le **compteur d'heures** (+24 par jour) et la référence ASCII « DB11370 ». Tous les CRC16 Modbus sont vérifiés.

Voir [protocole-tuya-monosplit.md](protocole-tuya-monosplit.md) et [protocole-7a7a-bisplit.md](protocole-7a7a-bisplit.md).

## 15. La chasse au bip

- Bi-splits : la « commande silencieuse » d'ACW02 fonctionne.
- Monosplits : DP8, 12, 108, 111 et 115 ont été testés un par un. Résultats : DP111 = **hors-gel**, DP115 = « LA » à l'écran, et aucun ne coupe le bip. **Impossible par le Wi-Fi.**

## 16. Température de la télécommande (I-Feel)

Elle n'est transmise au module Wi-Fi sur aucun des deux modèles. Pour réguler sur la température réelle d'une pièce, utiliser un capteur Home Assistant (Tado, ESPHome…).
