# Bandeau LED piloté par ESP32

*[English version](README.md)*

## Aperçu

Ce projet remplace le boîtier de contrôle propriétaire d'un bandeau LED RGB
12V par un ESP32 exécutant [ESPHome](https://esphome.io/), intégré à Home
Assistant. C'est avant tout un projet didactique : chaque étape ci-dessous
privilégie la compréhension du *pourquoi* plutôt que le simple fait que ça
marche, et ce document est pensé pour évoluer au fil du montage plutôt que
d'être rédigé après coup.

**État actuel :** matériel reçu, rien de câblé pour l'instant. On avance
dans les étapes ci-dessous dans l'ordre.

**Décisions prises jusqu'ici :**
- WiFi + ESPHome plutôt que Zigbee natif, pour la simplicité de mise en
  œuvre et le volume de documentation disponible. Une version Zigbee native
  (ESP32-C6/H2) reste une piste d'évolution future, voir
  [Pistes de suite](#pistes-de-suite).
- Le circuit de commutation MOSFET est câblé à la main à partir de
  composants nus (pas un module tout-monté), justement pour comprendre et
  construire le circuit soi-même plutôt que de le traiter comme une boîte
  noire.
- La configuration ESPHome est gérée en local via la CLI `esphome` et
  versionnée dans ce dépôt, plutôt que via l'add-on Home Assistant.

## Sommaire

1. [Matériel](#matériel)
2. [Outillage](#outillage)
3. [Mesures & constats sur le bandeau](#mesures--constats-sur-le-bandeau)
4. [Circuit de commutation MOSFET](#circuit-de-commutation-mosfet)
5. [Configuration ESPHome](#configuration-esphome)
6. [Flash & premier démarrage](#flash--premier-démarrage)
7. [Intégration Home Assistant](#intégration-home-assistant)
8. [Montage définitif](#montage-définitif)
9. [Pistes de suite](#pistes-de-suite)

## Matériel

Le bandeau LED est un bandeau RGB non adressable à anode commune, piloté à
l'origine par un petit boîtier de contrôle propriétaire (couleur +
intensité).

Liste du matériel pour le remplacement par ESP32 :

| Élément | Notes |
|---|---|
| **ESP32-C6-DevKitC-1** (boîte : `ESP32-C6-DEV-KIT-N8-M`, module : `ESP32-C6-WROOM-1`) | RISC-V, 8MB de flash. Ce n'est pas la WROOM-32 classique prévue par le plan initial — voir note ci-dessous. |
| 3x MOSFET N-channel logic-level, **IRLB8721** | Logic-level, sature correctement depuis un GPIO en 3.3V |
| 3x résistance 220Ω (kit d'assortiment, 10Ω–1MΩ) | Résistance de grille, limite le courant d'appel qui charge la Gate |
| 3x résistance 10kΩ (kit d'assortiment, 10Ω–1MΩ) | Pull-down de grille, maintient le MOSFET bloqué par défaut pendant le boot/reset de l'ESP32 |
| Convertisseur buck 12V→5V, **MP1584EN** | Pour alimenter l'ESP32 depuis l'alimentation 12V du bandeau dans le montage définitif |
| Adaptateur jack DC (femelle, 5,5×2,1mm) → bornier à vis | Pour dériver le +12V/GND de l'alimentation existante sans couper son câble d'origine |

*(Reste à vérifier : que la sortie du module MP1584EN est bien réglée sur
5V avant de brancher l'ESP32 — voir note ci-dessous.)*

> **Note — réglage du convertisseur buck :** le module MP1584EN est
> ajustable via un petit potentiomètre, pas à sortie fixe. Avant de le
> câbler à l'ESP32, régler la sortie sur 5V et vérifier au multimètre à
> vide — brancher une sortie non réglée pourrait griller la carte.

> **Note — changement de carte :** le matériel reçu est un ESP32-C6, pas
> l'ESP32-WROOM-32 générique supposé dans la
> [configuration ESPHome](#configuration-esphome) actuelle. Le C6 est en
> RISC-V et non Xtensa, donc `board: esp32dev` et le choix des pins
> GPIO16/17/18 devront changer une fois arrivés à cette étape — pas traité
> maintenant, juste noté ici. Avantage : le C6 intègre nativement le
> 802.15.4 (Zigbee/Thread) en plus du WiFi, donc la piste
> [Zigbee natif](#pistes-de-suite) ne nécessite plus de matériel séparé.

## Outillage

Outils déjà disponibles, utiles dès cette première phase de prototypage sur
breadboard :

| Outil | Usage |
|---|---|
| Breadboard | Prototypage sans soudure du circuit de commutation MOSFET |
| Fils Dupont | Connexions breadboard-vers-breadboard et breadboard-vers-ESP32 |
| Kit de tournevis de précision | Réglage du potentiomètre du convertisseur MP1584EN, travail sur petits composants en général |
| Alimentation de labo modulable (**MEGO**, EIM Technology), se branche directement sur les rails de la breadboard | Tester le circuit MOSFET sur banc à une tension connue et contrôlable avant de brancher l'alimentation 12V réelle du bandeau |
| Multimètre | Vérifications continuité/tension — mesures du bandeau, comportement Gate/Drain-Source des MOSFET, vérification de la sortie du convertisseur buck |

## Mesures & constats sur le bandeau

Mesures effectuées le 2026-08-09, boîtier de contrôle d'origine encore en
place :

- Tension d'alimentation : **12V DC**.
- Un fil reste à une tension constante de 12V quel que soit le réglage
  couleur/intensité → c'est le **V+** commun (anode commune).
- Les trois autres fils (R/G/B) chutent à ~0,3-0,5V en pleine
  luminosité/blanc plein, et remontent progressivement (jusqu'à ~10V)
  quand la luminosité d'un canal est baissée → chaque canal est commuté
  **côté masse** par PWM.
- Isoler une couleur (ex. rouge seul) fait chuter uniquement le fil
  correspondant pendant que les deux autres remontent → confirme le
  mapping fil ↔ canal et l'indépendance des trois canaux.

**Conclusion :** bandeau RGB à anode commune, commutation PWM côté masse
par canal. C'est exactement ce que le circuit MOSFET ci-dessous doit
reproduire.

## Circuit de commutation MOSFET

*(Pas encore câblé — cette section sera complétée pendant le prototypage
sur breadboard.)*

Circuit prévu, à répéter à l'identique pour chacun des 3 canaux (R, G, B) :

```
ESP32 GPIO --[R 220Ω]--> Gate ----+
                                   |
                          [R 10kΩ vers GND]  (pull-down)
                                   |
                        Drain -- fil R/G/B du bandeau
                        Source -- masse commune
```

- **Gate** : pilotée depuis le GPIO via la résistance ~220Ω. Le pull-down
  10kΩ maintient le MOSFET bloqué tant que le GPIO n'impose pas
  explicitement 3.3V — sans lui, l'état flottant du GPIO au boot/reset de
  l'ESP32 pourrait activer un canal de façon aléatoire.
- **Drain** : relié directement au fil R, G ou B correspondant du bandeau.
- **Source** : relié à la masse commune.
- Le **V+** du bandeau se branche directement sur l'alimentation 12V (en
  court-circuitant complètement le boîtier d'origine).
- **Masse commune obligatoire** entre l'ESP32, les 3 MOSFET et
  l'alimentation du bandeau — sans elle, le "3.3V" d'un GPIO n'a pas de
  sens défini du point de vue du MOSFET.

Avant de brancher le bandeau : tester chaque MOSFET individuellement au
multimètre (mode continuité), en vérifiant que la tension de Gate active
bien la continuité Drain-Source, et l'annule en son absence.

## Configuration ESPHome

Voir [`bandeau-salon.yaml`](bandeau-salon.yaml).

- `output: platform: ledc` — une sortie PWM par canal couleur.
- `light: platform: rgb` — combine les trois sorties en une seule entité
  lumière.
- Les secrets (identifiants WiFi, mot de passe OTA, clé de chiffrement
  API) sont dans `secrets.yaml`, ignoré par git ; `secrets.yaml.example`
  est le gabarit versionné.

Pins GPIO actuellement fixées à 16/17/18 — **à confirmer** contre le
pinout précis de la DevKitC une fois le câblage commencé (éviter les
strapping pins GPIO0/2/12/15 et les pins input-only 34-39).

## Flash & premier démarrage

*(Pas encore fait.)*

- Premier flash via USB : `esphome run bandeau-salon.yaml`.
- Vérifier dans les logs la connexion WiFi.
- Tester manuellement chaque canal (rouge seul / vert seul / bleu seul /
  blanc).
- Les flashs suivants pourront se faire en OTA une fois le device sur le
  réseau.

## Intégration Home Assistant

*(Pas encore fait.)*

- L'intégration native **ESPHome** devrait découvrir automatiquement le
  device via mDNS (Paramètres → Appareils & Services).
- Vérifier le contrôle couleur/luminosité depuis le tableau de bord et
  l'absence de scintillement (peut nécessiter d'ajuster la fréquence PWM
  du `ledc`).

## Montage définitif

*(À venir — une fois le prototype breadboard validé.)*

- Ressouder le circuit MOSFET sur perfboard.
- Isoler les jonctions 12V/GND à la gaine thermorétractable.
- Connectique externe démontable via bornier à vis.
- Boîtier en plastique (pas métallique — atténuerait l'antenne PCB de
  l'ESP32).
- Ajouter un fusible/polyfuse sur le +12V et un condensateur de découplage
  près de l'alimentation de l'ESP32.

## Pistes de suite

- **Zigbee natif** : ESP32-C6 (WiFi + BLE + 802.15.4) ou ESP32-H2, pour
  rejoindre le maillage Zigbee existant via le coordinateur Nabu Casa
  ZBT-2, en s'appuyant sur l'exemple officiel Espressif
  `HA_color_dimmable_light` (ESP-IDF, hors ESPHome). À explorer seulement
  une fois la version WiFi validée et fonctionnelle.
