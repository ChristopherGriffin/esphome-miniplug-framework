# Function: Timed Light

A scheduled light controller with flexible time source options. Supports sunrise/sunset-aware scheduling, fixed times, duration-based off times, and configurable offsets — all adjustable from Home Assistant without reflashing.

> **Note:** This function is structurally identical to `lamp` — it exposes the same `light` entity with the same button behavior. The only addition is the schedule engine and its supporting entities. If you don't need scheduling, [README_LAMP.md](README_LAMP.md) is the simpler starting point.

## What It Does

- Exposes the relay as a **Light** entity for manual control
- **Schedule engine** runs on the plug itself (not in HA) — survives HA restarts
- ON and OFF times can independently use: Fixed Time, Sunrise, Sunset, or Duration
- Minute-level offsets for sunrise/sunset (e.g., "1 hour before sunrise")
- Schedule can be enabled/disabled from HA (manual-only mode when disabled)
- Physical button toggles the lamp; **hold 3 seconds to toggle the schedule on/off** (blue LED confirms with 3 flashes = enabled, 1 long pulse = disabled)

## Home Assistant Entities

| Entity | Type | Category | Description |
|--------|------|----------|-------------|
| **Lamp** | `light` | — | Main on/off control |
| **Schedule Enabled** | `switch` | Config | Enable/disable schedule enforcement |
| **On Time Source** | `select` | — | Fixed Time, Sunrise, or Sunset |
| **Off Time Source** | `select` | — | Fixed Time, Sunrise, Sunset, or Duration |
| **Fixed On Time** | `datetime` | — | HH:MM picker (used when source = Fixed Time) |
| **Fixed Off Time** | `datetime` | — | HH:MM picker (used when source = Fixed Time) |
| **Duration Hours** | `number` | Config | Hours after ON to turn OFF (1–24, 0.5 steps) |
| **Sunrise Offset Minutes** | `number` | Config | Offset when Sunrise is selected (−120 to +120) |
| **Sunset Offset Minutes** | `number` | Config | Offset when Sunset is selected (−120 to +120) |
| **Actual On Time** | `text_sensor` | — | Calculated ON time (updates every minute) |
| **Actual Off Time** | `text_sensor` | — | Calculated OFF time (updates every minute) |

## Required Substitutions

```yaml
substitutions:
  latitude: "33.4484"     # Your latitude
  longitude: "-112.0740"  # Your longitude (negative = western hemisphere)
```

The plug needs your location to calculate sunrise and sunset times. These are compiled into the firmware — they don't change at runtime.

## Schedule Examples

All configured from the HA UI after flashing:

### 1. On at sunrise, off 12 hours later (default)
| Setting | Value |
|---------|-------|
| On Source | Sunrise |
| Off Source | Duration |
| Duration | 12h |

### 2. On at sunrise, off at sunset
| Setting | Value |
|---------|-------|
| On Source | Sunrise |
| Off Source | Sunset |

### 3. Night cycle (on at sunset, off at sunrise)
| Setting | Value |
|---------|-------|
| On Source | Sunset |
| Off Source | Sunrise |

The schedule engine handles overnight wrapping automatically.

### 4. On at 6am, off at 10pm
| Setting | Value |
|---------|-------|
| On Source | Fixed Time |
| Fixed On Time | 06:00 |
| Off Source | Fixed Time |
| Fixed Off Time | 22:00 |

### 5. On 1 hour before sunrise, off 2 hours after sunset
| Setting | Value |
|---------|-------|
| On Source | Sunrise |
| Sunrise Offset | −60 min |
| Off Source | Sunset |
| Sunset Offset | +120 min |

### 6. Mixed: Fixed on time, sun-based off time
| Setting | Value |
|---------|-------|
| On Source | Fixed Time |
| Fixed On Time | 08:00 |
| Off Source | Sunset |
| Sunset Offset | −30 min |

## How Offsets Work

Offsets apply **only when the corresponding sun event is the selected source**:

- **Sunrise Offset**: Applied wherever "Sunrise" is selected (ON or OFF source)
- **Sunset Offset**: Applied wherever "Sunset" is selected (ON or OFF source)
- Fixed Time and Duration sources ignore offsets

Negative = before the event, Positive = after the event.

## Schedule Enforcement

The schedule checks every **1 minute**:

1. Calculates current ON and OFF times based on selected sources + offsets
2. Updates the Actual On/Off Time text sensors (always, even if schedule disabled)
3. If `schedule_enabled` is ON: turns lamp on or off to match the calculated window
4. If `schedule_enabled` is OFF: skips step 3 (lamp is fully manual)

Manual overrides (button press or HA toggle) work until the next scheduled minute check. If the schedule says the lamp should be on and you turn it off, it'll come back on within 60 seconds. Disable the schedule for true manual control.

## Boot Behavior

On boot, the schedule engine waits **5 seconds** for time sync (from HA), then runs the first schedule check. If time isn't available yet, it logs a warning and retries on the next 1-minute interval.

Default restore mode: `RESTORE_DEFAULT_OFF` — the lamp starts off and the schedule turns it on at the right time.

## Button Behavior

| Action | Result |
|--------|--------|
| Short press (tap) | Toggle lamp on/off |
| Hold 3 seconds | Toggle Schedule Enabled on/off + blue LED confirmation |

A short press always works regardless of schedule state — useful for quick testing or when HA is unreachable. The schedule will re-assert at the next check if enabled.

Holding for 3 seconds is the physical equivalent of toggling the **Schedule Enabled** switch in HA. The blue LED flashes to confirm the new state:

- **3 quick flashes** — schedule is now **enabled**
- **1 long pulse** — schedule is now **disabled**

When the schedule is disabled, the lamp is fully manual and the schedule engine still calculates times (visible in the Actual On/Off Time sensors) but never acts on them.

## Example

```yaml
substitutions:
  device_name: "porch-light"
  device_description: "Front porch light on sunset schedule"
  friendly_name: "Porch Light"
  name_no_dashes: "porch_light"
  latitude: "33.4484"
  longitude: "-112.0740"

packages:
  base: !include common/esp_common.yaml
  hardware: !include hardware/switchbot_miniplug_W1901400.yaml
  function: !include functions/miniplug_timed_light.yaml
```

## Dependencies

| From | ID | Required |
|------|----|----------|
| Hardware | `relay_output` | Yes |
| Hardware | `relay_led` | Yes |
| Hardware | `power` | No (not used) |
| Common | `device_time` | Yes (schedule calculations) |
