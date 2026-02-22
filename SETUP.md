# Apartment Automation Setup

See `floorplan.jpg` for the visual layout.

## Rooms & Sensors

Layout top-to-bottom: Wohnzimmer → Durchgangszimmer + Kitchen → Entrance + Kammerl + Bathroom

| Room | Area | Sensors | Lights |
|------|------|---------|--------|
| **Wohnzimmer** (living/office) | 26 m² | praesenzsensor3 (FP1E mmWave) | wohnzimmerlampe, vintage_lampe, stehlampe, desklamp |
| **Durchgangszimmer** (pass-through) | 22 m² | praesenzsensor2 (mmWave), bewegungssensor2 (PIR, shared with entrance) | durchgangszimmer1, durchgangszimmer2, afrika_stehlampe |
| **Küche** (kitchen) | 11 m² | praesenzsensor (mmWave) | kuchenlampe |
| **Eingang** (entrance) | 17 m² | bewegungssensor2 (PIR, shared with durchgangszimmer) | entrance area lights |
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
  - bewegungssensor2 — entrance (also covers durchgangszimmer entry)

## Automation Files

| File | Room | Trigger |
|------|------|---------|
| `wohnzimmer_on_presence.yaml` | Wohnzimmer | praesenzsensor3 |
| `kueche_on_presence.yaml` | Kitchen | praesenzsensor |
| `durchgangszimmer_on_motion.yaml` | Durchgangszimmer | praesenzsensor2 + bewegungssensor2 (template trigger) |
| `eingang_on_motion.yaml` | Entrance | bewegungssensor2 |
| `bewegungssensor1_pathway_lights.yaml` | Pathway | bewegungssensor1 (turns on entrance + durchgangszimmer + kitchen) |
| `night_pathway_mode.yaml` | All pathway rooms | bewegungssensor2 + praesenzsensor2 (23:00-07:00 only) |
| `stuck_lights_cleanup.yaml` | All rooms | 15-min timer (catches lights stuck at 0-60%) |
| `turn_off_everything_on_6h_no_motion.yaml` | All | praesenzsensor off for 6h |

## Shutdown Sequences

All rooms follow the same pattern: grace period → dim to 50% → flash warning → turn off.

| Room | Day grace | Night grace | Notes |
|------|-----------|-------------|-------|
| Wohnzimmer | 4 min | 2 min | Night: skips turn-off if extra lights manually on |
| Kitchen | 4 min (2+2) | 2 min | `aus` trigger has 2min `for:`, day adds 2min action delay |
| Durchgangszimmer | 4 min | — | Template trigger: both sensors off for 4 min. Night handled by night_pathway_mode |
| Entrance | 4 min | — | Night handled by night_pathway_mode (2 min) |

## Known Pitfalls

- **`mode: restart` kills running shutdowns** — any trigger (even with failing conditions) cancels an in-progress shutdown sequence before conditions are evaluated
- **Template trigger `for:` vs action delay** — template triggers fire late (after the `for:` duration), so conditions are checked late. State triggers fire instantly and delay inside the action, so conditions are checked early. Template triggers are prone to time-boundary bugs (e.g., 23:00 crossing).
- **Pathway automations turn on lights in other rooms but don't turn them off** — they rely on the room's own automation or stuck_lights_cleanup
- **stuck_lights_cleanup only catches 0-60% brightness** — lights at 100% won't be caught (only the 6h timeout)
- **bewegungssensor2 is shared** between entrance and durchgangszimmer automations
