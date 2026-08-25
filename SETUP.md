# Apartment Automation Setup

See `floorplan.jpg` for the visual layout.

## Rooms & Sensors

Layout top-to-bottom: Wohnzimmer → Durchgangszimmer + Kitchen → Entrance + Kammerl + Bathroom

| Room | Area | Sensors | Lights |
|------|------|---------|--------|
| **Wohnzimmer** (living/office) | 26 m² | praesenzsensor3 (FP1E mmWave) | wohnzimmerlampe, vintage_lampe, stehlampe, desklamp |
| **Durchgangszimmer** (pass-through) | 22 m² | praesenzsensor2 (mmWave), bewegungssensor2 (PIR, shared with entrance) | durchgangszimmer1, durchgangszimmer2, afrika_stehlampe |
| **Küche** (kitchen) | 11 m² | praesenzsensor (mmWave) | kuchenlampe, billy_tischleuchte |
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
| `daylight_lights_off.yaml` | All rooms (wohnzimmer skipped while held) | illuminance > 50 lux for 5 min |
| `daylight_lights_on.yaml` | Occupied rooms | illuminance < 30 lux for 2 min |
| `turn_off_everything_on_6h_no_motion.yaml` | All | praesenzsensor off for 6h |

## Shutdown Sequences

| Room | Day grace | Night grace | Notes |
|------|-----------|-------------|-------|
| Wohnzimmer | 15 min → dim 50% → 30s → off | 1 min → off | Skips dim if light already off. A manual brightness touch sets `input_boolean.wohnzimmer_keep_on`, which pauses reset/turn-off until the room goes fully dark or presence is off 2h |
| Kitchen | 4 min (2+2) → dim 50% → flash → off | 2 min → off | `aus` trigger has 2min `for:`, day adds 2min delay. Both kuchenlampe *and* billy_tischleuchte |
| Durchgangszimmer | 3 min (pass-through) / 7 min (presence) → off | 1 min → off | Template triggers: both sensors off for 3 min (pass-through) / 7 min (presence). Night handled by night_pathway_mode (praesenzsensor2 off 1 min). A manual brightness touch sets `input_boolean.durchgangszimmer_keep_on`, which pauses the day reset/turn-off until the room goes fully dark or presence is off 2h |
| Entrance | 3 min (60s + 120s) → off | 2 min → off | Day: bewegungssensor1_pathway_lights, requires both sensors clear. Night: night_pathway_mode |

## Known Pitfalls

- **`mode: restart` kills running shutdowns** — any trigger (even with failing conditions) cancels an in-progress shutdown sequence before conditions are evaluated
- **Template trigger `for:` vs action delay** — template triggers fire late (after the `for:` duration), so conditions are checked late. State triggers fire instantly and delay inside the action, so conditions are checked early. Template triggers are prone to time-boundary bugs (e.g., 23:00 crossing).
- **Pathway brightness conflicts** — pathway automations set the kitchen to 50% (day) / 20% (night), presence boosts it to 100% (60% at night). The gap is deliberately wide so walking in *feels* like a boost. Shutdown only cleans up pathway brightness (≤62%) to avoid overriding the presence automation's grace period — that one-sided threshold still covers both 50% and 20%, so leave it at 62 rather than re-tuning it to the pathway level.
- **bewegungssensor2 fires on all room-to-room traffic** — not used for entrance ON (too noisy), only for entrance OFF check and durchgangszimmer/night pathway triggers
- **Top-level conditions block ALL trigger handlers** — put illuminance/time checks inside choose branches, not at the top level
- **Manual-touch detection via `context.user_id`** — `wohnzimmer_manual_override` and `durchgangszimmer_manual_override` treat a light change as manual only when `trigger.to_state.context.user_id is not none`. Frontend/app/voice calls carry a user_id; automation `light.turn_on` calls do not, so our own presence/daylight actions never trip the hold. A physical Zigbee remote bound directly to the bulb would NOT carry a user_id (it'd be missed), but the Tuya remotes here only toggle, and brightness is set from the app.
- **`wohnzimmer_keep_on` must be checked everywhere wohnzimmer can be turned off** — `wohnzimmer_on_presence` (both presence-off branches, re-checked after the delay), `stuck_lights_cleanup` (fires at 10 min) AND `daylight_lights_off` (>50 lux, incl. the desklamp sub-branch). Miss one and the hold leaks. The hold is released when all 3 wohnzimmer lights go off (any cause) or presence is off 2h.
- **`durchgangszimmer_keep_on` must be checked everywhere the durchgangszimmer can be turned off or re-levelled** — `durchgangszimmer_on_motion` (the 100% turn-on plus both absence branches), `daylight_lights_on`, `stuck_lights_cleanup`, `bewegungssensor1_pathway_lights` (its motion_off branch, easy to miss — the file reads as an *entrance* automation) and `night_pathway_mode` (all four durchgangszimmer branches: both 20% turn-ons, or pathway mode dims a hand-set level down to 20%). `daylight_lights_off` never touches the durchgangszimmer at all. Released when both ceiling lamps are off, or presence is off 2h.
- **The durchgangszimmer release ignores `switch.afrika_stehlampe`** — deliberately, mirroring how the wohnzimmer release ignores `switch.desklamp`. Keep it that way: a stehlampe-inclusive release is fragile against any path that turns off area *lights* without area *switches*.
- **Both `*_keep_on` helpers MUST exist in HA** — a `condition: state ... state: "off"` against a missing entity evaluates false, which silently disables every gated branch (i.e. the room stops lighting up). Create them under Settings → Devices & Services → Helpers → Toggle.
- **`light.billy_tischleuchte` mirrors `light.kuchenlampe`** — it lives in the kitchen area (`kuche`) and is targeted by entity_id everywhere the kuchenlampe is (presence on/off, both pathway automations, night pathway, daylight-on, stuck-lights). Same levels: 100% day / 60% night, 50% day-pathway, 20% night-pathway, 50% dim warning. Area-targeted actions (`daylight_lights_off`, `script.bedtime_mode`, `script.turn_off_all`) pick it up automatically via the area. Add it to any *new* kitchen entity_id target too, or it gets stranded on.
- **The two kitchen lamps are staggered by 1s** — on: billy first, kuchenlampe 1s later; off: kuchenlampe first, billy 1s later (same idiom as durchgangszimmer1/2). Applies in `kueche_on_presence`, both pathway automations, `night_pathway_mode` and `daylight_lights_on`. `stuck_lights_cleanup` turns both off simultaneously on purpose — it's a safety net, not a lighting effect.
- **`billy_tischleuchte` gets no `color_temp_kelvin`** — the night pathway sets 2700K on the kuchenlampe but not on billy; its color-temp support was never confirmed and an unsupported attribute would break the staggered sequence.
- **Area ids are `kuche` (one 'e') and `entrance`** — both confirmed by the user, 2026-08-25. Do NOT "fix" either one. `kuche` was briefly changed to `kueche` and `entrance` was nearly changed to `eingang`, both on the strength of the MCP `GetLiveContext` output, which prints the kitchen as `kueche, Küche` and the entrance as `eingang, Vorhaus`. **Neither of those tokens is the area id** — that output is not an area-registry dump and must not be used to derive ids. Check Settings → Areas, or ask.
- **`switch.turn_off` sweeps over `kuche` also hit the Kitchen Adaptive Lighting switches** — the area has no switch-driven lamp, so `daylight_lights_off`, `script.turn_off_all` and `tuya_4button_remote` turning switches off there only ever touches the Adaptive Lighting controls. Long-standing behaviour, left as-is; worth knowing if AL for the kitchen ever seems to disable itself.
- **The `<= 62` pathway-brightness guard now takes the max of both kitchen lamps** — they are always set together, so `[kuchenlampe, billy]|max` is the pathway test. Keying it on one lamp would strand the other.
