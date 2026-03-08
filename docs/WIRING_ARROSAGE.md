# Câblage — arrosage-jardin (ESP32-WROOM-32)

## Matériel requis

| Composant | Référence suggérée | Quantité |
|-----------|-------------------|----------|
| Microcontrôleur | ESP32-WROOM-32 (NodeMCU-32S) | 1 |
| Module relais | Module 4+1 relais 5V actif LOW | 1 ou 2 |
| Vannes | Électrovanne 12V DC normalement fermée | 4 |
| Pompe | Pompe 12V DC (centrifuge ou submersible) | 1 |
| Débitmètre | YF-S201 (G 1/2", 1–30 L/min) | 1 |
| Alimentation | 12V DC / 5A minimum | 1 |
| Régulateur | LM7805 ou module buck 12V→5V | 1 |

---

## Schéma de câblage

```
                    ┌─────────────────────┐
              5V ──►│ VIN             3V3 │
             GND ──►│ GND             GND │
                    │                     │
  Relais Zone 1 ◄───│ GPIO26          IO34│◄── YF-S201 signal
  Relais Zone 2 ◄───│ GPIO27          IO35│◄── [RÉSERVÉ] Sol
  Relais Zone 3 ◄───│ GPIO14              │
  Relais Zone 4 ◄───│ GPIO12              │
  Relais Pompe  ◄───│ GPIO13              │
                    └─────────────────────┘
                         ESP32-WROOM-32
```

---

## Module relais (actif LOW — standard chinois)

```
ESP32 GPIO ──► IN1..IN5 (module relais)
ESP32 5V   ──► VCC
ESP32 GND  ──► GND

          COM ──► + alimentation 12V
          NO  ──► + vanne / pompe
          (NC non utilisé)

          - vanne / pompe ──► GND alimentation 12V
```

> **Actif LOW** = la bobine du relais est activée quand le GPIO est à 0V (LOW).
> C'est le comportement par défaut des modules relais vendus sur Amazon/AliExpress.
> Le YAML utilise `inverted: true` pour compenser (turn_on → GPIO LOW).

Si votre module est actif HIGH, passer `inverted: false` dans chaque switch.

---

## Débitmètre YF-S201

```
YF-S201
  ├── Fil rouge   ──► 5V
  ├── Fil noir    ──► GND
  └── Fil jaune   ──► GPIO34 (+ résistance pull-up 10kΩ vers 3.3V)
                       OU utiliser l'option pullup: true dans le YAML
```

**Calibration** : 7,5 pulses/seconde = 1 L/min → facteur `0.1333`

Pour d'autres modèles :
```
facteur = 1 / (pulses_par_litre / 60)
```

---

## Sonde humidité sol (GPIO35 — placeholder)

GPIO35 est réservé mais non câblé. Pour ajouter une sonde :

```
Sonde capacitive (ex: DFRobot SEN0193)
  ├── VCC  ──► 3.3V
  ├── GND  ──► GND
  └── AOUT ──► GPIO35

Décommenter le bloc sensor adc dans arrosage-jardin.yaml
et calibrer les valeurs min/max (sec / saturé).
```

---

## Alimentation

```
12V DC 5A
  │
  ├──► Vannes (via relais) : ~300 mA chacune, une à la fois
  ├──► Pompe (via relais)  : ~1-3 A selon modèle
  │
  └──► Module buck 12V → 5V ──► ESP32 VIN (5V / 500 mA)
                              ──► Module relais VCC
```

> **Ne pas alimenter l'ESP32 via USB en même temps que le 12V** si vous utilisez un régulateur 12V→5V branché sur VIN.

---

## Protection optionnelle recommandée

- **Diode de roue libre** (1N4007) en parallèle sur chaque vanne (cathode côté +12V) pour protéger les relais des pics d'induction
- **Fusible** 5A sur le +12V principal
- **Boîtier étanche** IP65 minimum (installation extérieure)
