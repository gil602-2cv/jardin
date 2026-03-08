# Guide — Configuration Smart Irrigation

## Présentation

**Smart Irrigation** (HASmartIrrigation) est une intégration Home Assistant qui calcule automatiquement la durée d'arrosage nécessaire pour compenser l'évapotranspiration (ET₀), en tenant compte de la pluie et des prévisions météo.

Version utilisée dans ce projet : **v2025.10.0**  
Source météo configurée : **Pirate Weather**

---

## Principe de fonctionnement

```
Chaque soir à 23h00 :
  1. Récupération météo (Pirate Weather)
  2. Calcul ET₀ (Penman-Monteith)
  3. Soustraction précipitations
  4. Mise à jour du "bucket" (déficit hydrique en mm)
  5. Calcul durée arrosage = f(bucket, débit, surface zone)
  6. Mise à jour sensor.smart_irrigation_[zone] (en secondes)

Après arrosage :
  → Appeler smart_irrigation.reset_bucket pour remettre le bucket à 0
```

---

## Création des zones dans HA

### Étape 1 — Accéder à Smart Irrigation

**Paramètres → Intégrations → Smart Irrigation → Configurer**

### Étape 2 — Créer les 4 zones

Ajouter une zone pour chacune :

| Nom | Entity ID généré | Notes |
|-----|-----------------|-------|
| `Potager` | `sensor.smart_irrigation_potager` | Légumes |
| `Massifs` | `sensor.smart_irrigation_massifs` | Fleurs |
| `Gazon` | `sensor.smart_irrigation_gazon` | Pelouse |
| `Serre Ext` | `sensor.smart_irrigation_serre_ext` | Serre extérieure |

> Le nom est converti en slug pour l'entity_id : espaces → `_`, majuscules → minuscules.
> Vérifiez l'entity_id généré dans **Développeur → États** après création.

### Étape 3 — Paramètres de chaque zone

Pour chaque zone, configurer selon votre installation :

| Paramètre | Valeur typique | Description |
|-----------|---------------|-------------|
| Surface | 10–50 m² | Surface arrosée |
| Débit | 2–8 L/min | Débit de votre arroseur |
| Kc (coefficient cultural) | 0.5–1.2 | Selon type de plante |
| Efficacité | 75–90% | Efficacité de votre système |

---

## Vérification des entity_id

Après création des zones, vérifier dans **Développeur → États** :

```
sensor.smart_irrigation_potager
  état : 1380        ← durée en secondes (23 minutes)
  attributs :
    bucket: -5.2     ← déficit hydrique (mm négatif = arrosage requis)
    netto_precipitation: 0.0
    ...
```

Si les entity_id ne correspondent pas exactement, mettre à jour `armoire-semis.yaml` :

```yaml
# Bloc sensor: — remplacer les entity_id
  - platform: homeassistant
    id: si_zone1_duree
    entity_id: sensor.smart_irrigation_VOTRE_NOM_ICI  # ← adapter
```

---

## Automation HA — Arrosage automatique

Exemple d'automation à créer dans HA pour déclencher l'arrosage automatiquement :

```yaml
# automation.yaml ou via l'interface HA
alias: "Arrosage automatique - Potager"
description: "Déclenche l'arrosage Zone 1 selon calcul Smart Irrigation"
trigger:
  - platform: time
    at: "06:00:00"
condition:
  - condition: numeric_state
    entity_id: sensor.smart_irrigation_potager
    above: 0          # Arroser seulement si durée > 0
  - condition: state
    entity_id: binary_sensor.arrosage_en_cours
    state: "off"      # Pas d'arrosage en cours
action:
  - service: switch.turn_on
    target:
      entity_id: switch.arrosage_zone1
  - delay:
      seconds: "{{ states('sensor.smart_irrigation_potager') | int }}"
  - service: switch.turn_off
    target:
      entity_id: switch.arrosage_zone1
  - service: smart_irrigation.reset_bucket
    data:
      entity_id: sensor.smart_irrigation_potager
mode: single
```

Créer une automation similaire pour chaque zone (avec décalage de 5 min entre zones).

---

## Bouton Reset sur l'écran

L'écran `armoire-semis` expose un bouton **Reset** par zone qui appelle directement :

```yaml
homeassistant.service:
  service: smart_irrigation.reset_bucket
  data:
    entity_id: sensor.smart_irrigation_potager
```

À utiliser après un arrosage **manuel** pour indiquer à Smart Irrigation que la zone a été arrosée.

---

## Dépannage

| Symptôme | Solution |
|----------|----------|
| Durée affichée "Calcul en cours..." | Attendre la première synchronisation (23h00) |
| Durée toujours 0 | Vérifier le bucket (peut être positif si pluie récente) |
| Entity_id non trouvé | Vérifier l'orthographe exacte dans Développeur → États |
| Pirate Weather ne répond pas | Vérifier la clé API dans la config Smart Irrigation |
