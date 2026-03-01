# SolarFlow HA Blueprint – AI Coding Instructions

## Project Overview
This is a Home Assistant automation blueprint for PV + battery energy management. It controls a SolarFlow 800 Pro inverter by managing power limits based on:
- **Grid power flow** (to achieve grid-neutral operation)
- **Battery SOC** (State of Charge) for mode transitions
- **PV generation** for night mode detection

## Architecture & Logic Flow

### Three Operating Modes (Priority-Based)
1. **Nachtmodus (Night Mode):** When PV < threshold (15W default), discharge battery to zero-feed (Nulleinspeisung). Uses grid power + grid_bias to calculate dynamic limit adjustments.
2. **Export Mode:** When SOC ≥ `soc_high` (95%), maximize power output (`max_power`).
3. **Ladepriorität (Charge Priority):** When SOC ≤ `soc_low` (80%), prioritize battery charging using grid/PV surplus.

### Mode Transition Logic (Hysteresis)
- **Export → Charging:** Triggered at SOC ≤ `soc_low` (prevents oscillation)
- **Charging → Export:** Triggered at SOC ≥ `soc_high`
- **Night Mode Guard:** Only activates if SOC > `soc_min` (prevents over-discharge)

### Power Adjustment Algorithm
Grid-following inverter logic:
```
delta = grid_power + grid_bias
adjusted_delta = clamp(delta, -ramp_limit, +ramp_limit)
new_limit = clamp(current_limit + adjusted_delta, min_power, max_power)
```
**Key:** Ramp limiting prevents sudden load changes; grid_bias fine-tunes zero-cross operation.

## Critical Input Parameters
- `grid_power_sensor`: 0 = balanced; negative = exporting to grid
- `battery_soc_sensor`: Percentage (triggers mode transitions at thresholds)
- `pv_power_sensor`: Wattage (night mode activates when < `pv_threshold_night`)
- `inverter_limit_entity`: Number entity (directly controlled by automation)
- `soc_high` / `soc_low`: Hysteresis thresholds (default 95% / 80%)
- `tolerance`: Ignores small grid swings < this value

## Modification Guidelines
- **Adjust soc_high/soc_low:** Only change if adding different battery profiles (don't add new modes without understanding hysteresis risks)
- **Tweak power ramp:** Increment carefully—high values cause oscillations; low values prevent responsiveness
- **Grid bias tuning:** Increase for under-frequency support; rarely needs change
- **PV threshold:** Project-specific; only adjust if local solar drops earlier/later than expected

## Testing & Validation
- Trigger runs every 5 seconds (`time_pattern: seconds: "/5"`) → quick feedback
- Blueprint restarts on trigger (`mode: restart`; `max_exceeded: silent`) → only latest calculation applies
- Manual validation: Set `grid_bias` to 0, trigger manually via Developer Tools, watch inverter limit entity response

## Common Pitfalls (Avoid)
- **Don't remove hysteresis:** Export/charging modes will oscillate if using single SOC threshold
- **Don't add logic outside choose blocks:** This breaks the mode prioritization
- **Don't forget grid_power sign convention:** Negative = house exporting (common confusion point)
- **Avoid hardcoding entity IDs:** Always use input variables; blueprints must be reusable
