# 🌱 ESPHome — Armoire Semis Pro & Arrosage Jardin

Configuration ESPHome complète pour un système de surveillance et d'arrosage intelligent basé sur **Home Assistant**, comprenant un écran tactile 7" et un contrôleur d'arrosage 4 zones.

**Dernière mise à jour : 12/03/2026**

---

## 📦 Devices

| Fichier | Device | Rôle |
|---------|--------|------|
| [`armoire-semis.yaml`](armoire-semis.yaml) | Waveshare ESP32-S3-Touch-LCD-7 | Écran tactile 800×480 — surveillance serre, semis, arrosage |
| [`arrosage-jardin.yaml`](arrosage-jardin.yaml) | ESP32-WROOM-32 (à choisir) | Contrôleur 4 vannes 12V DC + pompe + débitmètre |

---

## 🖥️ armoire-semis — Écran tactile 7"

### Matériel

- **Waveshare ESP32-S3-Touch-LCD-7** (800×480, ESP32-S3, 8 MB PSRAM)
- IO expander **CH422G** (I2C) — backlight, reset display, reset touch
- Écran **MIPI RGB** 16 bits, 16 MHz pixel clock
- Touchscreen **GT911** (I2C, interrupt GPIO4)
- Capteur BLE **ATC MiThermometer** (`A4:C1:38:C9:4F:A6`)

### Fonctionnalités

```
┌─────────┬──────────────────────────────────────────────────────────┐
│ Sidebar │ Zone principale 720 px                                   │
│  80 px  │                                                          │
│         ├──────────────────────────────────────────────────────────┤
│ [Serre] │ Onglet 0 — SERRE                                        │
│ [Semis] │   · Horloge HH:MM (SNTP, fuseau Europe/Paris)           │
│ [Eau]   │   · Saint du jour (API nameday.abalin.net)              │
│         │   · Température BLE : --.-°C                            │
│ [WiFi]  │   · Humidité BLE    : ---%                              │
│         ├──────────────────────────────────────────────────────────┤
│         │ Onglet 1 — SEMIS                                        │
│         │   · Zone NFC / infos plantes (placeholder)              │
│         ├──────────────────────────────────────────────────────────┤
│         │ Onglet 2 — ARROSAGE                                     │
│         │   · 4 cartes zones (Potager / Massifs / Gazon / Serre)  │
│         │   · Durée Smart Irrigation calculée (L/min)             │
│         │   · Indicateur actif/inactif temps réel                 │
│         │   · Boutons ON / OFF / Reset bucket                     │
└─────────┴──────────────────────────────────────────────────────────┘
```

### Pins & bus

| Signal | GPIO / Bus |
|--------|-----------|
| I2C SDA | GPIO8 |
| I2C SCL | GPIO9 |
| Display reset | CH422G pin 3 |
| Touch reset | CH422G pin 1 |
| Backlight ON/OFF | CH422G pin 2 |
| Touch interrupt | GPIO4 |

> **Note backlight** : le rétroéclairage est câblé sur le CH422G (open-drain I2C). Le PWM est impossible sur ce chip — contrôle uniquement ON/OFF. La variante **7B** avec CH32V003 supporte la luminosité variable.

### Polices utilisées (Google Fonts)

| ID | Police | Taille | Usage |
|----|--------|--------|-------|
| `font_clock` | Bebas Neue | 72 px | Valeurs température/humidité |
| `font_text` | Montserrat | 24 px | Titres et labels |
| `font_small` | Montserrat | 16 px | Sous-titres, statuts |
| `font_icon` | Material Symbols Outlined | 36 px | Icônes navigation |

### Entités Home Assistant lues

| Entity ID | Source | Affichage |
|-----------|--------|-----------|
| `sensor.smart_irrigation_potager` | Smart Irrigation | Durée Zone 1 |
| `sensor.smart_irrigation_massifs` | Smart Irrigation | Durée Zone 2 |
| `sensor.smart_irrigation_gazon` | Smart Irrigation | Durée Zone 3 |
| `sensor.smart_irrigation_serre_ext` | Smart Irrigation | Durée Zone 4 |
| `switch.arrosage_zone1` à `zone4` | arrosage-jardin | État vanne (indicateur coloré) |

### Services HA appelés depuis l'écran

| Service | Déclencheur |
|---------|-------------|
| `switch.turn_on/off` | Boutons ON/OFF de chaque zone |
| `smart_irrigation.reset_bucket` | Bouton Reset de chaque zone |

---

## 💧 arrosage-jardin — Contrôleur 4 zones

### Matériel recommandé

- **ESP32-WROOM-32** (NodeMCU-32S ou AZ-Delivery ESP32)
- Module **4 relais** (actif LOW, standard)
- 1 relais supplémentaire pour la pompe
- Débitmètre **YF-S201** (pulse, 7,5 pulses/sec = 1 L/min)
- 4 vannes **12V DC** (électrovannes normalement fermées)
- 1 pompe **12V DC**

### Câblage

```
ESP32 GPIO26 ──► Relais Zone 1 ──► Vanne Potager   (12V DC)
ESP32 GPIO27 ──► Relais Zone 2 ──► Vanne Massifs   (12V DC)
ESP32 GPIO14 ──► Relais Zone 3 ──► Vanne Gazon     (12V DC)
ESP32 GPIO12 ──► Relais Zone 4 ──► Vanne Serre ext.(12V DC)
ESP32 GPIO13 ──► Relais Pompe  ──► Pompe            (12V DC)
ESP32 GPIO34 ◄── YF-S201 signal (pulse, pullup interne)
ESP32 GPIO35 ◄── [RÉSERVÉ] Sonde humidité sol (ADC)
```

> Les relais actif LOW sont les modules 4/8 relais chinois standards (niveau bas = bobine activée). Si vos relais sont actif HIGH, passer `inverted: false` dans chaque switch.

### Logique de sécurité

- **Séquentiel** : une seule zone active à la fois (les autres sont éteintes au ON)
- **Pompe automatique** : s'allume avec la zone, s'éteint quand toutes les zones sont OFF
- **Timeout** : coupure automatique après 60 min (configurable via `timeout_zone`)
- **Reboot-safe** : `restore_mode: ALWAYS_OFF` — tout éteint au démarrage

### Entités exposées dans Home Assistant

| Entity ID | Type | Description |
|-----------|------|-------------|
| `switch.arrosage_zone1` | switch | Vanne Potager |
| `switch.arrosage_zone2` | switch | Vanne Massifs |
| `switch.arrosage_zone3` | switch | Vanne Gazon |
| `switch.arrosage_zone4` | switch | Vanne Serre ext. |
| `switch.arrosage_pompe` | switch | Pompe principale |
| `sensor.arrosage_debit` | sensor | Débit en L/min |
| `sensor.arrosage_volume_session` | sensor | Volume arrosage actuel (L) |
| `sensor.arrosage_volume_total` | sensor | Volume cumulé total (L) — persistant |
| `binary_sensor.arrosage_en_cours` | binary_sensor | True si une zone est active |

### Calibration débitmètre

Le facteur de conversion est défini dans `substitutions` :

```yaml
debit_facteur: "0.1333"  # YF-S201 : 1/7.5 pulses → L/min
```

Modèles courants :

| Modèle | Facteur |
|--------|---------|
| YF-S201 (défaut) | `0.1333` |
| YF-B1 | `0.0600` |
| YF-S401 | `0.0530` |

---

## ⚙️ Installation

### 1. Prérequis

- [ESPHome](https://esphome.io) ≥ 2026.2.4
- Home Assistant avec l'intégration **Smart Irrigation** (HASmartIrrigation ≥ v2025.10.0)
- Add-on ESPHome Dashboard dans HA (ou CLI)

### 2. Secrets

Créer / compléter `secrets.yaml` à la racine de votre config ESPHome :

```yaml
# Réseau
wifi_ssid: "VotreSSID"
wifi_password: "VotreMotDePasse"
ap_password: "fallback_password"

# armoire-semis
# (clé déjà dans le YAML — à changer si vous régénérez)

# arrosage-jardin
arrosage_api_key: "VOTRE_CLE_32_BYTES_BASE64"
arrosage_ota_password: "votre_mot_de_passe_ota"
```

Pour générer une clé API ESPHome :
```bash
python3 -c "import secrets, base64; print(base64.b64encode(secrets.token_bytes(32)).decode())"
```

### 3. Flasher armoire-semis

```bash
esphome run armoire-semis.yaml
```

Premier flash obligatoirement en **USB** (port UART de la Waveshare).  
Les mises à jour suivantes se font en **OTA** via Wi-Fi.

### 4. Flasher arrosage-jardin

```bash
esphome run arrosage-jardin.yaml
```

### 5. Créer les zones Smart Irrigation dans HA

Paramètres → Intégrations → Smart Irrigation → **Ajouter une zone** :

| Nom de zone | Entity ID généré |
|-------------|-----------------|
| `Potager` | `sensor.smart_irrigation_potager` |
| `Massifs` | `sensor.smart_irrigation_massifs` |
| `Gazon` | `sensor.smart_irrigation_gazon` |
| `Serre Ext` | `sensor.smart_irrigation_serre_ext` |

> Si vous choisissez d'autres noms, adapter les `entity_id` dans les blocs `sensor:` de `armoire-semis.yaml`.

---

## 🔄 Intégration Smart Irrigation — Principe

```
Smart Irrigation (HA)
  │  Calcule chaque soir :
  │  durée = f(évapotranspiration, pluie, prévisions Pirate Weather)
  │  → sensor.smart_irrigation_[zone]  (valeur en secondes)
  │
  ▼
armoire-semis (écran)
  │  Lit le sensor, convertit en minutes
  │  Affiche : "23 min" ou "Pas d'arrosage"
  │  Bouton Reset → smart_irrigation.reset_bucket
  │
  ▼
arrosage-jardin (ESP32)
  │  switch.arrosage_zoneX → vanne physique 12V
  │  Pompe auto, timeout sécurité, comptage volume
  │
  ▼
Automation HA (à créer)
  │  Déclencheur : heure programmée (ex: 06:00)
  │  Condition : sensor.smart_irrigation_[zone] > 0
  │  Action : switch.turn_on arrosage_zone + delay + switch.turn_off
  │           + smart_irrigation.reset_bucket
```

---

## 📁 Structure du dépôt

```
.
├── README.md
├── armoire-semis.yaml          # Device écran tactile 7"
├── arrosage-jardin.yaml        # Device contrôleur arrosage
├── secrets.yaml.template       # Template secrets (ne pas committer secrets.yaml !)
├── .gitignore
└── docs/
    ├── WIRING_ARMOIRE.md       # Schéma câblage écran
    ├── WIRING_ARROSAGE.md      # Schéma câblage arrosage
    └── SMART_IRRIGATION.md     # Guide configuration Smart Irrigation
```

---

## 🗺️ Roadmap

- [ ] Intégration NFC (onglet Semis — identification plantes)
- [ ] Sonde humidité sol (GPIO35 — déjà réservé)
- [ ] Automatisation HA arrosage intelligent (déclenchement sur Smart Irrigation)
- [ ] Page statistiques : volume consommé par zone / par semaine
- [ ] Luminosité écran réglable (nécessite variante Waveshare 7B)

---

## 🐛 Historique des corrections ESPHome

| Version | Correction |
|---------|-----------|
| Run 40 | Syntaxe `line.points` : liste YAML `{x:, y:}` |
| Run 41 | Widget `btn` inexistant → `obj` + `clickable: true` |
| Run 42 | `tab_index` → `index` dans `lvgl.tabview.select` |
| Run 43 | `lvgl.widget.add_style` → `lvgl.widget.update` |
| Run 44 | `tab_bar_size` supprimé |
| Run 45 | Backlight GPIO → CH422G pin 2 (conflit `mipi_rgb`) |
| Run 47 | Glyph `U+43` (lettre C) ajouté dans `font_clock` |

---

## 📄 Licence

MIT — libre d'utilisation, modification et redistribution.
