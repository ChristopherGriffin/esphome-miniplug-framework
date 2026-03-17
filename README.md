# ESPHome Smart Plug Framework

A modular, three-layer configuration system for deploying ESPHome-based smart plugs quickly and consistently. Define a device in ~15 lines of YAML by combining a **hardware profile**, a **common base**, and a **function profile**.

> **This is not a program.** It's a file-based architecture for organizing ESPHome configurations so that changes to shared files automatically propagate to every device that includes them.

## The Problem

ESPHome configs tend to become monolithic copy-paste monsters. You end up with 20 devices that are 90% identical, and when you need to change something common (like your WiFi settings or uptime format), you're editing 20 files. Worse, each copy drifts slightly over time.

## The Solution: Three Layers

Every device is assembled from three package files:

```
┌─────────────────────────────────────────────────────────┐
│                    Device Config                        │
│           (substitutions + package includes)            │
│                     ~15 lines                           │
├──────────────┬──────────────────┬───────────────────────┤
│   Hardware   │     Common       │      Function         │
│   Profile    │     Profile      │      Profile          │
│              │                  │                       │
│  Board type  │  WiFi / OTA      │  What the plug DOES   │
│  GPIO pins   │  API / Web UI    │  Relay behavior       │
│  Power chip  │  Uptime / RSSI   │  Monitoring logic     │
│  LED wiring  │  Restart switch  │  Automations          │
│  Button pin  │  Time sync       │  HA entities          │
└──────────────┴──────────────────┴───────────────────────┘
```

**Hardware** absorbs all chip-specific differences (ESP32 vs ESP8266, GPIO mappings, power monitoring chips, LED types). **Common** provides universal services every device needs. **Function** defines what the plug actually does — a lamp, a heater with freeze protection, a timed light with scheduling, an appliance monitor.

The key insight: **update one layer file, recompile, and every device using it gets the change.**

## Quick Start

### 1. Get the framework

Clone or download this repo, then copy the three directories (`hardware/`, `common/`, `functions/`) into your ESPHome config directory. Adjust the `!include` paths in your device configs to match wherever you placed them.

A typical deployed layout:

```
esphome/
├── secrets.yaml
├── hardware/
│   ├── sonoff_s31.yaml
│   └── switchbot_miniplug_W1901400.yaml
├── common/
│   └── esp_common.yaml
├── functions/
│   ├── miniplug_lamp.yaml
│   ├── miniplug_heater.yaml
│   ├── miniplug_timed_light.yaml
│   └── miniplug_appliance.yaml
└── living-room-lamp.yaml          ← your device config
```

### 2. Add your secrets

Your `secrets.yaml` needs these entries:

```yaml
wifi_ssid: "YourSSID"
wifi_password: "YourPassword"
wifi_domain: ".local"
api_encryption_key: "your-base64-key-here"
ota_password: "your-ota-password"
```

Generate an API encryption key with `esphome wizard` or any base64 encoder (32 random bytes → base64). See `secrets.yaml.example` for a template.

### 3. Create a device config

A complete device — lamp on a SwitchBot Mini Plug:

```yaml
substitutions:
  device_name: "living-room-lamp"
  device_description: "Living room floor lamp"
  friendly_name: "Living Room Lamp"
  name_no_dashes: "living_room_lamp"

packages:
  base: !include common/esp_common.yaml
  hardware: !include hardware/switchbot_miniplug_W1901400.yaml
  function: !include functions/miniplug_lamp.yaml
```

That's it. Compile and flash. You get a light entity in Home Assistant with power monitoring, WiFi diagnostics, uptime tracking, connection status LEDs, a physical button, OTA updates, and a web UI.

## Supported Hardware

| Hardware | Chip | Power Monitoring | BLE Proxy | Details |
|----------|------|-----------------|-----------|---------|
| [Sonoff S31](https://sonoff.tech/product/smart-plugs/s31/) | ESP8266 (ESP12E) | CSE7766 (UART) | No | [README](hardware/README_SONOFF_S31.md) |
| [SwitchBot Mini Plug W1901400](https://www.switchbot.com/) | ESP32-C3 | BL0937 (HLW8012) | Yes | [README](hardware/README_SWITCHBOT_MINIPLUG.md) |

Both hardware profiles expose the same interface — `relay_output`, `relay_led`, `power` sensor, and an `on_button_press` script hook — so **any function works on any hardware**.

## Available Functions

| Function | Description | Power Monitoring | Details |
|----------|-------------|-----------------|---------|
| **Lamp** | Simple on/off light | Not required | [README](functions/README_LAMP.md) |
| **Heater** | Freeze protection + fault monitoring | Required | [README](functions/README_HEATER.md) |
| **Timed Light** | Sunrise/sunset/fixed/duration scheduling | Not required | [README](functions/README_TIMED_LIGHT.md) |
| **Appliance Monitor** | Power-based state detection + cycle alerts | Required | [README](functions/README_APPLIANCE.md) |

## How the Layers Connect

The layers communicate through a well-defined interface contract:

### Hardware provides → Function consumes

| ID | Type | Description |
|----|------|-------------|
| `relay_output` | `output` | GPIO output that controls the relay |
| `relay_led` | `light` | Internal light entity for the relay status LED |
| `power` | `sensor` | Real-time power consumption in watts |
| `on_button_press` | `script` | **Defined by function** — hardware calls this on button press |

### Common provides → All layers consume

| ID | Type | Description |
|----|------|-------------|
| `device_time` | `time` | Synced time component (from Home Assistant) |
| `uptime_sensor` | `sensor` | Raw uptime in seconds (internal) |

### How ESPHome merges packages

ESPHome merges all `packages:` entries into a single configuration. List-type sections (`sensor:`, `binary_sensor:`, `switch:`, etc.) are **additive** — items from all packages combine. Scalar values (`esphome: name:`, `esp32: board:`, etc.) use **last-write-wins** — the hardware profile overrides the common base.

## Creating Your Own Function

A function package needs to:

1. **Consume** `relay_output` and `relay_led` from the hardware package
2. **Implement** `on_button_press` as a script (hardware calls this when the physical button is pressed)
3. **Optionally consume** `power` for power monitoring logic
4. **Optionally consume** `device_time` for time-based logic

Minimal function template:

```yaml
#==============================================================================
# Mini Plug Function: My Custom Function
#==============================================================================
# DEPENDENCIES:
#   - Hardware providing: relay_output, relay_led
#   - Function must implement: on_button_press
#   - (Optional) power sensor for power monitoring
#==============================================================================

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

## LED Behavior

Both hardware profiles implement a consistent connection status LED pattern:

| State | LED Behavior |
|-------|-------------|
| WiFi disconnected | Fast blink (0.5s) |
| WiFi connected, API disconnected | Solid on |
| WiFi + API connected | Slow glow (2s sine wave) |

The relay status LED is managed by the function package (via `relay_led`), except on the Sonoff S31 where the red LED is hardware-coupled to the relay.

## Examples

See [`examples/`](examples/) for complete, ready-to-use device configs:

| Example | Hardware | Function |
|---------|----------|----------|
| [`lamp_switchbot.yaml`](examples/lamp_switchbot.yaml) | SwitchBot Mini Plug | Simple lamp |
| [`lamp_sonoff.yaml`](examples/lamp_sonoff.yaml) | Sonoff S31 | Simple lamp |
| [`heater_switchbot.yaml`](examples/heater_switchbot.yaml) | SwitchBot Mini Plug | Freeze-protected heater |
| [`timed_light_switchbot.yaml`](examples/timed_light_switchbot.yaml) | SwitchBot Mini Plug | Scheduled light |
| [`appliance_sonoff.yaml`](examples/appliance_sonoff.yaml) | Sonoff S31 | Appliance monitor |

## Deployment

### Option A: Local copy

Clone the repo and copy `hardware/`, `common/`, and `functions/` into your ESPHome config directory. Adjust `!include` paths in your device configs to match wherever you placed them.

### Option B: Pull directly from GitHub

ESPHome can reference packages from a GitHub repo at compile time:

```yaml
packages:
  base:
    url: https://github.com/YOUR_USERNAME/esphome-miniplug-framework
    file: common/esp_common.yaml
    refresh: 1d
  hardware:
    url: https://github.com/YOUR_USERNAME/esphome-miniplug-framework
    file: hardware/switchbot_miniplug_W1901400.yaml
    refresh: 1d
  function:
    url: https://github.com/YOUR_USERNAME/esphome-miniplug-framework
    file: functions/miniplug_lamp.yaml
    refresh: 1d
```

`!secret` references in remote packages resolve against your **local** `secrets.yaml`, so credentials stay on your machine. The `refresh` interval controls how often ESPHome checks for upstream changes.

## Project Structure

```
├── README.md                                  # You are here
├── hardware/                                  # Layer 1: Hardware profiles
│   ├── README.md
│   ├── README_SONOFF_S31.md
│   ├── README_SWITCHBOT_MINIPLUG.md
│   ├── sonoff_s31.yaml
│   └── switchbot_miniplug_W1901400.yaml
├── common/                                    # Layer 2: Universal base
│   └── esp_common.yaml
├── functions/                                 # Layer 3: What the plug does
│   ├── README.md
│   ├── README_LAMP.md
│   ├── README_HEATER.md
│   ├── README_TIMED_LIGHT.md
│   ├── README_APPLIANCE.md
│   ├── miniplug_lamp.yaml
│   ├── miniplug_heater.yaml
│   ├── miniplug_timed_light.yaml
│   └── miniplug_appliance.yaml
├── examples/                                  # Complete device configs
│   ├── README.md
│   ├── lamp_switchbot.yaml
│   ├── lamp_sonoff.yaml
│   ├── heater_switchbot.yaml
│   ├── timed_light_switchbot.yaml
│   └── appliance_sonoff.yaml
├── docs/                                      # Deep-dive documentation
│   ├── ARCHITECTURE.md
│   ├── ADDING_HARDWARE.md
│   └── SUBSTITUTIONS.md
├── secrets.yaml.example
├── LICENSE
└── .gitignore
```

## Philosophy

- **No vendor lock-in** — commodity smart plugs running open source firmware
- **Modular by design** — change one file, all devices update on next compile
- **Hardware-agnostic functions** — write once, deploy on any supported plug
- **Self-documenting** — every YAML file has a header listing dependencies, provided IDs, and exposed entities
- **Practical over clever** — YAML over C++, explicit over implicit, readable over compact

## Contributing

Adding a new plug type is straightforward — see [docs/ADDING_HARDWARE.md](docs/ADDING_HARDWARE.md). PRs welcome.

## License

MIT — use it, fork it, adapt it.
