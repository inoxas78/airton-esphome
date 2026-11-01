# Dépannage / FAQ

## Clims

**L'afficheur de la clim indique « 02 » au lieu de 21.**
La consigne est envoyée en °C. La carte l'attend en dixièmes : `set_integer_datapoint_value(2, x * 10)`.

**Pas de mode Auto sur un monosplit.**
`climate: tuya` ne le gère pas. Utilisez le climate `template` de `airton-monosplit.yaml`.

**La température ambiante saute à 0, 0,4 ou 12,7 °C.**
Le DP3 transporte une seconde valeur. Il faut le filtre `if (x < 50 || x > 450) return {};`, suivi de `throttle_average`.

**L'historique de température s'affiche en bandes colorées au lieu d'une courbe.**
Le capteur n'a pas d'unité. Ajoutez `unit_of_measurement: "°C"`, `device_class: temperature` et `state_class: measurement`. Si Home Assistant signale ensuite « l'unité a changé » pour les statistiques, choisissez de mettre à jour l'unité **sans conversion** (Outils de développement › Statistiques), car les valeurs étaient déjà en °C.

**DP12 ou DP15 n'apparaissent jamais dans ESPHome.**
C'est l'octet `0x55` parasite envoyé par la carte, voir [protocole-tuya-monosplit.md](protocole-tuya-monosplit.md). L'écriture de ces DP fonctionne quand même.

**J'ai coupé le hors-gel (DP111) et la clim s'est éteinte.**
C'est normal : il faut relancer le mode voulu ensuite.

**L'écran affiche « LA ».**
Le DP115 est à 0 : remettez-le à 1.

**Un monosplit bipe à chaque ordre.**
Rien à faire par le Wi-Fi : aucun DP ne le coupe. Sur les bi-splits, activez la « commande silencieuse ».

**Une bi-split refuse le mode demandé (défaut « conflit de mode »).**
Les unités d'un même groupe extérieur ne peuvent pas être l'une en chaud, l'autre en froid.

**La puissance moyenne d'une bi-split reste vide.**
Elle apparaît au **deuxième** pas du compteur d'énergie, c'est-à-dire après 0,2 kWh consommés depuis le démarrage de l'ESP.

**Le superviseur ne touche plus à une pièce.**
Un réglage manuel a lancé une dérogation de 3 h. Touchez la carte « Réglage manuel », ou « Tout en auto » pour libérer toutes les pièces.

## Capturer les trames sans Home Assistant

Le `web_server` d'ESPHome expose les journaux. Ouvrez `http://<ip-de-l-esp>/`, puis dans la console du navigateur :

```js
const L = []; const es = new EventSource('/events');
es.addEventListener('log', e => L.push(e.data));
```

## Base de données Home Assistant corrompue

Symptômes : « database disk image is malformed », historique vide, redémarrage en boucle.

1. **Arrêter Home Assistant** (`ha core stop`) avant toute manipulation. Ne jamais réparer pendant qu'il tourne ou pendant une sauvegarde.
2. Récupérer ce qui peut l'être :
   ```sh
   sqlite3 home-assistant_v2.db ".recover --ignore-freelist" | sqlite3 recup.db
   ```
3. Si on repart d'une base neuve, fusionner les statistiques long terme de l'ancienne. Avec `ATTACH`, puis `INSERT` dans `statistics` et `statistics_short_term` en remappant les `metadata_id`. Il faut **recalculer les colonnes `sum`** pour que les compteurs d'énergie restent continus.
4. Remplacer le fichier, puis redémarrer.

Pour éviter que ça se reproduise :

- décaler la sauvegarde automatique et l'envoi vers le cloud (par exemple 1 h 30 et 2 h 30) pour qu'ils ne se chevauchent pas ;
- alléger l'enregistreur ([recorder_exemple.yaml](../homeassistant/recorder_exemple.yaml)) ;
- surveiller l'état du SSD (SMART) et la RAM (memtest86+). Des erreurs de segmentation récurrentes dans le journal du système sont un signal d'alerte matériel.
