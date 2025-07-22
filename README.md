# Home Assistant Automations

Smart home automations for adaptive lighting and presence detection.

## Home Layout
```
Entrance → Kitchen / Bathroom / Durchgangszimmer → Livingroom → Bedroom
```

## Areas
- **bathroom**: Bathroom area
- **bedroom**: Bedroom area  
- **durchgangszimmer**: Pass-through room
- **entrance**: Entrance area
- **kitchen**: Kitchen area
- **livingroom**: Living room area

## Devices

### Sensors
- **bewegungssensor1**: Motion sensor (pathway/bathroom area)
- **bewegungssensor2**: Motion sensor (entrance area)  
- **praesenzsensor**: Presence sensor (kitchen)
- **praesenzsensor2**: Presence sensor (durchgangszimmer)

### Lights
- **Entrance**: Area lights
- **Kitchen**: `light.kuchenlampe`
- **Durchgangszimmer**: `light.durchgangszimmer1`, `light.durchgangszimmer2`
- **Livingroom**: `light.wohnzimmerlampe`
- **Bedroom**: Area lights
- **Yamaha Boxes**: `switch.box_left`, `switch.box_right`

## Automations

### Motion-Activated Lighting
- **entrance_on_motion**: Turns on entrance + durchgangszimmer pathway lighting. Dims after 1:30, off after 2:00
- **kitchen_on_presence**: Full brightness on presence, waits 45s → dims to 60% → 15s → off (1min total)
- **kitchen_pathway_timer**: Auto-off kitchen pathway light after 30s when no presence
- **durchgangszimmer_on_motion**: Adaptive brightness with 15% boost on presence. Waits 30s before dimming
- **livingroom_on_motion**: Motion-activated during low light (10:00-23:00). No brightness control
- **bewegungssensor1_pathway_lights**: Activates entrance, kitchen, and durchgangszimmer as pathway

### Special Modes
- **night_bathroom_mode**: Minimal 15% lighting for bathroom trips (22:00-06:00)
- **bedtime_mode**: Script to turn off all lights except bedroom
- **toggle_yamaha_boxes**: Script to control both speakers together

### Energy Saving
- **turn_off_everything_on_6h_no_motion**: Turns off all lights after 4 hours of no motion

### Remote Control
- **tuya_button_switch**: 4-button Zigbee remote for manual light control

## Key Features
- Adaptive brightness: 100% day / 30% night (23:00-07:00)
- Pathway lighting between rooms
- Presence-aware turn-off with grace periods (no dimming while present)
- All transitions use 1-2 second fades
- Individual room control with no conflicts
