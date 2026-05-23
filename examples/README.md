# Examples

Complete, ready-to-use device configs. Copy one, change the substitutions, compile, and flash.

Each example is ~15 lines — that's the whole point of the three-layer system.

| Example | Hardware | Function | File |
|---------|----------|----------|------|
| Lamp (SwitchBot) | ESP32-C3 Mini Plug | Simple on/off light | [`lamp_switchbot.yaml`](lamp_switchbot.yaml) |
| Lamp (Sonoff) | ESP8266 S31 | Simple on/off light | [`lamp_sonoff.yaml`](lamp_sonoff.yaml) |
| Heater | SwitchBot Mini Plug | Freeze protection + fault monitoring | [`heater_switchbot.yaml`](heater_switchbot.yaml) |
| Timed Light | SwitchBot Mini Plug | Sunrise/sunset scheduling | [`timed_light_switchbot.yaml`](timed_light_switchbot.yaml) |
| Appliance Monitor | Sonoff S31 | Power-based state detection | [`appliance_sonoff.yaml`](appliance_sonoff.yaml) |

## Adapting an Example

1. Copy the example closest to what you need
2. Change the three required substitutions (`device_name`, `device_description`, `friendly_name`)
3. Swap the hardware line if you're using a different plug
4. Add any function-specific substitutions (e.g., `latitude`/`longitude` for timed light)
5. Compile and flash

## Mix and Match

Any function works on any hardware. These examples show specific pairings, but you can freely combine:

```yaml
# Heater on a Sonoff S31? Sure.
packages:
  base: !include common/esp_common.yaml
  hardware: !include hardware/sonoff_s31.yaml
  function: !include functions/miniplug_heater.yaml

# Appliance monitor on a SwitchBot? Also works.
packages:
  base: !include common/esp_common.yaml
  hardware: !include hardware/switchbot_miniplug_W1901400.yaml
  function: !include functions/miniplug_appliance.yaml
```
