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

## Execution Behavior
- Runs every 5 seconds using `time_pattern: seconds: "/5"`
- Uses `mode: restart` and `max_exceeded: silent` so only the latest cycle is applied
- Uses `export_mode_helper` to persist the current export mode when SOC is between `soc_low` and `soc_high`

## Decision Priorities
1. Invalid or unavailable sensors → set inverter to `min_power`
2. `soc <= soc_min` → protect battery by forcing `min_power`
3. Night mode (`pv < pv_threshold_night`) → adjust inverter limit for zero battery export
4. Export mode active → allow export and use PV as the inverter limit when house load is present
5. Charge priority → adjust inverter limit to balance PV, household load, and battery charging
6. Default fallback → set `min_power`

## Common Pitfalls (Avoid)
- **Do not remove hysteresis:** Separate `soc_high` and `soc_low` thresholds prevent mode oscillation.
- **Do not hardcode entity IDs:** Use blueprint inputs for all entities.
- **Do not assume `grid_bias` is added:** The blueprint subtracts `grid_bias` from `grid_power`.
- **Do not ignore `soc_min`:** It is the battery protection cutoff.
