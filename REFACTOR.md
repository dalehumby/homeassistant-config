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

- [x] Is the **pn532 ESP (NFC tag reader)** at 10.0.0.126 still in service? — **N/A.**
      NFC tags are read via the phone (iOS NFC), not the pn532 hardware reader. The
      reader being unreachable doesn't affect the tag automations either way; `laundry_start`
      and `lunch_reminer_on/off` are NOT dead code. The pn532 ESP's fate is a separate,
      lower-priority question (unrelated to these automations) — not pursued further here.
- [x] Is the **mimic3/marytts container** (`home_mimic3`) still running? — **No, removed.**
      A7 is confirmed safe to do.
- [x] Is the **Vindriktning ESP** (10.0.0.13) retired? — **Not retired**, just unplugged
      for now (may come back). A8 is **on hold** — leave the Google Assistant
      `entity_config` in place until it's actually decommissioned.
- [ ] What do the **NSPanel dashboards / Node-RED flows** reference? Decides the fate of
      `binary_sensor.entrance_hall_occupancy`, `binary_sensor.study_show_sleep_button`,
      `binary_sensor.illuminance_bright`, `input_datetime.easy_wakeup`, `timer.variable`,
      and whether the scene renames in B15 are safe.
- [x] Why is `climate.bedroom_thermostat` **unavailable**? — **Seasonal**: it's offline
      for summer (not a fault). The two daily `set_temperature` automations will keep
      failing silently until it's reconnected in autumn — expected, not a bug. Worth a
      guard condition (skip silently / notify once) if it bothers us, but not urgent;
      see D.

---

## A. Dead code — safe deletions (after consumer check)

- [x] **A1. Delete `google_calendars.yaml`** — relic of pre-2022 YAML Google Calendar
      setup; NOT `!include`d by `configuration.yaml`; contains old-employer (Nomanini),
      Spotify, SA/US holiday calendars. Already gitignored. **Done** — checked
      `.storage/google.<entry_id>`, confirmed the config-entry-based Google integration
      tracks calendars by real calendar ID in its own storage, not via this file (options
      are just `{"calendar_access": "read_write"}`, no calendar list). File deleted.
- [x] **A2. Delete `known_devices.yaml`** — legacy device_tracker registry; one empty
      `dale_apple_watch` entry (no MAC); no YAML tracker platforms exist. Dead. **Done** —
      confirmed no `device_tracker:` platform anywhere in YAML. File deleted.
- [x] **A4. Remove `input_datetime.easy_wakeup`** (`configuration.yaml:56`) — nothing in
      YAML references it; the easy-wakeup automation uses `dale_alarm_time`. Added
      `0fd9f962b0` (Dec 2022) as the manually-set wakeup time; superseded `3d7f8cf`
      (Jan 2024) when the automation switched to `dale_alarm_time` (fed from the phone's
      actual alarm) — the old helper declaration was just never cleaned up. **Done** —
      removed, `input_datetimes` reloaded, entity confirmed gone.
- [x] **A5. Remove `timer.variable`** (`timer.yaml:7`) — zero YAML references. Leftover
      from an old voice-assistant project. **Done** — removed, `timers` reloaded, entity
      confirmed gone.
- [x] **A6. Remove `binary_sensor.illuminance_bright`** (`template.yaml:104`) — no
      automation uses it; near-inverse duplicate of `illuminance_needs_light`. It was the
      predecessor, kept around only to eyeball that `illuminance_needs_light`'s inverted
      logic matched — never cleaned up. **Done** — removed, `templates` reloaded,
      entity confirmed gone; `illuminance_needs_light` unaffected.
- [x] **A7. Remove `tts: platform: marytts`** (`configuration.yaml:122`) — no YAML calls
      `tts.marytts_say`. mimic3 container confirmed removed. **Done** — YAML edited,
      `yamllint` + `check_config` both pass. Pending a full HA restart (not
      hot-reloadable) to actually take effect — held off per user request.
- [ ] **A8. Remove Google Assistant `entity_config` for `light.vindriktning_status`**
      (`configuration.yaml:108-110`) — **on hold**: Vindriktning is only unplugged for
      now, not retired. Revisit if/when it's actually decommissioned.
- [x] **A9. Remove empty `camera:` and `media_player:` keys** from `configuration.yaml`
      — no YAML platforms use them; UI integrations don't need them. Verified all live
      camera (`blitzortung`, `sto_yr_no`) and media_player (`all_speakers`,
      `kitchen_display`, `lounge_speaker`) entities have a `config_entry_id` (UI-managed:
      Generic Camera, Google Cast) — none depend on the bare key. No Spotify integration
      currently configured either. **Done** — keys removed, `check_config` passes.
      Pending a full restart to apply (structural `configuration.yaml` change).
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

- pn532 ESP offline — unrelated to the tag automations (those read NFC tags via phone,
  not the hardware reader); reader's own status is a separate, non-blocking question.
- `climate.bedroom_thermostat` unavailable → seasonal (unplugged for summer), daily
  set_temperature automations fail silently. Expected for now; consider a guard
  condition before autumn if the silent failures are noisy in logs.
- Dashboard cards using `states('sensor.precipitation_next_hour') | float` without a
  default throw template errors while the sensor is `unknown` at startup — add
  `| float(0)` in those dashboard card templates (storage dashboards, not this repo).
- ICA shopping list integration getting 403s / can't find list ID
  `669cfebb-5f5e-4178-88ac-2df6b6fdd18e`; the sync automation runs into this every
  10 minutes.

## E. Deliberately leaving alone

- **`binary_sensor.yaml`** (and its include at `configuration.yaml:140`) — comments-only
  today, but kept intentionally: various old internet instructions/guides reference this
  file, and it's the designated home for `trend`/`threshold` sensors per the header
  comment. Not dead code to delete (was A3) — a placeholder to remember NOT to use unless
  a trend/threshold sensor genuinely needs the legacy `binary_sensor:` platform (prefer
  `template.yaml` otherwise).
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

1. A1–A9 dead file/helper deletions (each gated on its consumer check; A3 excluded —
   see section E)
2. B10 + B11 syntax normalization
3. B13 unique_ids
4. B14 MQTT state_class/device
5. C19 device-ID comments
6. C16 rain-alert dedup
7. C17 time-trigger offsets
8. C18 guest-mode conditions
9. B12 TTS migration (most care needed)
10. B15 scene renames (only after consumer check)
