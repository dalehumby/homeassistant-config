# CLAUDE.md

This is a [Home Assistant](https://www.home-assistant.io/) configuration repository — YAML files that define all automations, sensors, lights, scripts, and scenes for a home running HA. There is no build step or test runner; changes take effect when HA reloads the relevant config or restarts. The repo is checked out directly into the HA config directory.

## Infrastructure

HA runs as a Docker container (not HA OS) on the home server. Related services, also containerised on the same host:

| Service | Role |
|---|---|
| `zigbee2mqtt` | Zigbee coordinator → MQTT bridge (all Zigbee devices) |
| `mosquitto` | MQTT broker (zigbee2mqtt, ESPHome, and MQTT sensors publish here) |
| `esphome` | Firmware and dashboard for ESP32/ESP8266 devices  |
| `node-red` | Supplementary automations and flows; companion integration exposed to HA |

## MCP server

A `home-assistant` MCP server is configured in `.mcp.json`. Use its tools to interact with the live HA instance — inspect state, control devices, reload config, and debug automations — without needing the HA UI or docker commands.

After editing YAML files, use `ha_reload_core` to apply changes (target: `automations`, `scripts`, `scenes`, `templates`, `all`, etc.) rather than restarting. Use `ha_restart(confirm=True)` only for structural `configuration.yaml` changes.

## YAML linting

`yamllint` (v1.38.0) is installed and configured via `.yamllint`. After editing any YAML file, run:

```bash
yamllint -c .yamllint <file>
```

A pre-commit git hook (`.git/hooks/pre-commit`) automatically lints all staged `.yaml`/`.yml` files and then runs the Home Assistant config check inside the running Docker container (`home_homeassistant`). Both must pass or the commit is aborted.

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
- `nordpool` — electricity spot price sensor
- `bambu_lab` — Bambu 3D printer integration
- `google_keep_sync` — sync Google Keep lists
- `ica_shopping` — ICA supermarket shopping list
- `blitzortung` — lightning strike sensor
- `nodered` — Node-RED companion
- `spook` — HA utilities/repairs
- `xiaomi_miio_airpurifier` — Xiaomi air purifier

## Home location

Stockholm, Sweden (`Europe/Stockholm`, metric, `country: SE`).
Zones defined: Home (50 m radius), Work (75 m), Mall (100 m).

## Residents

Dale and Mark. Both tracked via iPhone for presence (`person.dale`, `person.mark`). `notify.family` notifies both; `notify.mobile_app_dales_iphone` is Dale only.

## Rooms

Entrance, hall, bedroom, study, bathroom, kitchen (open to dining room), lounge, balcony, storeroom.

## Notable devices

**NSPanels** — Three Sonoff NSPanel Pro touchscreen displays running Android, located in the entrance, study, and bedroom. Entities are prefixed `nspanel_*`. Each runs the HA companion app (device tracker, media control) plus [nspanel_pro_tools_apk](https://github.com/seaky/nspanel_pro_tools_apk) which exposes additional sensors (ambient light, proximity) and screen control to HA.

**WLED** — Three WLED LED controllers:
- Hall cabinet (`light.hall_cabinet`) — supports effects (Candle Multi, Twinklefox, etc.)
- Lounge TV cabinet (`light.lounge_tv_cabinet`) — supports effects
- Balcony lights (`light.balcony_lights`) — 30 individually addressable segments
