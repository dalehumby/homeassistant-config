# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

This is a [Home Assistant](https://www.home-assistant.io/) configuration repository — YAML files that define all automations, sensors, lights, scripts, and scenes for a home running HA. There is no build step or test runner; changes take effect when HA reloads the relevant config or restarts. The repo is checked out directly into the HA config directory (provisioned via [rpi-provision](https://github.com/dalehumby/rpi-provision)).

## Validating changes

HA has a built-in config checker. Run it from the HA UI under **Developer Tools → YAML → Check Configuration**, or via the CLI if `ha` is available:

```bash
ha core check
```

To reload only specific domains without restarting HA (available under **Developer Tools → YAML**):
- Automations, Scripts, Scenes, Templates, etc. each have individual reload buttons.

## File structure and where things go

| File | Purpose |
|---|---|
| `configuration.yaml` | Top-level config; includes all other files |
| `automations.yaml` | All automations (flat list, each with a unique `id`) |
| `template.yaml` | **Preferred** place for new sensors and binary sensors — uses the `template:` platform |
| `sensor.yaml` | **Legacy** — only for `statistics` and `min_max` aggregation sensors that can't be expressed as templates |
| `binary_sensor.yaml` | **Legacy** — only for `trend`/`threshold` sensors |
| `scripts.yaml` | Reusable action sequences called from automations or the UI |
| `scenes.yaml` | Named light/device states; note `color_name` is not supported in scenes |
| `lights.yaml` | Light groups (`platform: group`) and switch-backed lights (`platform: switch`) |
| `mqtt.yaml` | MQTT sensors (e.g., Wunderground weather station) |
| `timer.yaml` | Named timers |
| `customize.yaml` | Entity customisations |

## Key architectural patterns

**Adaptive Lighting** is a custom component (`custom_components/adaptive_lighting`) that continuously adjusts brightness and colour temperature. Manual control must be explicitly cleared (`adaptive_lighting.set_manual_control: false`) before calling `adaptive_lighting.apply`. The `adaptive_lights_on` script in `scripts.yaml` shows the canonical pattern.

**Illuminance hysteresis** — Two separate `input_number` helpers (`illuminance_lower`, `illuminance_upper`) define a deadband. The `binary_sensor.illuminance_bright` and `binary_sensor.illuminance_needs_light` sensors in `template.yaml` use different trigger strategies to avoid oscillation. Automations check these sensors rather than raw lux values.

**Scene snapshots** — When a transient brightness change is needed (e.g., entrance PIR), a scene snapshot is created before the change and restored afterwards (see `entrance_lights_brighten` / `entrance_lights_restore` in `automations.yaml`).

**Notification targets** — `notify.family` sends to all family members' phones. `notify.mobile_app_dales_iphone` is Dale's phone only. TTS announcements go to `media_player.kitchen_display`, `media_player.bedroom_speaker`, and `media_player.lounge_speaker`.

**Secrets** — All credentials and location data are in `secrets.yaml` (git-ignored). Reference them with `!secret <key>`.

**Preferred entity types** — When adding new derived sensors or binary sensors, use `template.yaml` (the `template:` platform), not the legacy `sensor.yaml` or `binary_sensor.yaml` files.

## Custom components (installed via HACS, git-ignored)

- `adaptive_lighting` — circadian rhythm light adjustment
- `spotcast` — cast Spotify to Chromecast/Google Home speakers
- `nordpool` — electricity spot price sensor
- `bambu_lab` — Bambu 3D printer integration
- `google_keep_sync` — sync Google Keep lists
- `ica_shopping` — ICA supermarket shopping list
- `blitzortung` — lightning strike sensor
- `nodered` — Node-RED companion
- `spook` — HA utilities/repairs
- `xiaomi_miio_airpurifier` — Xiaomi air purifier

## Home location

Stockholm, Sweden (`Europe/Stockholm`, metric, `country: SE`). Zones defined: Home (50 m radius), Work (75 m), Mall (100 m).
