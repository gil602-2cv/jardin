# Câblage — armoire-semis (Waveshare ESP32-S3-Touch-LCD-7)

## Présentation du board

La Waveshare ESP32-S3-Touch-LCD-7 est une carte tout-en-un :
- ESP32-S3 (dual-core 240 MHz, 8 MB PSRAM)
- Écran MIPI RGB 800×480 intégré
- Touchscreen GT911 intégré
- IO expander CH422G (I2C) pour les signaux de contrôle
- Connecteur FPC pour l'écran

**Tout est intégré — aucun câblage externe nécessaire pour l'écran.**

---

## Bus I2C interne

| Signal | GPIO |
|--------|------|
| SDA | GPIO8 |
| SCL | GPIO9 |
| Fréquence | 50 kHz |

Périphériques sur le bus :
- CH422G IO expander → adresse `0x24` (scan I2C confirmé)
- GT911 touchscreen → adresse `0x5D` (scan I2C confirmé)

---

## CH422G — Attribution des pins

| Pin CH422G | Fonction | Sens |
|-----------|----------|------|
| Pin 1 | Reset touchscreen GT911 | Sortie |
| Pin 2 | **Backlight ON/OFF** | Sortie |
| Pin 3 | Reset display MIPI | Sortie |

> Le CH422G est un expander open-drain. **Le PWM est impossible** sur ce chip.
> Le backlight est donc uniquement ON/OFF (pas de réglage de luminosité).
> La variante **Waveshare 7B** embarque un CH32V003 dédié qui supporte le PWM.

---

## Display MIPI RGB — Pins ESP32-S3

| Signal | GPIO |
|--------|------|
| PCLK | GPIO7 |
| DE | GPIO5 |
| HSYNC | GPIO46 |
| VSYNC | GPIO3 |
| Blue[0:4] | GPIO14, 38, 18, 17, 10 |
| Green[0:5] | GPIO39, 0, 45, 48, 47, 21 |
| Red[0:4] | GPIO1, 2, 42, 41, 40 |

---

## Capteur BLE ATC MiThermometer

| Paramètre | Valeur |
|-----------|--------|
| MAC | `A4:C1:38:C9:4F:A6` |
| Protocole | BLE advertisement (format ATC custom) |
| Données | Température, humidité, batterie |
| Portée | ~10 m (mur béton) |

Firmware recommandé : [ATC_MiThermometer](https://github.com/pvvx/ATC_MiThermometer) — mode "Custom" ou "ATC"

---

## Alimentation

| Source | Tension | Usage |
|--------|---------|-------|
| USB-C | 5V | Flash initial, debug |
| Connecteur 5V | 5V / 2A min | Utilisation normale |

> L'écran 7" consomme environ 800 mA. Prévoir une alimentation 5V/2A minimum.
