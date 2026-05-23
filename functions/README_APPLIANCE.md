# Function: Appliance Monitor

Monitors an appliance's power consumption to detect running, idle, and off states. Designed for "dumb" appliances where the only way to know if they're done is to watch the power draw — dishwashers, washing machines, dryers, coffee makers, 3D printers, etc.

> **Note:** This function is under active development. The core monitoring works, but refinements are ongoing.

## What It Does

- Detects appliance state based on power consumption thresholds
- **Binary mode**: Simple running/off (one threshold)
- **Staged mode**: Active/idle/off (two thresholds for multi-state appliances)
- **Cycle Complete** sensor fires when the appliance stops — trigger HA notifications
- Configurable button enable/disable (prevent accidental relay toggle on monitored appliances)
- All thresholds and modes adjustable from HA without reflashing

## Home Assistant Entities

| Entity | Type | Category | Description |
|--------|------|----------|-------------|
| **Appliance** | `switch` | — | Relay control (default: ALWAYS_ON) |
| **Running** | `binary_sensor` | — | True when power is above threshold |
| **Cycle Complete** | `binary_sensor` | — | Momentary pulse when Running goes false |
| **State** | `text_sensor` | — | "Running"/"Off" or "Active"/"Idle"/"Off" |
| **Monitor Mode** | `select` | Config | Binary or Staged |
| **Button** | `select` | Config | Enabled or Disabled |
| **Power Threshold** | `number` | Config | Single threshold for Binary mode (default: 50W) |
| **Threshold High** | `number` | Config | Upper threshold for Staged mode (default: 1000W) |
| **Threshold Low** | `number` | Config | Lower threshold for Staged mode (default: 50W) |

## Monitoring Modes

### Binary Mode (Default)

One threshold. Power above it = Running. Power below it = Off.

```
Power ≥ threshold → Running
Power < threshold → Off
```

Good for: dishwashers, washing machines, dryers, pool pumps — anything with a clear on/off power signature.

### Staged Mode

Two thresholds. Distinguishes between active use, idle/standby, and off.

```
Power ≥ threshold_high → Active
Power ≥ threshold_low  → Idle
Power < threshold_low  → Off

Running = Active OR Idle (power ≥ threshold_low)
```

Good for: coffee makers (brewing at 1000W → warming at 50W → off), 3D printers (heating bed at 300W → printing at 150W → idle at 5W), or any appliance with distinct power states.

## Cycle Complete

The **Cycle Complete** binary sensor is the key automation trigger. It fires a 1-second pulse when `Running` transitions from ON to OFF. Use it in HA automations:

```yaml
# Example HA automation
automation:
  - alias: "Dishwasher Done Notification"
    trigger:
      - platform: state
        entity_id: binary_sensor.dishwasher_cycle_complete
        to: "on"
    action:
      - service: notify.mobile_app
        data:
          message: "Dishwasher is done!"
```

## Setting Thresholds

The right threshold depends on your specific appliance. Here's how to find it:

1. Deploy with default thresholds
2. Run the appliance through a full cycle
3. Watch the power readings in HA (or ESPHome logs)
4. Set the threshold between the idle draw and the active draw

| Appliance | Typical Idle | Typical Active | Suggested Threshold |
|-----------|-------------|----------------|-------------------|
| Dishwasher | 0–2W | 200–1800W | 10W |
| Washing machine | 0–3W | 100–500W | 10W |
| Dryer | 0–2W | 2000–5000W | 50W |
| Coffee maker (staged) | 0W / 50W warming | 1000W brewing | Low: 20W, High: 500W |
| 3D printer (staged) | 5W idle | 150W printing / 300W heating | Low: 50W, High: 200W |

## Relay Behavior

Default restore mode: `ALWAYS_ON` — the relay stays on so the appliance is always powered and monitored. This is intentional: you're monitoring the appliance, not controlling it.

If you need the relay to start off, override it in your device config:

```yaml
switch:
  - id: !extend appliance
    restore_mode: RESTORE_DEFAULT_OFF
```

## Button Behavior

The physical button is controlled by the **Button** select entity:

- **Enabled** (default): Button press toggles the relay
- **Disabled**: Button press does nothing

Disable it for appliances where accidentally cutting power mid-cycle would be bad (dishwasher mid-fill, 3D printer mid-print).

## LED Behavior

The relay LED follows the **Running** sensor, not the relay state. This gives you a visual indicator of whether the appliance is actually doing something:

- LED on → appliance is consuming power above threshold
- LED off → appliance is idle or off

## Example

```yaml
substitutions:
  device_name: "dishwasher"
  device_description: "Dishwasher power monitor and cycle detector"
  friendly_name: "Dishwasher"

packages:
  base: !include common/esp_common.yaml
  hardware: !include hardware/sonoff_s31.yaml
  function: !include functions/miniplug_appliance.yaml
```

## Dependencies

| From | ID | Required |
|------|----|----------|
| Hardware | `relay_output` | Yes |
| Hardware | `relay_led` | Yes |
| Hardware | `power` | Yes |
| Common | `device_time` | No (not used) |
