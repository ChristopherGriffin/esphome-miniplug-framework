# Function: Lamp

The simplest function — turns a smart plug into a controllable light. The relay is exposed as a `light` entity in Home Assistant (not a `switch`), so it integrates naturally with HA's lighting controls, automations, and dashboards.

## What It Does

- Exposes the relay as a **Light** entity in Home Assistant
- Physical button toggles the light
- Relay LED follows light state (on/off)
- Restores to OFF after power loss

That's it. No monitoring, no scheduling, no thresholds. If you just need a plug that turns something on and off, this is the one.

> **Note:** `timed_light` is essentially this function with a schedule engine added on top. If you think you might want scheduling later, it's worth knowing the two are structurally identical — switching over is just a package swap plus adding your latitude/longitude. See [README_TIMED_LIGHT.md](README_TIMED_LIGHT.md).

## Home Assistant Entities

| Entity | Type | Description |
|--------|------|-------------|
| **Lamp** | `light` | Main on/off control |

## Why Light Instead of Switch?

Home Assistant treats `light` and `switch` entities differently in the UI and in automations. Using a `light` entity means:

- It shows up in the Lights section of dashboards
- Voice assistants handle "turn on the living room lamp" naturally
- It works with HA's light groups
- Scenes can include it alongside dimmable lights

If you'd prefer a `switch` entity instead, use the minimal template from the [functions README](README.md) as a starting point.

## Button Behavior

Single press toggles the light. No long-press or multi-press functionality — keep it simple.

## Restore Mode

Default: `RESTORE_DEFAULT_OFF` — if the plug loses power and comes back, the light stays off. This is the safe default for lamps (you don't want lights turning on at 3am after a power blip).

To change this, add to your device config:

```yaml
# Override restore mode - light comes back on after power loss
light:
  - id: !extend lamp
    restore_mode: RESTORE_DEFAULT_ON
```

## Example

```yaml
substitutions:
  device_name: "living-room-lamp"
  device_description: "Floor lamp next to the couch"
  friendly_name: "Living Room Lamp"

packages:
  base: !include common/esp_common.yaml
  hardware: !include hardware/switchbot_miniplug_W1901400.yaml
  function: !include functions/miniplug_lamp.yaml
```

## Dependencies

| From | ID | Required |
|------|----|----------|
| Hardware | `relay_output` | Yes |
| Hardware | `relay_led` | Yes |
| Hardware | `power` | No (not used) |
| Common | `device_time` | No (not used) |
