# Plan — ESP32-CAM Birdcam → Home Assistant

Objectif : surveiller un nichoir avec une caméra ESP32-CAM, détecter la présence
d'oiseaux et afficher/alerter dans Home Assistant.

## 1. Choix d'architecture

Deux approches possibles, à choisir selon le niveau de contrôle souhaité :

**A. ESPHome (recommandé pour démarrer)**
- Intégration native avec Home Assistant (découverte auto, pas de code à écrire côté HA).
- Composant `esp32_camera` : stream MJPEG + snapshot exposés direct dans HA.
- Pas de détection IA embarquée (l'ESP32-CAM n'a pas la puissance pour ça) →
  la détection "oiseau présent" se fait soit par :
  - capteur de mouvement (PIR) câblé sur une GPIO, exposé comme `binary_sensor`,
  - ou détection IA côté serveur (voir §3) qui pousse un état vers HA.

**B. Arduino/ESP-IDF custom + MQTT**
- Plus de contrôle (flash LED IR, deep sleep, buffer vidéo custom), plus de code à maintenir.
- Publie des snapshots/état via MQTT, HA les récupère via l'intégration MQTT.

→ Recommandation : **ESPHome** pour le flux vidéo + un **PIR** pour déclencher,
et **Frigate** (ou un script de détection) côté serveur pour confirmer "c'est un oiseau"
avant de notifier.

## 2. Matériel

- ESP32-CAM (module AI-Thinker typique).
- Alimentation stable 5V (le Wi-Fi + caméra tirent des pics de courant — alim USB
  de qualité ou régulateur dédié, pas juste le port 5V d'un Arduino).
- Capteur PIR (HC-SR501 ou AM312) câblé sur une GPIO libre (ex: GPIO13) pour
  déclencher la capture sans attendre un polling vidéo permanent.
- Optionnel : LED IR (vision nocturne, nichoir sombre) pilotée par une GPIO/relais,
  avec un filtre IR-cut retiré ou caméra "no-IR" selon le module.
- Boîtier étanche/isolé pour le nichoir (humidité).

## 3. Détection "présence d'oiseau"

L'ESP32 seul ne peut pas faire de la vraie détection d'objet fiable. Options :

- **Frigate** (NVR avec détection IA, tourne sur ton serveur/NAS/RPi) :
  - consomme le flux MJPEG/RTSP de l'ESP32-CAM comme une caméra classique,
  - modèle de détection généraliste (COCO) ne connaît pas "bird" nativement sur
    tous les modèles — vérifier que le label `bird` est supporté par le modèle
    utilisé (EdgeTPU/coral avec SSD mobilenet le supporte).
  - s'intègre nativement à Home Assistant (capteurs `binary_sensor.frigate_*`,
    snapshots, clips).
- Alternative plus légère : script Python (OpenCV + modèle léger type
  MobileNet-SSD ou YOLO-nano) qui pull les snapshots de l'ESP32 à intervalle
  régulier ou sur trigger PIR, et publie le résultat sur MQTT.

→ Recommandation : commencer avec juste le **PIR** comme trigger "présence"
(suffisant pour un nichoir, peu de faux positifs), ajouter Frigate plus tard
si tu veux distinguer "oiseau" de "autre mouvement".

## 4. Étapes de mise en œuvre

1. **Flasher ESPHome sur l'ESP32-CAM**
   - Installer l'add-on ESPHome dans HA (ou CLI `esphome`).
   - Créer `birdcam.yaml` avec :
     - `esp32_camera:` (résolution, framerate, qualité JPEG adaptés à la bande passante Wi-Fi),
     - `binary_sensor:` sur la GPIO du PIR,
     - `api:` + `ota:` pour mises à jour à distance,
     - `wifi:` avec IP fixe recommandée (réseau nichoir souvent en bord de portée).
   - Flasher via USB la première fois (FTDI/USB-TTL, mode flash GPIO0 à GND).

2. **Intégrer dans Home Assistant**
   - L'entité `esphome` se découvre automatiquement si HA et l'ESP sont sur le
     même réseau (mDNS). Ajouter l'intégration ESPHome depuis l'UI.
   - Vérifier : `camera.birdcam` (stream live) + `binary_sensor.birdcam_pir`.

3. **Automatisation présence**
   - Trigger `binary_sensor.birdcam_pir` → `on` :
     - notification mobile avec snapshot (`camera.snapshot` + `notify.mobile_app_...`),
     - enregistrement clip si tu utilises aussi un NVR (Frigate/go2rtc),
     - toggle d'un `input_boolean.nichoir_occupe` avec timeout d'auto-reset.

4. **(Option) Ajouter Frigate pour confirmer "oiseau"**
   - Ajouter la cam comme source RTSP/MJPEG dans Frigate.
   - Configurer le label `bird` dans les objets détectés.
   - Lier `binary_sensor.frigate_birdcam_bird` à l'automatisation au lieu du PIR seul
     (ou en ET logique PIR + détection IA pour réduire les faux positifs).

5. **Fiabilité / autonomie**
   - Si alimentation par batterie envisagée : deep sleep entre détections PIR
     (ESPHome supporte `deep_sleep:` avec réveil sur GPIO) — mais attention,
     le flux caméra continu n'est pas compatible avec du deep sleep agressif.
   - Si alimentation secteur : pas de contrainte, flux continu possible.

## 5. Prochaines actions concrètes

- [ ] Décider : ESPHome + PIR seul, ou ESPHome + Frigate direct.
- [ ] Récupérer le pinout exact du module ESP32-CAM utilisé (AI-Thinker vs autre).
- [ ] Écrire le premier `birdcam.yaml` ESPHome.
- [ ] Test de portée Wi-Fi depuis l'emplacement du nichoir.
- [ ] Choisir l'alimentation (secteur vs batterie/solaire).

---

Dis-moi quel module ESP32-CAM tu as exactement, et si tu as déjà Home Assistant
en local/add-on ESPHome installé — je peux écrire directement le `birdcam.yaml`
de départ.
