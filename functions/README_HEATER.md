# Function: Heater Control

An autonomous heater controller with freeze protection and power fault detection. Designed for space heaters, pipe heat tape, well house heaters, and any resistive heating element where you need reliable cold-weather automation with safety monitoring.

## What It Does

- **Freeze protection**: Automatically turns on when Home Assistant's `freeze_alert` is active, turns off when cleared
- **Overcurrent detection**: Immediately kills the relay if power exceeds the max threshold, latches the fault until manually reset
- **Undercurrent detection**: Flags when the relay is on but power is too low (burned-out element, disconnected heater)
- **Physical button**: Toggles heater and clears overcurrent faults (works without HA access)
- **Thermostat-friendly**: Power monitoring can be disabled for heaters with built-in thermostats that cycle on/off

## Home Assistant Entities

| Entity | Type | Category | Description |
|--------|------|----------|-------------|
| **Heater** | `switch` | — | Main relay control |
| **Power Monitoring** | `switch` | Config | Enable/disable fault detection |
| **Freeze Alert** | `binary_sensor` | — | Imported from HA (`input_boolean.freeze_alert`) |
| **Heating** | `binary_sensor` | — | True when power draw confirms active heating |
| **Undercurrent** | `binary_sensor` | — | Relay on but power below minimum (30s delay) |
| **Overcurrent** | `binary_sensor` | — | **Latches** — power exceeded max, relay killed |
| **Power Min Threshold** | `number` | Config | Watts below which = undercurrent (default: 100W) |
| **Power Max Threshold** | `number` | Config | Watts above which = overcurrent (default: 1500W) |
| **Reset Overcurrent** | `button` | Config | Clear the latched overcurrent fault |

## Prerequisites

Create this entity in Home Assistant before deploying:

```yaml
# In your HA configuration.yaml or via UI
input_boolean:
  freeze_alert:
    name: Freeze Alert
    icon: mdi:snowflake-alert
```

Then automate it based on weather forecast, outdoor temperature sensor, or whatever makes sense for your setup. When this boolean turns on, the heater turns on. When it turns off, the heater turns off.

## Freeze Protection Behavior

```
freeze_alert ON  → heater ON  (immediate)
freeze_alert OFF → heater OFF (immediate)

Manual button toggle → works, but freeze protection
                       re-enables within 30 seconds
                       if freeze_alert is still active
```

The 30-second enforcement loop means you can briefly toggle off for maintenance, but the system won't let you forget the heater is supposed to be on during a freeze.

## Power Monitoring

### Overcurrent (Safety-Critical)

Checked every **2 seconds**. If power exceeds `power_threshold_max`:

1. Relay immediately turns off
2. `overcurrent_latched` global sets to true
3. Overcurrent binary sensor turns on and **stays on**
4. Fault must be manually cleared via the Reset button or physical button

The latch exists so you can see that an overcurrent event happened even if you weren't watching. It's a "something is wrong, investigate before re-enabling" signal.

### Undercurrent (Diagnostic)

Checked every **30 seconds** with a 30-second `delayed_on` filter. If the relay is on but power is below `power_threshold_min`:

- Binary sensor turns on
- Warning logged

The 30-second delay prevents false alarms during heater startup (inrush current settling, thermostat cycling, etc.). Undercurrent doesn't kill the relay — it just flags that the heater might not be working.

### Disabling Power Monitoring

Some heaters have built-in thermostats that cycle the element on and off. This causes constant undercurrent alerts during the off-cycle. Flip the **Power Monitoring** switch off in HA to disable both overcurrent and undercurrent detection.

When power monitoring is disabled, the **Heating** binary sensor assumes the heater is heating whenever the relay is on.

## Setting Thresholds

| Heater Type | Min Threshold | Max Threshold | Notes |
|------------|--------------|---------------|-------|
| 1500W space heater | 100W | 1600W | Standard US portable heater |
| 500W pipe heat tape | 50W | 600W | Lower draw, tighter range |
| 250W well house heater | 25W | 350W | Small enclosed heater |
| Thermostat-controlled | — | — | Disable power monitoring |

Set thresholds from the HA UI after deploying — no reflashing needed.

## Button Behavior

Single press:
1. If overcurrent fault is latched → clears the fault
2. Toggles the heater relay

This means a physical button press always does something useful — either clears a fault or toggles power. Important for situations where HA is unreachable.

## Restore Mode

Default: `ALWAYS_OFF` — heaters start off after power loss. The freeze protection loop will turn it back on within 30 seconds if the freeze alert is active. This is the safe default (you never want a heater turning on unexpectedly after a power restoration).

## Example

```yaml
substitutions:
  device_name: "well-heater"
  device_description: "Well house freeze protection heater"
  friendly_name: "Well Heater"

packages:
  base: !include common/esp_common.yaml
  hardware: !include hardware/switchbot_miniplug_W1901400.yaml
  function: !include functions/miniplug_heater.yaml
```

## Dependencies

| From | ID | Required |
|------|----|----------|
| Hardware | `relay_output` | Yes |
| Hardware | `relay_led` | Yes |
| Hardware | `power` | Yes |
| Common | `device_time` | No |
| Home Assistant | `input_boolean.freeze_alert` | Yes |
