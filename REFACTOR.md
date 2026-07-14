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
- [x] What do the **NSPanel dashboards / Node-RED flows** reference? — Checked all 7
      Lovelace storage dashboards (cards/badges/headers) and Node-RED's `flows.json`
      directly. `binary_sensor.study_show_sleep_button` **is used on a dashboard** —
      confirmed by user, keep. `binary_sensor.entrance_hall_occupancy` has zero
      references anywhere inspectable (automations/scripts/scenes/dashboards/Node-RED);
      confirmed obsolete by user (see A10). `illuminance_bright`/`easy_wakeup`/
      `timer.variable` already resolved (A4-A6, done). Whether the B15 scene renames are
      safe (NSPanel button/Google Assistant routine references) is still open — NSPanel's
      own on-device dashboard config isn't visible from here, so that part needs a manual
      check on the panel/routines before renaming.
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
      `tts.marytts_say`. mimic3 container confirmed removed. **Done and live** — restarted
      HA, no `tts`-related errors in the log.
- [ ] **A8. Remove Google Assistant `entity_config` for `light.vindriktning_status`**
      (`configuration.yaml:108-110`) — **on hold**: Vindriktning is only unplugged for
      now, not retired. Revisit if/when it's actually decommissioned.
- [x] **A9. Remove empty `camera:` and `media_player:` keys** from `configuration.yaml`
      — no YAML platforms use them; UI integrations don't need them. Verified all live
      camera (`blitzortung`, `sto_yr_no`) and media_player (`all_speakers`,
      `kitchen_display`, `lounge_speaker`) entities have a `config_entry_id` (UI-managed:
      Generic Camera, Google Cast) — none depend on the bare key. No Spotify integration
      currently configured either. **Done and live** — restarted HA, no `camera`/
      `media_player`-related errors in the log.
      NOTE: the other bare keys (`person:`, `frontend:`, `history:`, `sun:`, `logbook:`,
      `mobile_app:`, `ios:`, `system_health:`, `media_source:`) ARE required because
      `default_config:` is not used — leave them. Switching to `default_config:` is a
      separate discussion (pulls in cloud, conversation, energy, etc.; needs restart).
- [x] **A10. Remove `binary_sensor.entrance_hall_occupancy`** (`template.yaml`) —
      combined `entrance_pir_occupancy` + `hall_pir_occupancy` into one virtual sensor to
      cut false triggers (phantom lights-on at night) on the old PIR hardware. Hardware
      was since replaced and light automations now key off a single PIR directly
      (`automations.yaml:39,60,212,260` use `entrance_pir_occupancy`;
      `automations.yaml:448,632,961,984` use `hall_pir_occupancy` — never the combined
      sensor). The false-trigger problem it was built to solve no longer happens.
      Confirmed obsolete by user. Zero references in automations/scripts/scenes, all 7
      Lovelace dashboards, and Node-RED's `flows.json`. **Done** — removed, `templates`
      reloaded, entity confirmed gone.

Verified NOT dead (leave alone): `ip_bans.yaml` (runtime file),
`binary_sensor.study_show_sleep_button` (confirmed used on a dashboard).

## B. Legacy syntax — mechanical modernization, no behavior change

- [x] **B10. `data_template:` → `data:`** (deprecated ~2020). Seven occurrences:
      `automations.yaml:87, 120, 240, 298, 322, 717, 1130`. **Done** — mechanical
      replace_all, confirmed no site had a pre-existing sibling `data:` key
      (yamllint's `key-duplicates` rule would have caught a conflict), `check_config`
      passes, automations reloaded, no new errors in the log.
- [x] **B11. Normalize trigger syntax** — `platform: time_pattern` →
      `trigger: time_pattern` (`automations.yaml:1215`, last old-style trigger there);
      trigger-based template sensor's 4 `platform:` keys (`template.yaml`, the
      `illuminance_needs_light` trigger block) → `trigger:`. **Done** — `yamllint` +
      `check_config` pass, automations + templates reloaded, `illuminance_needs_light`
      confirmed live and updating, no new errors in the log.
- [ ] **B12. Migrate legacy TTS** ⚠️ family-facing, own task, test on one speaker first.
      `tts.google_cloud_say` + top-level `entity_id` → `tts.speak` targeting the TTS
      entity with `media_player_entity_id`. The YAML `tts: platform: google_cloud` setup
      is also legacy (Google Cloud has a config flow now).
      Call sites: `automations.yaml:82, 115, 238, 293, 317, 715, 1128`.
- [x] **B13. Add `unique_id` to all template entities** — **Decided against.**
      Reframed the original "zero risk, enables UI management" pitch: registering a
      template entity gives HA's entity registry a second place `name`/`icon`/`area`
      can live, and a UI-set override there silently wins over YAML forever — exactly
      what happened to 4 of the 9 Wunderground sensors in B14 (one, `wunderground_temperature`,
      stayed stuck on a stale UI name through a YAML rename until manually cleared). User
      prefers YAML as sole source of truth; the entity-registry side door that
      `unique_id` opens is the opposite of that, so not worth it for entities that don't
      have a concrete UI-feature need (voice exposure, area assignment) today.
      Checked the one existing exception, `Fridge Too Warm` (`unique_id: fridge_too_warm`,
      added `730df96`, Jan 2026, "for easier reuse") — its registry entry had zero
      customization (no name/icon/area/voice-exposure override), so it wasn't actually
      serving a UI-management purpose, just sitting there as a latent confusion risk.
      Removed its `unique_id` for consistency. Doing so left an orphaned registry entry
      behind (`template.reload` doesn't prune it automatically — same issue seen with
      the B15 automation-id rename), which briefly caused the entity to show
      `unavailable`/`restored: true`; removed the stale registry entry and reloaded
      again, entity now back to a live computed state with the same entity_id, no
      registry entry, matching every other template sensor.
- [x] **B14. `mqtt.yaml` Wunderground sensors** — add `state_class: measurement` (enables
      long-term statistics); optionally a shared `device:` block to group as one device;
      prefer `suggested_display_precision` over `round()` in `value_template`. **Done**,
      all three, all 9 sensors:
      - `state_class: measurement` added to all.
      - Shared `device:` block (YAML anchor/alias) groups all 9 under one "Wunderground
        Home" device — confirmed via entity registry, all share one `device_id`.
      - `round()` removed from `value_template`s; `suggested_display_precision` set per
        sensor instead (raw precision now kept in the recorded state/statistics, only
        display is rounded).
      - Side effect caught and fixed: grouping under a device makes HA combine
        "<device name> <entity name>" for any entity without its own registry name
        override, which stuttered ("Wunderground Home wunderground humidity") for the 5
        sensors that had never been manually renamed. Shortened those YAML `name:`
        values (Humidity, UV, Wind Speed, Direction, Solar Radiation) to read cleanly
        with the device-name prefix. The 4 sensors with pre-existing custom registry
        names (Temperature, Pressure, 24 hour rainfall, Rainfall rate) were left as-is —
        registry overrides always win over YAML `name:`, so they're unaffected either
        way. `check_config` passes, `mqtt.reload` twice, all 9 entities confirmed live
        with correct `friendly_name`/`state_class`/`suggested_display_precision`, zero
        errors in the log.
- [x] **B15. Typos** — "tomorrom" → "tomorrow" in lunch TTS message
      (`automations.yaml:241`). **Done.**
      Scene names "Bedroom lights **bight**" / "Study lights **bight**"
      (`scenes.yaml:141, 171`) → renamed to "...bright". User chose to fix the resulting
      dashboard breakage rather than avoid it. Found 3 button cards referencing the old
      entity IDs (`scene.bedroom_lights_bight`, `scene.study_lights_bight`) via
      `ha_config_get_dashboard` search across all 7 storage dashboards — none on the
      NSPanel dashboards specifically (`bedroom-smart-switch`, `dashboard-tablets`,
      `lovelace-main` 2nd view) — and updated all 3 to the new entity IDs, verified live.
      No Google Assistant routine references found. **Done.**
      `lunch_reminer_on`/`lunch_reminer_off` automation `id:` fields → renamed to
      `lunch_reminder_on`/`lunch_reminder_off` (typo fix; user said history/traces churn
      is fine, not used often). Since automation entity_id is derived from `alias` (not
      `id`) and the alias was already correct, this created orphaned duplicate entities
      (`automation.turn_on_lunch_reminder_2`/`_2` off) rather than renaming in place —
      the old `automation.turn_on_lunch_reminder`/`turn_off_lunch_reminder` entity
      registrations became stale/unavailable, losing their assigned area + category.
      Cleaned up: removed the two orphaned registry entries, renamed the `_2` entities
      back to the original clean entity_ids, and restored area (`87cdccc600a3...`) +
      category (`01HYQMX62C5...`) — net effect is zero change to entity_ids/organization,
      only the internal `id:` typo is fixed. **Done.**

## C. Structural improvements — discuss before doing

- [x] **C16. Deduplicate rain alerts** (`automations.yaml:123-206`) — three automations
      share an identical two-notification action block (title/mm message +
      `update_complications` push). Extract `script.rain_alert` or merge into one
      automation with trigger IDs. **Done** — went with the script (discussed with
      user): the shared block is a pure, context-free side effect (no `trigger.*`
      references, identical in all three), so extracting it doesn't split logic
      that belongs together — the actual complexity (zones, time windows,
      presence/Wi-Fi checks) stays untouched in each automation's
      `triggers`/`conditions`. Merging into one automation with trigger IDs was
      considered and rejected — it would force a `choose` block re-deriving each
      automation's distinct condition set keyed to trigger id, which is more
      YAML for no readability gain. Added `script.rain_alert` to `scripts.yaml`;
      `rainfall_alert_morning`, `rainfall_alert_afternoon`, `rainfall_alert_away`
      each now call it as their sole action. (`rainfall_alert_frontdoor`, just
      past this block, flashes a light and was never part of the duplication —
      correctly left alone.) `yamllint` + `script.reload` + `automation.reload`
      all pass; script and all three automations fetched back via the config API
      confirming triggers/conditions unchanged and actions collapsed to
      `script.rain_alert`; no new errors in the log.
- [x] **C17. Replace `sensor.time` string-match triggers with native time trigger +
      offset** (supported since 2024.10):
      `easy_wakeup` (`automations.yaml:610`) → `trigger: time / at:
      input_datetime.dale_alarm_time / offset: "-00:10:00"`;
      `bedroom_thermostat_day` (`automations.yaml:738`) → same with `-01:00:00`.
      Validated at load; removes the `sensor.time` dependency. **Done** — both
      converted to `trigger: time` with the dict form of `at:`
      (`entity_id`/`offset`), confirmed against the HA docs (`at:` supports a
      fixed time, an entity_id, or `{entity_id, offset}`). `yamllint` passes,
      `automation.reload` succeeded (schema validated at load), both automations
      fetched back via the config API showing the new trigger, no new errors in
      the log. The remaining `states('sensor.time')` at `automations.yaml:1099` is
      an unrelated TTS message (not a trigger) — out of scope, left alone.
- [x] **C18. Close the guest-mode gap** — `input_boolean.guest_mode` only guards the
      goodnight scene + vacuum-flag reset. But `all_lights_off` (line 746) and
      `start_robot_vacuum_when_away` (line 327) key off tracked-person count: untracked
      guests get lights-off and vacuum. Add `guest_mode = off` condition to both.
      (Small behavior change, drama-prevention.) **Done** — added
      `condition: state, entity_id: input_boolean.guest_mode, state: "off"` to
      `start_robot_vacuum_when_away`'s existing `conditions:` list, and a new
      `conditions:` block (it had none) to `all_lights_off` with the same guard.
      `yamllint` passes, `automation.reload` succeeded, both automations fetched
      back via the config API showing the new condition, `guest_mode` currently
      `off` so no behavior change today, no new errors in the log.
- [x] **C19. Label the opaque Z2M device IDs** — device triggers are fine per best
      practice, but add a comment naming each device:
      `bc5b0d80…` = bedside switch, `8858514f…` = bathroom switch,
      `627eeddb…` = towel-rail switch, `1fc5be6b…` = cat doorbell. Zero risk.
      **Done** — added a trailing `# <device name>` comment to all 9 occurrences
      across 5 automations (`bc5b0d80…` ×6 in `bedroom_bedside_light_switch` and
      `nspanel_bedroom_screen_dark`, `8858514f…` ×2 in `bathroom_light_switch`,
      `627eeddb…` ×1 in `bathroom_towelrail_on`, `1fc5be6b…` ×1 in
      `cat_doorbell`). Names verified against the live device registry
      (`ha_get_device`) rather than trusted as-written — two of the four
      original names in this task item were imprecise: `bc5b0d80…` is actually
      registered as **"Bedside Dale switch"** (not just "bedside switch" — Dale
      and Mark each have their own bedside switch, so the qualifier matters),
      and `627eeddb…` is an IKEA TRADFRI shortcut *button*, registered as
      **"Bathroom towelrail shortcut"** (not a "switch"). `8858514f…` and
      `1fc5be6b…` matched as given ("Bathroom switch", "Cat doorbell").
      Comments now use the exact registry `name` for all four. Comment-only
      change; `yamllint` + `automation.reload` pass, diff confirmed to touch
      only comments.
- [ ] **C20. (Parked / future)** HA 2026.7 purpose-specific triggers
      (`motion.detected` with area targets) could simplify the motion-light automations.
      A rewrite, not a cleanup — revisit later.
- [ ] **C21. (Parked / future) `unique_id` naming convention** — `mqtt.yaml`'s
      Wunderground sensors use dotted `unique_id`s (`wunderground.home.temperature`,
      set 2022-06-25 in `c22080f`) that look like `entity_id`s (`domain.object_id`) but
      aren't — `unique_id` is an opaque registry key with no format requirement, HA
      never parses or splits on it. Still, the dotted style invites confusion with real
      entity_ids. User's established convention going forward is underscores
      (`wunderground_home_temperature`). Not applied retroactively here: changing a
      `unique_id` re-keys the entity registry (equivalent to deleting and recreating the
      entity — loses history/statistics continuity unless carefully migrated), so this
      needs its own deliberate task, not a drive-by rename. TODO left in `mqtt.yaml`.

## D. Live issues noticed (not YAML cleanup)

- pn532 ESP offline — unrelated to the tag automations (those read NFC tags via phone,
  not the hardware reader); reader's own status is a separate, non-blocking question.
- `climate.bedroom_thermostat` unavailable → seasonal (unplugged for summer), daily
  set_temperature automations fail silently. Expected for now; consider a guard
  condition before autumn if the silent failures are noisy in logs.
- ~~Dashboard cards using `states('sensor.precipitation_next_hour') | float` without a
  default throw template errors while the sensor is `unknown` at startup~~ **Fixed** —
  found both live instances (`custom:mushroom-template-card`'s `primary` + `icon_color`
  templates, in both the `entrance-smart-switch` and `bedroom-smart-switch` storage
  dashboards — 4 template strings total; no other dashboard references this sensor).
  Changed `| float` → `| float(0)`: on `unknown`/`unavailable` the card now renders as
  if precipitation were 0mm (grey icon, no "(X mm)" line) instead of throwing — same
  fallback the card already used for a genuine 0mm forecast, so no new ambiguity.
  Doesn't affect the `rainfall_alert_*` automations (they use `numeric_state above: 0`,
  which already tolerates unknown/unavailable silently). Applied live via the Lovelace
  API (`ha_config_set_dashboard` python_transform); not part of this git repo
  (storage dashboards live in `.storage/`), so nothing to commit for this item.
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
