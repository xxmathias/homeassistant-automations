# Apartment Automation Setup

See `floorplan.jpg` for the visual layout.

## Rooms & Sensors

Layout top-to-bottom: Wohnzimmer → Durchgangszimmer + Kitchen → Entrance + Kammerl + Bathroom

| Room | Area | Sensors | Lights |
|------|------|---------|--------|
| **Wohnzimmer** (living/office) | 26 m² | praesenzsensor3 (FP1E mmWave) | wohnzimmerlampe, vintage_lampe, stehlampe, desklamp |
| **Durchgangszimmer** (pass-through) | 22 m² | praesenzsensor2 (mmWave), bewegungssensor2 (PIR, shared with entrance) | durchgangszimmer1, durchgangszimmer2, afrika_stehlampe |
| **Küche** (kitchen) | 11 m² | praesenzsensor (mmWave) | kuchenlampe |
| **Eingang** (entrance) | 17 m² | bewegungssensor1 (PIR, near entrance/bathroom), bewegungssensor2 (PIR, shared with durchgangszimmer) | entrance area lights |
| **Bedroom** | 8.2 m² | — | — |
| **Bathroom** | — | humidity + temperature sensor only | — |
| **Kammerl** (storage) | — | — | — |

## Sensor Types

- **Presence sensors (mmWave):** Detect stationary presence (breathing, micro-movements). Used in rooms where people sit/stay.
  - praesenzsensor — kitchen
  - praesenzsensor2 — durchgangszimmer
  - praesenzsensor3 — wohnzimmer (Seeed FP1E, max sensitivity, 6m range configured but struggles with stationary detection beyond ~3m)
- **Motion sensors (PIR):** Detect movement only. Used in transit areas.
  - bewegungssensor1 — pathway area near entrance/bathroom
  - bewegungssensor2 — entrance (also covers durchgangszimmer entry, fires on all room-to-room traffic)

## Automation Files

| File | Room | Trigger |
|------|------|---------|
| `wohnzimmer_on_presence.yaml` | Wohnzimmer | praesenzsensor3 |
| `wohnzimmer_manual_override.yaml` | Wohnzimmer | wohnzimmer light state + praesenzsensor3 off 2h |
| `kueche_on_presence.yaml` | Kitchen | praesenzsensor |
| `durchgangszimmer_on_motion.yaml` | Durchgangszimmer | praesenzsensor2 + bewegungssensor2 (day only, night handled by night_pathway_mode) |
| `bewegungssensor1_pathway_lights.yaml` | Entrance + pathway | bewegungssensor1 (turns on entrance + durchgangszimmer + kitchen pathway lights) |
| `night_pathway_mode.yaml` | All pathway rooms | bewegungssensor2 + praesenzsensor2 (22:30-07:00 only) |
| `stuck_lights_cleanup.yaml` | All rooms | 15-min timer (any brightness) |
| `daylight_lights_off.yaml` | All rooms | illuminance > 50 lux for 5 min |
| `daylight_lights_on.yaml` | Occupied rooms | illuminance < 30 lux for 2 min |
| `turn_off_everything_on_6h_no_motion.yaml` | All | praesenzsensor off for 6h |

## Shutdown Sequences

| Room | Day grace | Night grace | Notes |
|------|-----------|-------------|-------|
| Wohnzimmer | 10 min → dim 50% → 30s → off | 3 min → off | Skips dim if light already off. A manual brightness touch sets `input_boolean.wohnzimmer_keep_on`, which pauses reset/turn-off until the room goes fully dark or presence is off 2h |
| Kitchen | 4 min (2+2) → dim 50% → flash → off | 2 min → off | `aus` trigger has 2min `for:`, day adds 2min delay |
| Durchgangszimmer | 10 min → off | — | Template trigger: both sensors off for 10 min. Night handled by night_pathway_mode |
| Entrance | 3 min (60s + 120s) → off | 2 min → off | Day: bewegungssensor1_pathway_lights, requires both sensors clear. Night: night_pathway_mode |

## Known Pitfalls

- **`mode: restart` kills running shutdowns** — any trigger (even with failing conditions) cancels an in-progress shutdown sequence before conditions are evaluated
- **Template trigger `for:` vs action delay** — template triggers fire late (after the `for:` duration), so conditions are checked late. State triggers fire instantly and delay inside the action, so conditions are checked early. Template triggers are prone to time-boundary bugs (e.g., 23:00 crossing).
- **Pathway brightness conflicts** — pathway automations set kitchen to 60%, presence sets 80%. Shutdown only cleans up pathway brightness (≤62%) to avoid overriding the presence automation's grace period.
- **bewegungssensor2 fires on all room-to-room traffic** — not used for entrance ON (too noisy), only for entrance OFF check and durchgangszimmer/night pathway triggers
- **Top-level conditions block ALL trigger handlers** — put illuminance/time checks inside choose branches, not at the top level
- **Manual-touch detection via `context.user_id`** — `wohnzimmer_manual_override` treats a light change as manual only when `trigger.to_state.context.user_id is not none`. Frontend/app/voice calls carry a user_id; automation `light.turn_on` calls do not, so our own presence/daylight actions never trip the hold. A physical Zigbee remote bound directly to the bulb would NOT carry a user_id (it'd be missed), but the Tuya remotes here only toggle, and brightness is set from the app.
- **`wohnzimmer_keep_on` must be checked everywhere wohnzimmer can be turned off** — `wohnzimmer_on_presence` (both presence-off branches, re-checked after the delay) AND `stuck_lights_cleanup` (fires at 10 min). Miss one and the hold leaks. The hold is released when all 3 wohnzimmer lights go off (any cause) or presence is off 2h.
