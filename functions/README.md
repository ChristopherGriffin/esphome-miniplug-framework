# Function Profiles

Function profiles define **what the plug does** — the relay behavior, automation logic, monitoring, and Home Assistant entities. They consume the hardware interface (`relay_output`, `relay_led`, `power`) and implement the `on_button_press` script.

**The key principle:** a function works on any supported hardware. Write it once, deploy on ESP8266 or ESP32.

## Available Functions

| Function | File | Complexity | Power Monitoring | Details |
|----------|------|-----------|-----------------|---------|
| Lamp | [`miniplug_lamp.yaml`](miniplug_lamp.yaml) | Simple | No | [README →](README_LAMP.md) |
| Heater | [`miniplug_heater.yaml`](miniplug_heater.yaml) | Medium | Yes | [README →](README_HEATER.md) |
| Timed Light | [`miniplug_timed_light.yaml`](miniplug_timed_light.yaml) | Advanced | No | [README →](README_TIMED_LIGHT.md) |
| Appliance Monitor | [`miniplug_appliance.yaml`](miniplug_appliance.yaml) | Medium | Yes | [README →](README_APPLIANCE.md) |

## Interface Contract

Every function package must:

1. **Consume** `relay_output` — the GPIO output for the relay
2. **Consume** `relay_led` — the internal light for relay status LED
3. **Implement** `on_button_press` — script called when the physical button is pressed
4. **Optionally consume** `power` — real-time power sensor (for monitoring functions)
5. **Optionally consume** `device_time` — synced time component (for scheduling functions)

## Creating Your Own

Minimal template:

```yaml
switch:
  - platform: output
    name: "My Switch"
    id: my_switch
    output: relay_output
    restore_mode: RESTORE_DEFAULT_OFF
    on_turn_on:
      - light.turn_on: relay_led
    on_turn_off:
      - light.turn_off: relay_led

script:
  - id: on_button_press
    then:
      - switch.toggle: my_switch
```

That's a complete function. The hardware provides the relay and LED, you just define what to do with them.
