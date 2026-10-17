# i-Vent Rekuperator — Home Assistant Integration

Custom HACS integration for [i-Vent](https://i-vent.com/) heat recovery ventilation units via the **i-Vent Cloud API**.

## Screenshots

### i-Vent Mobile App
<img src="docs/ivent-app-kabinet.png" width="500" alt="i-Vent app - Kabinet room control">

*Native i-Vent app showing room controls: mode (Rekuperacija/Bypass/Boost/OFF), fan speed (Low/Med/High), and presets (Normal/Snooze/Night/Night 2)*

### Home Assistant — Overview
<img src="docs/ha-rekuperatorji-overview.png" width="400" alt="HA overview of all recuperators">

*All 8 i-Vent units displayed in Home Assistant with power, fan, and mode controls*

### Home Assistant — Climate Card
<img src="docs/ha-climate-card.png" width="400" alt="HA climate thermostat card">

*Climate entity with thermostat dial showing current HVAC mode*

### Home Assistant — Mode Selection
<img src="docs/ha-mode-selector.png" width="400" alt="HA HVAC mode selector">

*Available HVAC modes: Heat/Cool (recovery), Fan only (ventilation), Off*

### Home Assistant — Preset Modes
<img src="docs/ha-preset-selector.png" width="400" alt="HA preset mode selector">

*Preset modes: None, Boost, Snooze, Night, Night Silent*

### Home Assistant — Fan Speed
<img src="docs/ha-fan-mode-selector.png" width="400" alt="HA fan mode selector">

*Fan speed control: Low, Medium, High, Boost*

## Why This Is Useful

i-Vent recuperators have no official Home Assistant integration. This integration exposes your entire ventilation system to HA, enabling:

- **Central overview** — see all recuperators in one dashboard instead of checking each room in the i-Vent app
- **Mode control** — switch between recovery, ventilation, and off modes per room
- **Fan speed control** — set fan speed (low, medium, high, boost) per unit
- **Airflow direction** — see whether each unit is in supply (dovod) or extract (odvod) mode
- **LED and buzzer control** — toggle the LED indicator and buzzer on each unit
- **Automations** — turn on boost when cooking or when CO₂ is high, turn bedrooms off at night, lower the fan when nobody's home

## Do you need an i-Cube?

[i-Cube](https://i-vent.com/si/prezracevalni-sistemi/i-cube/) is i-Vent's wireless air-quality sensor. It measures CO₂, VOC, humidity and temperature and adjusts up to 4 i-Vent zones automatically. It is a good product if you want a plug-and-play solution, but **if you already run Home Assistant, this integration plus any air-quality sensor you already own can do the same job** — usually for less money and with more flexibility.

| | i-Cube | Home Assistant + this integration |
|---|---|---|
| **Cost** | ~249 € + VAT (≈ 304 €) | free integration + any sensor you already have (or e.g. IKEA VINDSTYRKA / Aqara, ~30–60 €) |
| **Sensors** | CO₂, VOC, humidity, temperature, pressure — one location | any number of sensors, in every room, any brand HA supports |
| **Control logic** | i-Vent's built-in thresholds | your own automations: CO₂, humidity, presence, time of day, weather, window contacts, cooking, … |
| **Zones** | up to 4 | every i-Vent group in your location |
| **Combines with the rest of your home** | no | yes — lights, presence, calendars, notifications to your phone |
| **Works without Home Assistant** | **yes** | no |
| **Works without internet** | i-Vent's local link | no — this integration uses the i-Vent Cloud API |
| **Setup effort** | place it and pair it | install, configure, write a few automations |

**In short:** no Home Assistant → i-Cube is the easy answer. Already on Home Assistant → you most likely don't need it.

> Note: the i-Vent units themselves have **no temperature, humidity or CO₂ sensor** (verified against the device protocol). Room air quality always has to come from a separate sensor — either an i-Cube or a sensor in Home Assistant.

### Example: boost when CO₂ is high

```yaml
automation:
  - alias: "i-Vent: boost on high CO2"
    triggers:
      - trigger: numeric_state
        entity_id: sensor.living_room_co2
        above: 1200
        for: { minutes: 5 }
    actions:
      - action: fan.set_preset_mode
        target: { entity_id: fan.i_vent_dnevna_ventilator }
        data: { preset_mode: boost }
```

## Known behaviour: automatic night mode

Some installations switch **all rooms** into night mode (speed 2 for 1 h, then "night silent" for 8 h) every evening, without anyone pressing a button — most likely triggered by a light sensor when a room goes dark. In Home Assistant this shows up as `fan_only` with preset `night` / `night_silent`. If you don't want it, a small automation can put the rooms back into recovery mode:

```yaml
automation:
  - alias: "i-Vent: no automatic night mode"
    mode: parallel
    triggers:
      - trigger: state
        entity_id: [climate.i_vent_dnevna, climate.i_vent_kabinet]   # your rooms
        attribute: preset_mode
        to: [night, night_silent]
        for: { minutes: 1 }
    actions:
      - action: climate.set_hvac_mode
        target: { entity_id: "{{ trigger.entity_id }}" }
        data: { hvac_mode: heat_cool }
      - delay: { seconds: 10 }
      - action: climate.set_fan_mode          # heat_cool alone keeps the night speed
        target: { entity_id: "{{ trigger.entity_id }}" }
        data: { fan_mode: low }
```

## Prerequisites

You need two values from the i-Vent Cloud platform:

1. **API Key** — your authentication token from [cloud.i-vent.com](https://cloud.i-vent.com)
2. **Location ID** — the numeric ID of your i-Vent installation

### How to get your API Key and Location ID

1. Log in to [cloud.i-vent.com](https://cloud.i-vent.com)
2. Open your browser's Developer Tools (F12) → Network tab
3. Navigate around the dashboard and look for API requests to `/api/v1/live/...`
4. The `Authorization: Bearer <token>` header contains your **API Key**
5. The URL path `/live/{number}/info` contains your **Location ID**

## Installation

### HACS (Custom Repository)

1. In HACS, go to Integrations → ⋮ → Custom Repositories
2. Add this repository URL, category: Integration
3. Install "i-Vent Rekuperator"
4. Restart Home Assistant
5. Go to Settings → Devices & Services → Add Integration → i-Vent
6. Enter your API Key and Location ID

### Manual

1. Copy `custom_components/ivent/` to your HA `config/custom_components/` directory
2. Restart Home Assistant
3. Go to Settings → Devices & Services → Add Integration → i-Vent
4. Enter your API Key and Location ID

## Entities

Each i-Vent unit (group) creates the following entities:

| Platform | Entity | Description |
|----------|--------|-------------|
| `climate` | i-Vent {name} | HVAC mode (recovery, ventilation, off), fan speed, presets |
| `fan` | {name} Ventilator | Fan on/off and speed control (low/medium/high/boost) |
| `sensor` | {name} Smer pihanja | Airflow direction (Dovod / Odvod) |
| `switch` | {name} LED | LED indicator on/off |
| `switch` | {name} Buzzer | Buzzer on/off |

## Roadmap

- [ ] **Local UDP diagnostics** — real fan speed (in/out %), motor RPM and current, Wi-Fi signal and status codes read directly from each unit (no cloud needed). The units do not report temperature, humidity or filter state.
- [ ] **Local control** — switch modes over the local protocol, so the integration keeps working without internet
- [ ] **Multi-location support** — allow multiple i-Vent locations in a single HA instance
- [ ] **HACS default repository** — submit to HACS default repo once stable

## License

MIT
