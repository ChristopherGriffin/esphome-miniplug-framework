# Battery Backup Addon

Combines two addon packages to provide full battery backup monitoring when a JBD BMS is connected via BLE to a SwitchBot Mini Plug.

## Packages

| File | Purpose |
|------|---------|
| `jbd_bms_ble.yaml` | BLE connection to JBD/Xiaoxiang BMS — exposes battery voltage, current, SOC, cell voltages, etc. |
| `battery_power_analysis.yaml` | Template sensors that combine AC outlet power with BMS data to estimate load, runtime, and power source |

## Sensors Overview

### From `jbd_bms_ble.yaml`
- **Net Power** — Net battery power (W). Positive = charging, negative = discharging.
- **Charging Power / Discharging Power** — One-directional power flow sensors
- **State of Charge, Capacity Remaining, Total Voltage, Cell Voltages**, etc.
- **Charging switch** (`${bms_id}_charging`) — Exposed with internal ID for use by the battery backup function's exercise mode
- **Discharging switch** (`${bms_id}_discharging`) — Exposed with internal ID for use by the battery backup function's exercise mode

### From `battery_power_analysis.yaml`
- **Outlet Power** *(from hardware)* — Raw AC watts at the plug
- **Battery Load Power** — Load power when running on battery (NAN when on AC)
- **Total Load** — Actual load regardless of power source = `Outlet Power - Charging Power + Discharging Power`
- **Remaining Runtime** — Hours of battery remaining (uses actual discharge current on battery, or projects from current AC load if on AC)
- **Runtime Source** — Text: `Battery (actual)` or `AC (estimated)`

## Notes

- `Power from AC` and `Outlet Power` will show the same value whenever the battery is idle (fully charged, not charging). This is correct — when no charging overhead exists, all outlet power is load power.
- The `battery_power_analysis` addon requires the hardware package's `power` sensor ID and the BMS addon's `${bms_id}_charging_power`, `${bms_id}_discharging_power`, etc.
- Both addons must use the same `bms_id` value.

## Example Device Config

```yaml
substitutions:
  device_name: "battery-monitor"
  friendly_name: "Battery Monitor"
  name_no_dashes: "battery_monitor"
  device_description: "SwitchBot Mini Plug + JBD BMS battery backup monitor"

external_components:
  - source: github://syssi/esphome-jbd-bms@main

packages:
  common:    !include esphome-miniplug-framework/common/esp_common.yaml
  hardware:  !include esphome-miniplug-framework/hardware/switchbot_miniplug_W1901400.yaml
  function:  !include esphome-miniplug-framework/functions/miniplug_monitor.yaml
  bms0: !include
    file: esphome-miniplug-framework/addon/jbd_bms_ble.yaml
    vars:
      bms_id: "bms0"
      bms_name: "Battery"
      bms_mac_address: "A4:C1:37:36:9A:2D"
  battery0: !include
    file: esphome-miniplug-framework/addon/battery_power_analysis.yaml
    vars:
      bms_id: "bms0"
      bms_name: "Battery"
      max_battery_current: "100"
      inverter_efficiency: "0.85"
```
