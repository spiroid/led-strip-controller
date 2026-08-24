# Bandeau LED piloté par ESP32

*[English version](README.md)*

## Aperçu

Ce projet remplace le boîtier de contrôle propriétaire d'un bandeau LED RGB
12V par un ESP32 exécutant [ESPHome](https://esphome.io/), intégré à Home
Assistant. C'est avant tout un projet didactique : chaque étape ci-dessous
privilégie la compréhension du *pourquoi* plutôt que le simple fait que ça
marche, et ce document est pensé pour évoluer au fil du montage plutôt que
d'être rédigé après coup.

**État actuel :** matériel et outillage inventoriés, config ESPHome adaptée
à la carte reçue, les 3 MOSFET testés et validés individuellement sur banc.
Circuit RGB complet à 3 canaux câblé de bout en bout sur breadboard,
flashé, et validé stable — canaux individuels et blanc plein à 100% de
luminosité. Intégré à Home Assistant, contrôle couleur/luminosité confirmé
sans scintillement depuis le tableau de bord.

**Ordre des manipulations prévu :**
- [x] Mesures du bandeau (anode commune confirmée, voir
      [Mesures & constats sur le bandeau](#mesures--constats-sur-le-bandeau))
- [x] Inventaire matériel & outillage
- [x] Adapter `bandeau-salon.yaml` à l'ESP32-C6 (board id, framework, pins
      GPIO) — config validée, pas encore flashée
- [x] Tester les 3 MOSFET individuellement sur banc (continuité au
      multimètre, indépendamment de l'ESP32) — voir
      [`mosfet-test-schema.html`](mosfet-test-schema.html)
- [x] Câbler et valider un seul canal de bout en bout sur breadboard
      (GPIO → MOSFET → fil du bandeau), en flashant une config minimale à
      une seule sortie — voir [`cablage-canal-rouge.html`](cablage-canal-rouge.html)
      et [Flash & premier démarrage](#flash--premier-démarrage)
- [x] Répliquer aux 2 autres canaux — voir
      [`cablage-canaux-vert-bleu.html`](cablage-canaux-vert-bleu.html)
- [x] Config ESPHome complète à 3 canaux, flash, test par canal — voir
      [Flash & premier démarrage](#flash--premier-démarrage)
- [x] Intégration Home Assistant — voir
      [Intégration Home Assistant](#intégration-home-assistant)
- [x] Tester sur banc la prise jack et le convertisseur buck (MP1584EN)
      sur l'alimentation 12V réelle du bandeau, indépendamment du montage
      définitif — voir [Montage définitif](#montage-définitif)
- [ ] Montage définitif (perfboard, réglage du buck, boîtier) — voir
      [Montage définitif](#montage-définitif)

Note : l'ESP32 reste alimenté en USB pendant tout le prototypage sur
breadboard — le convertisseur buck n'est nécessaire qu'au montage
définitif, son réglage est donc reporté jusque-là. Dès que l'alimentation
12V réelle du bandeau est branchée (à partir de l'étape câblage d'un seul
canal), la masse USB de l'ESP32 et la masse de l'alimentation du bandeau
doivent partager un même rail commun sur la breadboard.

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
> l'ESP32-WROOM-32 générique prévu au départ. Le C6 est en RISC-V et non
> Xtensa ; la [configuration ESPHome](#configuration-esphome) a été mise à
> jour en conséquence (`board: esp32-c6-devkitc-1`, framework `esp-idf`,
> GPIO18/19/20). Avantage : le C6 intègre nativement le 802.15.4
> (Zigbee/Thread) en plus du WiFi, donc la piste
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

- `esp32: board: esp32-c6-devkitc-1`, `framework: type: esp-idf` — colle à
  l'ESP32-C6-DevKitC-1 réellement reçue (RISC-V ; le framework `esp-idf` a
  un support ESP32-C6 plus mature qu'`arduino` dans ESPHome).
- `output: platform: ledc` — une sortie PWM par canal couleur, sur
  **GPIO18/19/20**. Choisies dans le pool de GPIO sans conflit de
  l'ESP32-C6-DevKitC-1 (0,1,2,3,6,7,10,11,18,19,20,21,22,23), en évitant :
  les strapping pins GPIO4/5/8/9/15, l'USB natif GPIO12/13, et la console
  UART GPIO16/17.
- `light: platform: rgb` — combine les trois sorties en une seule entité
  lumière.
- Les secrets (identifiants WiFi, mot de passe OTA, clé de chiffrement
  API) sont dans `secrets.yaml`, ignoré par git ; `secrets.yaml.example`
  est le gabarit versionné.

> **Note — repérage des broches GND :** l'ESP32-C6-DevKitC-1 a 4 broches
> GND (toutes reliées entre elles en interne, n'importe laquelle convient),
> réparties sur les deux connecteurs J1/J3 (pinout confirmé via la doc
> officielle Espressif) :
> - J1, position 15 (dernière, juste après le 5V)
> - J3, position 1 (première, juste avant TX/RX)
> - J3, position 12 (entre GPIO9 et GPIO13)
> - J3, position 15 (dernière, après GPIO12)
>
> Pratique : GPIO18/19/20 sont sur J3, positions 10/9/8 — juste à côté du
> GND de J3 position 12 (2 crans de GPIO18). Les 3 sorties PWM et une
> masse tiennent donc sur 5 broches consécutives du même connecteur.
> Positions confirmées physiquement sur la carte reçue le 2026-08-20.

Config validée avec `esphome config bandeau-salon.yaml` (le fichier est
syntaxiquement correct et se résout sans erreur — ça ne compile ni ne
flashe encore).

## Flash & premier démarrage

Canal Rouge validé le 2026-08-21 avec
[`bandeau-salon-test-rouge.yaml`](bandeau-salon-test-rouge.yaml) — une
variante temporaire à une seule sortie de `bandeau-salon.yaml`
(`light: platform: monochromatic` sur `output_red` uniquement), voir
[`cablage-canal-rouge.html`](cablage-canal-rouge.html) pour le câblage.

- Premier flash via USB : `esphome run bandeau-salon-test-rouge.yaml`. Sur
  l'ESP32-C6-DevKitC-1, la carte s'énumère comme "USB JTAG/serial debug
  unit" Espressif (`idVendor=303a`) — choisir ce périphérique série si
  plusieurs sont proposés. Le câble USB doit porter les lignes data, pas
  juste l'alimentation (un câble de téléphone charge-only allume bien la
  LED de la carte mais ne s'énumère jamais côté PC).
- L'entité lumière a été pilotée depuis Home Assistant (découverte
  automatique via l'intégration ESPHome/mDNS une fois flashé), la CLI
  `esphome` n'ayant pas de commande pour basculer une entité directement.
- Connexion WiFi vérifiée dans les logs, puis montée en luminosité de 0%
  à 100%.
- **Constat :** vers 99-100% de luminosité (appel de courant quasi continu,
  contre un courant moyen plus faible à un rapport cyclique PWM partiel),
  le bandeau coupait — remonté à la limitation de courant de l'alimentation
  de banc MEGO, pas à un défaut de câblage ou de MOSFET (confirmé par les
  logs ESPHome ne montrant aucun reboot/perte WiFi de l'ESP32 au même
  moment). Baisser légèrement la tension réglée sur le MEGO a ramené
  l'appel sous la limite ; stable à 100% depuis. Ne devrait pas se
  reproduire sur l'alimentation 12V réelle du bandeau, dimensionnée pour la
  charge complète à 3 canaux.
- MOSFET resté à peine tiède après plusieurs minutes à 100% — validé.
- Les flashs suivants pourront se faire en OTA une fois le device sur le
  réseau.

RGB complet à 3 canaux validé le 2026-08-23 avec
[`bandeau-salon.yaml`](bandeau-salon.yaml), câblage suivant
[`cablage-canaux-vert-bleu.html`](cablage-canaux-vert-bleu.html) :

- Flashé via USB (nouvelle identité de device par rapport à
  `bandeau-salon-test-rouge`, donc l'USB était de nouveau nécessaire pour
  ce premier flash de `bandeau-salon`).
- **Constat :** piloter le canal bleu allumait le vert à la place — remonté
  à une inversion des fils de Gate entre GPIO19 et GPIO20 sur la
  breadboard, pas à un défaut de MOSFET ou de config. Corrigé en
  rebranchant correctement les Gates ; rouge/vert/bleu retestés
  individuellement ensuite pour confirmer que l'inversion était bien
  résolue dans les deux sens.
- Le blanc plein (les 3 canaux à 100% simultanément) a de nouveau déclenché
  la limitation de courant du MEGO, cette fois à sa tension de consigne
  d'origine — l'appel cumulé des 3 canaux est plus élevé que celui d'un
  seul canal. Baisser le MEGO à **9,5V** a ramené l'appel sous la limite ;
  stable en blanc plein depuis. Même réserve que le constat sur le canal
  Rouge ci-dessus : ne devrait pas se reproduire sur l'alimentation 12V
  réelle du bandeau, dimensionnée pour la charge complète.
- Les 3 MOSFET sont restés froids après une charge blanc plein prolongée —
  validé.

## Intégration Home Assistant

Validé le 2026-08-23. L'intégration native **ESPHome** a découvert
automatiquement le device via mDNS (Paramètres → Appareils & Services) une
fois `bandeau-salon.yaml` flashé et connecté au WiFi ; l'ajout a demandé la
clé de chiffrement API (`api_encryption_key` de `secrets.yaml`). L'entité
lumière "Bandeau LED" est apparue avec un contrôle complet couleur (roue
chromatique) + luminosité, et le mélange de couleurs comme la variation de
luminosité fonctionnent proprement depuis le tableau de bord, sans
scintillement — aucun ajustement de la fréquence PWM du `ledc` n'a été
nécessaire.

## Montage définitif

*(Chaîne d'alimentation testée sur banc ; le reste reste à venir une fois
le prototype breadboard complet validé.)*

Avant de souder quoi que ce soit, la chaîne d'alimentation a été testée
seule sur breadboard, sur le même principe que la validation de chaque
MOSFET isolément avant de le câbler dans le circuit lumière — voir
[`test-buck-converter.html`](test-buck-converter.html) pour le câblage :

- [x] Brancher l'adaptateur jack DC → bornier à vis sur l'alimentation 12V
  réelle du bandeau (en dérivation, sans couper le câble d'origine) —
  ~12V confirmés au multimètre entre + et GND.
- [x] Câbler cette dérivation sur l'entrée du MP1584EN et régler son
  potentiomètre — 5,0V stables confirmés en sortie au multimètre, à vide,
  avant de brancher l'ESP32.
- [x] Alimenter l'ESP32 depuis la sortie du buck (pin `5V` du connecteur
  J1, même rail que le `5V` USB), câble USB débranché pour éviter de
  relier deux sources 5V entre elles — boot et reconnexion WiFi confirmés
  normaux.
- [x] Ajouter les condensateurs de découplage (470µF électrolytique +
  100nF céramique, en parallèle, câblés au plus près des pins `5V`/`GND`
  de l'ESP32) et confirmer l'absence de reboot en usage WiFi normal — voir
  [`decouplage-condensateurs.html`](decouplage-condensateurs.html). Câblés
  et validés : l'ESP32 boot et tient sans reboot malgré une activité WiFi
  répétée.

**Pourquoi le découplage est utile ici :** l'ESP32 tire des pics de
courant brefs à chaque émission WiFi ; sans réserve locale, ces pics
doivent être fournis via le fil venant du buck converter, dont la
résistance provoque une chute de tension momentanée qui peut faire
rebooter la carte en pleine transmission. Les deux condensateurs couvrent
des échelles de temps différentes plutôt que de faire doublon :
l'électrolytique 470µF est le réservoir qui lisse les pics WiFi
(relativement lents, de l'ordre de la centaine de µs) ; le céramique
100nF n'a pas l'inductance parasite de l'électrolytique, donc il continue
à fonctionner aux fréquences plus élevées (bruit de commutation du buck
converter, fronts numériques rapides) là où l'électrolytique cesse d'être
efficace. Ils sont câblés **en parallèle** sur les mêmes rails 5V/GND, pas
en série — en série la capacité totale s'effondrerait vers la plus
petite des deux (perdant l'essentiel du réservoir de 470µF), et aucun des
deux ne formerait le pont basse impédance direct entre les rails que le
découplage exige réellement.

**Calibre du fusible :** mesuré le 2026-08-24 en série sur le +12V, entre
l'alimentation réelle du bandeau et le V+ du bandeau, en blanc plein 100%
(les 3 canaux, le cas le plus gourmand) — **0,83–0,895A**. Cible un
polyfuse PTC avec un courant de maintien (*hold current*) d'environ
**1,5A** (marge d'environ 30-70% au-dessus du max mesuré, assez serré
pour réagir à un vrai court-circuit avant tout dommage aux fils/MOSFET,
assez large pour ne pas déclencher sur une variation normale). Aucun
disponible sous la main pour l'instant — commande en cours d'un polyfuse
traversant (radial), ~1,5A hold current, tension nominale ≥12V. Ne
bloque pas le reste du montage définitif : il se câble en série sur le
+12V et peut être ajouté à n'importe quel moment.

Reste à faire pour le montage permanent — voir
[`cablage-perfboard.html`](cablage-perfboard.html) pour le schéma de
câblage consolidé (les 3 canaux + buck converter + découplage, en un seul
diagramme) et l'ordre de soudure recommandé :

- Ressouder le circuit MOSFET sur perfboard.
- Isoler les jonctions 12V/GND à la gaine thermorétractable.
- Connectique externe démontable via bornier à vis.
- Boîtier en plastique (pas métallique — atténuerait l'antenne PCB de
  l'ESP32).
- Ajouter le polyfuse sur le +12V une fois reçu.

## Pistes de suite

- **Zigbee natif** : ESP32-C6 (WiFi + BLE + 802.15.4) ou ESP32-H2, pour
  rejoindre le maillage Zigbee existant via le coordinateur Nabu Casa
  ZBT-2, en s'appuyant sur l'exemple officiel Espressif
  `HA_color_dimmable_light` (ESP-IDF, hors ESPHome). À explorer seulement
  une fois la version WiFi validée et fonctionnelle.
