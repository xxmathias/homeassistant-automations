# Smart Home Lighting Improvements

## Changes Made

### 1. Adaptive Lighting Scripts
Created helper scripts in `/scripts/` directory:
- **adaptive_brightness.yaml**: Returns brightness based on time of day and mode (normal/night/bathroom)
- **adaptive_color_temp.yaml**: Returns color temperature in Kelvin (5000K morning → 2700K night)
- **room_state_tracker.yaml**: Tracks room occupancy to prevent unnecessary cross-room activation

### 2. Separated Room Automations
- **entrance_on_motion.yaml**: Independent entrance lighting with adaptive features
- **durchgangszimmer_on_motion.yaml**: Improved logic that doesn't unnecessarily trigger other rooms
- **kitchen_on_presence**: Updated with adaptive lighting and adjacent room checks
- **livingroom_on_motion**: Added adaptive brightness and color temperature

### 3. Night Mode
- **night_bathroom_mode.yaml**: Special automation for nighttime bathroom trips
  - Only activates between 22:00-06:00 when living room is off
  - Uses minimal 15% brightness with warm 2700K light
  - Only turns on one durchgangszimmer light to minimize disturbance

## Key Improvements

### Problem 1: Unnecessary Cross-Room Activation
**Solution**: Each room now checks if adjacent rooms are occupied before turning off lights. The durchgangszimmer automation only triggers kitchen lights briefly as pathway lighting when no presence is detected.

### Problem 2: Bright Lights During Night Bathroom Trips
**Solution**: Night bathroom mode provides minimal warm lighting when all other lights are off between 22:00-06:00.

### Problem 3: No Adaptive Lighting
**Solution**: All automations now use adaptive brightness and color temperature:
- Morning (06:00-09:00): Bright cool white (90-100%, 5000K)
- Day (09:00-18:00): Full brightness neutral white (100%, 4000K)
- Evening (18:00-22:00): Dimmed warm white (70%, 3000K)
- Night (22:00-06:00): Very dim warm (30%, 2700K)

## Next Steps

1. **Disable the old combined automation**: 
   - The `entrance_kitchen_durchgangszimmer_on_motion` file should be disabled or removed
   
2. **Create input_boolean helpers** (if not already present):
   - `input_boolean.entrance_occupied`
   - `input_boolean.kitchen_occupied`
   - `input_boolean.durchgangszimmer_occupied`

3. **Test the automations**:
   - Test normal movement patterns during day/evening
   - Test night bathroom trips
   - Verify lights don't turn on unnecessarily when sitting in durchgangszimmer

4. **Fine-tune parameters**:
   - Adjust brightness percentages in adaptive_brightness.yaml
   - Modify timeout durations if needed
   - Tweak color temperatures to preference