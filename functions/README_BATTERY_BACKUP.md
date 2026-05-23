# Function: Battery Backup (AC Disconnect)

Configures the plug as a controllable AC disconnect switch for battery backup systems. The relay sits between grid power and a charger/inverter and defaults ON so the system stays powered under normal conditions. The plug watches a **lightning alert** signal from Home Assistant and drives the relay itself — no HA automations required. It also monitors the battery via BMS addons.

Includes an on-device **Battery Exercise Mode** that cycles a LiFePO4 battery between configurable SOC thresholds to prevent the cell degradation that comes from always sitting at full charge or reaching the BMS hard cutoff.

## What It Does

- Exposes the relay as an **AC Power** switch (on = grid connected, off = isolated)
- Watches `input_boolean.lightning_alert` from Home Assistant — turns relay OFF when lightning is detected, back ON when it clears. Reacts on every HA state update, so reboots and reconnects are handled immediately without a polling interval
- Physical button toggles the relay as a manual override
- **Battery Exercise Mode** — when enabled, manages the BMS charging and discharging switches autonomously on the plug (no HA required at runtime):
  - Stops charging once SOC reaches the high threshold
  - Resumes charging once SOC drops to the low threshold
  - Stops discharging if SOC falls to the discharge floor — a software-level cutoff above the BMS hard protection to prevent the over-discharge damage that can leave LiFePO4 cells needing a kickstart to recover
  - Re-enables discharging once SOC recovers back to the low threshold

## Home Assistant Entities

| Entity | Type | Category | Description |
|--------|------|----------|-------------|
| **AC Power** | `switch` | — | Main relay control. On = AC connected to charger/inverter |
| **Lightning Alert** | `binary_sensor` | — | Imported from HA `input_boolean.lightning_alert` — drives relay automatically |
| **Battery Exercise Mode** | `switch` | — | Enables automatic SOC cycling for LiFePO4 health |
| **Battery Exercise High SOC** | `number` | Config | Charging stops at this SOC % (default 90, range 50–100, step 5) |
| **Battery Exercise Low SOC** | `number` | Config | Charging resumes at this SOC % (default 20, range 5–50, step 5) |
| **Battery Exercise Discharge Floor** | `number` | Config | Discharging stops at this SOC % (default 12, range 5–25, step 1) |

All number entities persist across reboots and are adjustable from HA without reflashing.

## Exercise Mode Behavior

```
SOC rises to High SOC (90%)  → charging OFF
SOC falls to Low SOC (20%)   → charging ON
SOC falls to Floor (12%)     → discharging OFF  ← prevents BMS hard cutoff
SOC rises back to Low (20%)  → discharging ON
```

The discharge floor is intentionally set a percent or two above the BMS hardware protection threshold. When a LiFePO4 cell reaches the BMS hard cutoff it can enter a state that requires an external kickstart to recover. The floor prevents that from ever happening.

The check runs every 30 seconds on-device. This is sufficient — LiFePO4 cells change SOC slowly enough that 30s latency has no practical effect on the cycle boundaries.

Exercise mode only manages the BMS charging/discharging switches. It does not touch the AC disconnect relay.

## Lightning Alert Behavior

The plug imports `input_boolean.lightning_alert` from Home Assistant and reacts on every state update (`on_state`):

```
lightning_alert ON  → AC Power OFF  (isolate from grid)
lightning_alert OFF → AC Power ON   (reconnect to grid)
```

This fires on every value received from HA — not just transitions — so if the plug reboots or the API reconnects while lightning is active, it immediately asserts the correct relay state without needing a polling interval.

Set `input_boolean.lightning_alert` in HA however you like (lightning detector, manual dashboard toggle, automation) — the plug handles the rest.

## Button Behavior

| Action | Result |
|--------|--------|
| Short press (tap) | Toggle AC Power relay |
| Hold 3 seconds | No action (reserved) |

## Dependencies

| From | ID | Required |
|------|----|----------|
| Hardware | `relay_output` | Yes |
| Hardware | `relay_led` | Yes |
| Home Assistant | `input_boolean.lightning_alert` | Yes (lightning protection) |
| BMS addon | `${exercise_bms_id}_state_of_charge` | Yes (exercise mode) |
| BMS addon | `${exercise_bms_id}_charging` | Yes (exercise mode) |
| BMS addon | `${exercise_bms_id}_discharging` | Yes (exercise mode) |

The BMS dependency is only active when Battery Exercise Mode is switched on. The relay and button work regardless. If the HA API is unavailable, the relay holds its last known state.

## Optional Substitutions

| Key | Default | Description |
|-----|---------|-------------|
| `exercise_bms_id` | `bms0` | BMS instance ID to watch for exercise mode |

## Example Device Config

```yaml
substitutions:
  device_name: "lightning-switch-tower"
  friendly_name: "Lightning Switch Tower"
  device_description: "Lightning Tower UPS - SwitchBot Mini Plug + JBD BMS"
  exercise_bms_id: "bms0"   # optional, bms0 is the default

external_components:
  - source: github://syssi/esphome-jbd-bms@main

packages:
  base:     !include common/esp_common.yaml
  hardware: !include hardware/switchbot_miniplug_W1901400.yaml
  function: !include functions/miniplug_battery_backup.yaml
  bms0: !include
    file: addon/jbd_bms_ble.yaml
    vars:
      bms_id: "bms0"
      bms_name: "Battery"
      bms_mac_address: "A5:C2:37:37:D0:16"
  battery0: !include
    file: addon/battery_power_analysis.yaml
    vars:
      bms_id: "bms0"
      bms_name: "Battery"
      max_battery_current: "100"
      inverter_efficiency: "0.85"
```
