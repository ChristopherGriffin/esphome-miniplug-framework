# Hardware Profiles

Hardware profiles define everything specific to a physical smart plug: the chip, GPIO pins, power monitoring chip, LED wiring, and button input.

**The key rule:** hardware profiles expose a consistent interface so function packages work on any supported plug without modification.

## Interface Contract

Every hardware profile **provides** these IDs:

| ID | Type | Description |
|----|------|-------------|
| `relay_output` | `output` | GPIO output that drives the physical relay |
| `relay_led` | `light` | Internal light entity for relay status LED |
| `power` | `sensor` | Real-time power in watts |
| `current` | `sensor` | Current draw in amps |
| `voltage` | `sensor` | Line voltage in volts |
| `energy` | `sensor` | Cumulative energy in kWh |

Every hardware profile **expects** this from the function package:

| ID | Type | Description |
|----|------|-------------|
| `on_button_press` | `script` | Called when the physical button is pressed |

## Supported Hardware

| Plug | File | Chip | Power Chip | Details |
|------|------|------|-----------|---------|
| Sonoff S31 | [`sonoff_s31.yaml`](sonoff_s31.yaml) | ESP8266 (ESP12E) | CSE7766 | [README →](README_SONOFF_S31.md) |
| SwitchBot Mini Plug W1901400 | [`switchbot_miniplug_W1901400.yaml`](switchbot_miniplug_W1901400.yaml) | ESP32-C3 | BL0937 | [README →](README_SWITCHBOT_MINIPLUG.md) |

## Adding a New Plug

See [docs/ADDING_HARDWARE.md](../docs/ADDING_HARDWARE.md) for a step-by-step guide.
