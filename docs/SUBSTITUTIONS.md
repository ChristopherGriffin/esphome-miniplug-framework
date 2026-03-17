# Substitutions Reference

Every device config needs a set of substitutions that the three layers consume. This document lists all of them.

## Required (Every Device)

These substitutions are referenced by the common and hardware packages. Every device config must define them.

| Substitution | Example | Used By | Description |
|-------------|---------|---------|-------------|
| `device_name` | `"living-room-lamp"` | Common, Hardware | ESPHome device name. Lowercase, dashes OK. Becomes the mDNS hostname. |
| `friendly_name` | `"Living Room Lamp"` | Common, Hardware | Human-readable name shown in Home Assistant. |
| `name_no_dashes` | `"living_room_lamp"` | Common | Used for internal IDs (ESPHome IDs can't have dashes). |
| `device_description` | `"Floor lamp in living room"` | Common, Hardware | Comment stored in the device config. Shows in ESPHome dashboard. |

## Hardware-Specific (Defined by Hardware Profiles)

These have defaults in the hardware profile but can be overridden in your device config.

### Sonoff S31

| Substitution | Default | Description |
|-------------|---------|-------------|
| `pin_button` | `GPIO0` | Physical button GPIO |
| `pin_relay` | `GPIO12` | Relay GPIO |
| `pin_led_green` | `GPIO13` | Connection status LED |
| `power_update_interval` | `"5s"` | How often power readings update |

### SwitchBot Mini Plug W1901400

| Substitution | Default | Description |
|-------------|---------|-------------|
| `pin_button` | `GPIO02` | Physical button GPIO |
| `pin_relay` | `GPIO06` | Relay GPIO |
| `pin_led_white` | `GPIO07` | Relay status LED |
| `pin_led_blue` | `GPIO08` | Connection status LED |
| `pin_hlw_cf` | `GPIO18` | BL0937 CF pin |
| `pin_hlw_cf1` | `GPIO19` | BL0937 CF1 pin |
| `pin_hlw_sel` | `GPIO20` | BL0937 SEL pin |
| `voltage_divider` | `"1467"` | BL0937 calibration |
| `current_resistor` | `"0.001"` | BL0937 calibration |
| `power_update_interval` | `"2s"` | How often power readings update |

## Function-Specific

### Timed Light (`miniplug_timed_light.yaml`)

| Substitution | Required | Example | Description |
|-------------|----------|---------|-------------|
| `latitude` | **Yes** | `"33.4484"` | Location latitude for sun calculations |
| `longitude` | **Yes** | `"-112.0740"` | Location longitude (negative for western hemisphere) |

All other timed light parameters (schedule source, times, duration, offsets) are configurable from Home Assistant at runtime — no substitutions needed.

### Heater, Lamp, Appliance Monitor

No additional substitutions required beyond the base set.

## Secrets

The common package references these `!secret` entries. Define them in your ESPHome `secrets.yaml`:

| Secret | Description |
|--------|-------------|
| `wifi_ssid` | WiFi network name |
| `wifi_password` | WiFi password |
| `wifi_domain` | DNS domain suffix (e.g., `".local"` or `".home.lan"`) |
| `api_encryption_key` | Base64 API encryption key (generate with `esphome wizard`) |
| `ota_password` | OTA update password |

## Overriding Defaults

Substitutions defined in your device config override defaults from any package. This means you can customize hardware defaults per-device:

```yaml
substitutions:
  device_name: "my-heater"
  friendly_name: "My Heater"
  name_no_dashes: "my_heater"
  device_description: "Garage heater"
  # Override the default 2s update interval for faster monitoring
  power_update_interval: "1s"

packages:
  base: !include common/esp_common.yaml
  hardware: !include hardware/switchbot_miniplug_W1901400.yaml
  function: !include functions/miniplug_heater.yaml
```
