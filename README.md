<div align="center">

# Home Assistant Blueprint: MQTT Air Multisensor

Create a virtual multi-sensor (temperature, humidity and optional pressure) that publishes a single JSON state and auto-creates individual MQTT Discovery sensor entities – with automatic (or manual) area-aware naming and per‑sensor precision control.

This is useful when you have separate helper entities that calculate averages from many sensors and you want to put them into one device for each room or to export via Matter etc.

</div>

## ✨ Features

- Single consolidated JSON state published to `<base_topic>/state`.
- Automatic MQTT Discovery of individual temperature / humidity / (optional) pressure sensors.
- Optional pressure sensor (leave blank to skip).
- Automatic area detection (can also supply or disable manually).
- Optional automatic appending of area to the device name.
- Custom per‑sensor displayed names (translation / wording) independent from device name.
- Individual precision (decimal places) per sensor applied in the value template (raw payload stays full precision from source sensors).
- Stable unique_id generation with optional manual prefix override.
- Periodic refresh plus immediate publish on state change (configurable toggle & interval).
- Retained discovery + retained state (configurable retain & QoS).
- Safe Jinja templates (no unsafe dict mutation) compatible with HA sandbox.

## 📁 File Location

Blueprint file: `blueprints/automation/krz/mqtt_air_multisensor.yaml`

Place it under your Home Assistant `config/` blueprint path (keep folder structure or any folder you prefer under `blueprints/automation/`).

## 🚀 Quick Start

1. Either copy the YAML blueprint file into your HA `config/blueprints/automation/...` directory or click Import Blueprint and use this link `https://github.com/krzkraw/Create-MQTT-air-multisensor-from-entities/blob/main/blueprints/automation/krzkraw/mqtt_air_multisensor.yaml`
2. If installing by manual copying: In HA UI: Settings → Automations & Scenes → Blueprints → Import Blueprint → (select the file) or Refresh if already present.
3. Create an automation from the blueprint.
4. Fill required inputs: temperature & humidity entities (pressure optional).
5. (Optionally) set Suggested area or leave blank to let auto‑detection pick an area from your source entities.
6. Adjust precision, naming, and MQTT options if desired.
7. Save. The automation publishes discovery messages at HA start and when you trigger the custom refresh event.

## 🔧 Inputs Overview

Grouped logically inside the blueprint UI (grouping uses nested input containers supported by newer HA versions):

1. Location / Area
  - Suggested area: Text appended to default device name (if enabled) and sent as `suggested_area` in discovery.
  - Auto detect area: Derives area from first available source entity if Suggested area blank.
2. Source sensor entities
  - Temperature entity (required)
  - Humidity entity (required)
  - Pressure entity (optional)
3. Entity naming / translation
  - Temperature / Humidity / Pressure created entity names (defaults: Temperature, Humidity, Pressure)
4. Device name & model
  - Device name (default: "Air multisensor")
  - Appended area (boolean, default true) – if on and an area is resolved, device name becomes `"<Device name> - <Area>"`.
  - Device model (default: "KRZ's air multisensor")
5. Precision settings (decimals)
  - Temperature (default 1)
  - Humidity (default 0)
  - Pressure (default 1)
6. Updates & behaviour
  - Publish interval (seconds) – periodic refresh tick (default 3600)
  - Publish on state change (boolean, default true)
7. MQTT options
  - QoS (0–2, default 0)
  - Retain (default true)
  - Base MQTT topic (blank = auto-generated: `homeassistant/sensor/<normalized_device_name>`)
  - Discovery prefix (default `homeassistant`; blank disables discovery entirely)
  - unique_id prefix (blank = derived from normalized device name)

## 🧠 Naming Logic

1. Start with user `Device name` (default: Air multisensor).
2. Resolve `suggested_area`:
  - If user input provided → use it.
  - Else if auto detect enabled → first non-empty `area_name()` from temperature, humidity, (pressure) entities.
  - Else blank.
3. If `Appended area` enabled AND resolved area non-empty → final device name = `<Device name> - <Area>`.
4. Each created entity gets the custom name you set in Entity naming / translation (no automatic addition of device name).
5. Normalized device name (`norm_device_name`) used to form a default base topic and unique_id prefix (lowercase, non-alphanumeric → `_`).

## 📦 Published State

Topic: `<base_topic>/state`

With pressure:
```jsonc
{
  "temperature": 22.4,
  "humidity": 51.0,
  "pressure": 1007.8,
  "temperature_uom": "°C",
  "humidity_uom": "%",
  "pressure_uom": "hPa",
  "device": "Air multisensor - Living Room",
  "ts": 1723872000
}
```

Without pressure:
```jsonc
{
  "temperature": 22.4,
  "humidity": 51.0,
  "temperature_uom": "°C",
  "humidity_uom": "%",
  "device": "Air multisensor - Living Room",
  "ts": 1723872000
}
```

Precision rounding is applied only inside each discovery sensor's `value_template`, preserving raw payload values for downstream processing.

## 🔍 MQTT Discovery

If a discovery prefix is provided (non-empty): retained configs are published to
```
<discovery_prefix>/sensor/<uid_prefix>/temperature/config
<discovery_prefix>/sensor/<uid_prefix>/humidity/config
<discovery_prefix>/sensor/<uid_prefix>/pressure/config   (only if pressure provided)
```

Each config includes:
- name (custom per sensor)
- state_topic (shared JSON state)
- `value_template` with per-sensor rounding
- unit_of_measurement
- device_class & state_class=measurement
- suggested_display_precision
- suggested_area (if resolved)
- unique_id (prefix-based)
- device object (identifiers, model, manufacturer, sw_version)
- expire_after (2 × publish interval) – helps mark sensors unavailable if updates stop

## 🔄 Refresh / Republish

Trigger a republish of retained discovery configs by firing the custom event:
```
mqtt_discovery_retain_refresh
```
via Developer Tools → Events → Fire Event.

## ❌ Removing / Cleaning Up

1. Delete or disable the automation (state publication stops).
2. (Optional) Manually publish an empty retained payload (`""`) to each discovery `.../config` topic to make HA remove entities instantly (HA usually cleans them once config is gone or changed).

## 🧪 Troubleshooting

| Issue | Check |
|-------|-------|
| No entities created | Discovery prefix empty? Broker retained messages allowed? MQTT integration loaded? |
| Wrong area in name | Suggested area overrides auto detection; clear it or disable auto detect. |
| Duplicate devices | unique_id prefix collision – set a manual unique_id prefix. |
| Stale values | Retain on + no new publishes; verify source sensors changing & interval not too large. |
| Precision not applied | Rounding happens only in discovery sensor `value_template`, not in raw state JSON. |

## 🧩 Extending

Ideas you can implement by forking the blueprint:
- Add dew point / heat index calculations and publish extra discovery sensors.
- Add an availability topic (publish online/offline & include `availability_topic` in configs).
- Add battery level / RSSI if relevant for your hardware.
- Add a toggle to always append area even when a fully custom device name is used.

## 📝 YAML Notes

`!input` tags are blueprint placeholders resolved by Home Assistant; external YAML linters may flag them. Nested input groups (`device:`, `precision:` etc.) rely on newer HA blueprint schema – if your version is older and rejects the file, flatten those groups by moving their `input:` children to the top level.

## 📄 License

Public domain / unrestricted use. Attribution appreciated but not required.

---

Enjoy your unified MQTT air multisensor! PRs & suggestions welcome.
