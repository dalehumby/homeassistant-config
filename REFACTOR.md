# Config cleanup / refactor plan

Review of all YAML config files, done 2026-07-14 (HA 2026.7.2) with Claude Code.
Findings verified against the live instance: all ~110 entities referenced in YAML were
checked for existence/availability; scripts checked via the config API; system log scanned.

**Overall verdict:** config is in good shape — small (~2,300 lines), mostly modern
`triggers:`/`actions:` syntax, almost every referenced entity exists. Cleanup is mostly
dead files, a few legacy idioms, and some duplication.

**Ground rules:** suggestions only until discussed. This is a family home — no breaking
stuff. Work one item at a time, `yamllint` + config check + `ha_reload_core` after each,
verify behavior before moving on.

---

## Open questions (blockers for specific items)

- [ ] Is the **pn532 ESP (NFC tag reader)** at 10.0.0.126 still in service? It was
      unreachable during review. If retired, the tag automations (`laundry_start`,
      `lunch_reminer_on/off`) are dead code too.
- [ ] Is the **mimic3/marytts container** (`home_mimic3`) still running? Decides A7.
- [ ] Is the **Vindriktning ESP** (10.0.0.13) retired? It was offline and
      `light.vindriktning_status` unavailable. Decides A8.
- [ ] What do the **NSPanel dashboards / Node-RED flows** reference? Decides the fate of
      `binary_sensor.entrance_hall_occupancy`, `binary_sensor.study_show_sleep_button`,
      `binary_sensor.illuminance_bright`, `input_datetime.easy_wakeup`, `timer.variable`,
      and whether the scene renames in B15 are safe.
- [ ] Why is `climate.bedroom_thermostat` **unavailable**? (Battery / removed for
      summer?) Two automations set it daily and currently fail silently.

---

## A. Dead code — safe deletions (after consumer check)

- [ ] **A1. Delete `google_calendars.yaml`** — relic of pre-2022 YAML Google Calendar
      setup; NOT `!include`d by `configuration.yaml`; contains old-employer (Nomanini),
      Spotify, SA/US holiday calendars. Already gitignored. ⚠️ The Google integration
      historically still reads this file for calendar entity config — confirm calendars
      are UI-managed first.
- [ ] **A2. Delete `known_devices.yaml`** — legacy device_tracker registry; one empty
      `dale_apple_watch` entry (no MAC); no YAML tracker platforms exist. Dead.
- [ ] **A3. Delete `binary_sensor.yaml`** (comments only) + its include at
      `configuration.yaml:140`.
- [ ] **A4. Remove `input_datetime.easy_wakeup`** (`configuration.yaml:56`) — nothing in
      YAML references it; the easy-wakeup automation uses `dale_alarm_time`. Verify no
      dashboard use first.
- [ ] **A5. Remove `timer.variable`** (`timer.yaml:7`) — zero YAML references. Possibly
      an old dashboard/Siri target; verify first.
- [ ] **A6. Remove `binary_sensor.illuminance_bright`** (`template.yaml:104`) — no
      automation uses it; near-inverse duplicate of `illuminance_needs_light`. Verify no
      dashboard use first.
- [ ] **A7. Remove `tts: platform: marytts`** (`configuration.yaml:122`) — no YAML calls
      `tts.marytts_say`. Only if mimic3 container is retired.
- [ ] **A8. Remove Google Assistant `entity_config` for `light.vindriktning_status`**
      (`configuration.yaml:108-110`) — only if the Vindriktning is retired.
- [ ] **A9. Remove empty `camera:` and `media_player:` keys** from `configuration.yaml`
      — no YAML platforms use them; UI integrations don't need them.
      NOTE: the other bare keys (`person:`, `frontend:`, `history:`, `sun:`, `logbook:`,
      `mobile_app:`, `ios:`, `system_health:`, `media_source:`) ARE required because
      `default_config:` is not used — leave them. Switching to `default_config:` is a
      separate discussion (pulls in cloud, conversation, energy, etc.; needs restart).

Verified NOT dead (leave alone): `ip_bans.yaml` (runtime file),
`binary_sensor.entrance_hall_occupancy` / `binary_sensor.study_show_sleep_button`
(no YAML consumers but almost certainly NSPanel/Node-RED — verify before touching).

## B. Legacy syntax — mechanical modernization, no behavior change

- [ ] **B10. `data_template:` → `data:`** (deprecated ~2020). Seven occurrences:
      `automations.yaml:87, 120, 240, 298, 322, 717, 1130`.
- [ ] **B11. Normalize trigger syntax** — `platform: time_pattern` →
      `trigger: time_pattern` (`automations.yaml:1215`, last old-style trigger there);
      trigger-based template sensor uses `platform:` keys (`template.yaml:139-155`).
- [ ] **B12. Migrate legacy TTS** ⚠️ family-facing, own task, test on one speaker first.
      `tts.google_cloud_say` + top-level `entity_id` → `tts.speak` targeting the TTS
      entity with `media_player_entity_id`. The YAML `tts: platform: google_cloud` setup
      is also legacy (Google Cloud has a config flow now).
      Call sites: `automations.yaml:82, 115, 238, 293, 317, 715, 1128`.
- [ ] **B13. Add `unique_id` to all template entities** — every template
      sensor/binary_sensor and the `lounge_candles` template switch lack one (only
      Fridge Too Warm has it). Doesn't change entity IDs; enables UI management
      (area/icon/hide). Zero risk. (MQTT sensors already have unique_ids.)
- [ ] **B14. `mqtt.yaml` Wunderground sensors** — add `state_class: measurement` (enables
      long-term statistics); optionally a shared `device:` block to group as one device;
      prefer `suggested_display_precision` over `round()` in `value_template`.
- [ ] **B15. Typos** — "tomorrom" in lunch TTS message (`automations.yaml:241`) is safe.
      ⚠️ Scene names "Bedroom lights **bight**" / "Study lights **bight**"
      (`scenes.yaml:141, 171`): renaming changes entity IDs
      (`scene.bedroom_lights_bight`, `scene.study_lights_bight`) — check NSPanel buttons
      and Google Assistant routines first. Leave `lunch_reminer_*` automation IDs alone
      (IDs invisible; changing churns traces/history) — fix aliases only if desired.

## C. Structural improvements — discuss before doing

- [ ] **C16. Deduplicate rain alerts** (`automations.yaml:123-206`) — three automations
      share an identical two-notification action block (title/mm message +
      `update_complications` push). Extract `script.rain_alert` or merge into one
      automation with trigger IDs.
- [ ] **C17. Replace `sensor.time` string-match triggers with native time trigger +
      offset** (supported since 2024.10):
      `easy_wakeup` (`automations.yaml:610`) → `trigger: time / at:
      input_datetime.dale_alarm_time / offset: "-00:10:00"`;
      `bedroom_thermostat_day` (`automations.yaml:738`) → same with `-01:00:00`.
      Validated at load; removes the `sensor.time` dependency.
- [ ] **C18. Close the guest-mode gap** — `input_boolean.guest_mode` only guards the
      goodnight scene + vacuum-flag reset. But `all_lights_off` (line 746) and
      `start_robot_vacuum_when_away` (line 327) key off tracked-person count: untracked
      guests get lights-off and vacuum. Add `guest_mode = off` condition to both.
      (Small behavior change, drama-prevention.)
- [ ] **C19. Label the opaque Z2M device IDs** — device triggers are fine per best
      practice, but add a comment naming each device:
      `bc5b0d80…` = bedside switch, `8858514f…` = bathroom switch,
      `627eeddb…` = towel-rail switch, `1fc5be6b…` = cat doorbell. Zero risk.
- [ ] **C20. (Parked / future)** HA 2026.7 purpose-specific triggers
      (`motion.detected` with area targets) could simplify the motion-light automations.
      A rewrite, not a cleanup — revisit later.

## D. Live issues noticed (not YAML cleanup)

- pn532 ESP offline → tag automations possibly dead (see open questions).
- `climate.bedroom_thermostat` unavailable → daily set_temperature automations fail
  silently.
- Dashboard cards using `states('sensor.precipitation_next_hour') | float` without a
  default throw template errors while the sensor is `unknown` at startup — add
  `| float(0)` in those dashboard card templates (storage dashboards, not this repo).
- ICA shopping list integration getting 403s / can't find list ID
  `669cfebb-5f5e-4178-88ac-2df6b6fdd18e`; the sync automation runs into this every
  10 minutes.

## E. Deliberately leaving alone

- Adaptive-lighting delay choreography in `scripts.yaml` (IKEA bulb workaround).
- WLED double-turn-off (`hall_nightlight_off`) and NSPanel MQTT retry hacks — commented,
  deliberate workarounds for real hardware quirks.
- `sensor.yaml` statistics/min_max platforms — still fully supported in YAML; migrating
  to UI helpers would change entity IDs for no gain.
- `dale_charge_phone` script's bare `condition:` list — verified HA parses it as an
  implicit AND; valid.
- `entity_id: all` in light.turn_off calls — still supported.

---

## Suggested execution order (safest first)

1. A1–A9 dead file/helper deletions (each gated on its consumer check)
2. B10 + B11 syntax normalization
3. B13 unique_ids
4. B14 MQTT state_class/device
5. C19 device-ID comments
6. C16 rain-alert dedup
7. C17 time-trigger offsets
8. C18 guest-mode conditions
9. B12 TTS migration (most care needed)
10. B15 scene renames (only after consumer check)
