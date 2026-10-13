# Écran tactile DomBusEVSE + priorité solaire (Home Assistant)

*[English version](README.md)*

Un petit **écran tactile près de la borne** pour piloter une wallbox
[Creasol DomBusEVSE](https://www.creasol.it/) via Home Assistant, et des paquets Home Assistant
qui permettent à **la voiture et au ballon d'eau chaude de se partager le surplus solaire**
sans se gêner.

<img src="docs/img/screen_photo.jpg" width="420" alt="L'écran CYD dans son boîtier imprimé en 3D">

Utilisé au quotidien avec une DomBusEVSE (kit EVSE1P, monophasé, 6 à 13 A), une Hyundai Ioniq PHEV,
des micro-onduleurs APsystems et un routeur d'eau chaude MSunPV.

## L'écran

| | |
|---|---|
| Carte | ESP32-2432S028 « CYD », version 2 USB : écran 2,8" 320×240 ILI9341, tactile résistif XPT2046, LED RGB, environ 15 € |
| Logiciel | ESPHome + LVGL, Wi-Fi, API native Home Assistant, mises à jour sans fil |
| Principe | **Aucune logique dans l'écran.** Il affiche les états de Home Assistant et, à chaque appui, lui demande de lancer un script. Les mêmes scripts fonctionnent depuis le tableau de bord. |

| Charge solaire | Branchée, attend le surplus | Forçage à 10 A |
|---|---|---|
| ![](docs/img/solar_charging.png) | ![](docs/img/plugged_waiting.png) | ![](docs/img/forced_10a.png) |
| **Heures creuses programmées** | **Débranchée** | **Alarme de la borne** |
| ![](docs/img/offpeak_scheduled.png) | ![](docs/img/unplugged.png) | ![](docs/img/alarm.png) |

**Affichage :** mode avec l'intensité réelle (en couleur), batterie de la voiture, état de la borne
(pictogramme, puissance et part solaire pendant la charge), production solaire, durée et énergie
de la charge en cours, heure.

**4 boutons, à maintenir 2 secondes** (un appui court affiche seulement « MAINTENIR ») :

| Bouton | Action |
|---|---|
| **STOP** | Arrête la charge ; la borne repasse d'elle-même en Solaire quelques minutes plus tard |
| **SOLAIRE** | Recharge uniquement sur le surplus solaire |
| **HC NUIT** | Programme la recharge en heures creuses (un nouvel appui annule) |
| **FORCER 6 A** | Charge immédiate ; chaque appui passe au courant suivant : 6 → 8 → 10 → 12 → 6 A |

La LED reprend la couleur du mode (vert Solaire, bleu HC, orange forçage, rouge arrêt).
L'écran baisse sa luminosité (30 % le jour après 5 min, 5 % la nuit après 1 min) sans jamais
s'éteindre ; le premier toucher le rallume seulement.

## Partager le soleil entre la voiture et le ballon

Un routeur d'eau chaude (ici MSunPV) envoie **tout** le surplus au ballon jusqu'à ce que son
thermostat coupe : une wallbox en mode Solaire ne démarre alors jamais. Et quand les deux régulent
sur la même mesure réseau, ils oscillent (« yoyo » : la charge s'arrête et redémarre toutes les
quelques minutes).

`wallbox_priorite_ballon.yaml` donne la priorité à la voiture dès que l'eau est assez chaude
(seuil `input_number.seuil_temperature_ce`) :

- **Démarrage :** ballon au-dessus du seuil, wallbox en Solaire, voiture branchée et pas pleine,
  production ≥ 1 800 W depuis 5 min → charge à 6 A (y compris si la voiture charge dès le branchement).
- **Régulation lente :** chaque minute, +1 A tant que le ballon reçoit encore plus de 400 W, jusqu'à 12 A.
- **Baisse rapide :** si la maison achète au réseau, le courant baisse d'un coup d'autant (en 15 s, jamais sous 6 A), puis attend 5 min avant de remonter.
- **Fin :** batterie pleine, achats au réseau prolongés, ou ballon 3 °C sous le seuil → retour au mode Solaire normal.

La voiture bouge **lentement**, le routeur **vite** : le routeur donne toujours au ballon exactement ce qui reste.

## Fichiers

```
esphome/ecran-wallbox.yaml          configuration de l'écran (entités dans `substitutions:`)
esphome/secrets.yaml.example        à copier en secrets.yaml et à remplir
homeassistant/packages/
  ecran_wallbox.yaml                scripts appelés par les 4 boutons
  wallbox_amperes.yaml              forçage à 6/8/10/12 A (EVMaxCurrent + mode 100%)
  wallbox_solaire_defaut.yaml       Solaire comme mode de repos (retour automatique après un arrêt)
  wallbox_heures_creuses.yaml       demande de recharge HC, démarrage et fin
  wallbox_priorite_ballon.yaml      priorité voiture sur le ballon, avec régulation
  wallbox_puissance_reseau.yaml     envoie la puissance réseau au module (registre Modbus 11)
```

## Prérequis

- La **configuration Home Assistant de Creasol pour la DomBusEVSE** (Modbus), qui fournit
  `input_select.evmodename`, `input_select.evstatename`, `input_number.evmaxcurrent`,
  `input_button.evmodeset` et les automatisations d'écriture des registres.
- Ces entités, venant de votre installation (à renommer, ou à créer en capteurs template) :

| Entité | Signification |
|---|---|
| `sensor.grid_power` | Puissance réseau en W, **positive quand on achète**, négative quand on injecte |
| `sensor.solar_power` | Production solaire actuelle (W) |
| `sensor.car_battery_level` | Batterie de la voiture (%) |
| `binary_sensor.car_plugged_in` | Voiture branchée |
| `sensor.evse_charging_power` / `sensor.evse_charging_current` | Puissance (W) / courant (A) de charge réels |
| `sensor.evse_solar_share` | Part solaire de la charge (%) |
| `sensor.evse_mode_raw` | Registre brut du mode du module (sert seulement à vérifier qu'il répond) |
| `sensor.water_heater_temperature` / `sensor.water_heater_power` | Température (°C) / puissance (W) du ballon, pour la priorité seulement |

## Installation

1. Copier `homeassistant/packages/*.yaml` dans le dossier `packages` de Home Assistant
   (avec `homeassistant: packages: !include_dir_named packages` dans `configuration.yaml`), puis redémarrer.
2. Dans ESPHome, créer `secrets.yaml` à partir de l'exemple, adapter `substitutions:` et flasher le CYD une première fois en USB ; ensuite, les mises à jour passent par le Wi-Fi.
3. Dans **Paramètres > Appareils et services > ESPHome > l'écran > Configurer**, cocher
   **« Autoriser l'appareil à effectuer des actions Home Assistant »**, sinon les boutons ne font rien.
4. Adapter `hub` / `slave` Modbus dans `wallbox_puissance_reseau.yaml`, et la calibration du tactile si les appuis tombent à côté des boutons.

## Licence

MIT. Retours et améliorations bienvenus.
