# Architecture: The Three-Layer Pattern

## Why Three Layers?

When you manage more than a handful of ESPHome devices, two problems emerge:

1. **Config drift** — each device is a copy-pasted monolith that diverges over time
2. **Shotgun surgery** — changing something common (WiFi, uptime format, API key) means editing every file

The three-layer pattern solves both by separating concerns into composable packages:

```
┌─────────────────────────────────────────────────────────┐
│                    Device Config                        │
│           (substitutions + package includes)            │
├──────────────┬──────────────────┬───────────────────────┤
│   Hardware   │     Common       │      Function         │
│   Profile    │     Profile      │      Profile          │
│              │                  │                       │
│ "What is it" │ "What everything │ "What does it DO"     │
│              │  needs"          │                       │
└──────────────┴──────────────────┴───────────────────────┘
```

## How ESPHome Package Merging Works

When you write:

```yaml
packages:
  base: !include common/esp_common.yaml
  hardware: !include hardware/switchbot_miniplug_W1901400.yaml
  function: !include functions/miniplug_lamp.yaml
```

ESPHome merges all three files into a single configuration. The rules:

- **List sections** (`sensor:`, `binary_sensor:`, `switch:`, `light:`, `text_sensor:`, `script:`, etc.) are **additive** — items from all packages combine into one list.
- **Scalar/dict sections** (`esphome:`, `esp32:`, `wifi:`, `api:`, etc.) use **last-write-wins** — later packages override earlier ones.
- **Substitutions** defined in the device config override any defaults in packages.

This is why the hardware package can set `esphome: name:` and it overrides the common package's value. And why all three packages can define `sensor:` entries that all end up in the final config.

## The Interface Contract

The layers communicate through well-known ESPHome IDs. This is the contract:

### Hardware → Function

The hardware package **provides** these IDs that function packages **consume**:

| ID | Type | Description |
|----|------|-------------|
| `relay_output` | `output` | GPIO output controlling the physical relay |
| `relay_led` | `light` | Internal light entity for relay status LED |
| `power` | `sensor` | Real-time power consumption in watts |
| `current` | `sensor` | Current draw in amps |
| `voltage` | `sensor` | Line voltage in volts |
| `energy` | `sensor` | Cumulative energy in kWh |

### Function → Hardware

The function package **provides** this script that hardware **calls**:

| ID | Type | Description |
|----|------|-------------|
| `on_button_press` | `script` | Called when the physical button is pressed |

This is the key design decision: the hardware knows *that* the button was pressed, but the function decides *what happens*. A lamp toggles. A heater toggles (and clears faults). A timed light toggles regardless of schedule.

### Common → Everyone

The common package **provides**:

| ID | Type | Description |
|----|------|-------------|
| `device_time` | `time` | Synced time (from Home Assistant) |
| `uptime_sensor` | `sensor` | Raw uptime in seconds |

## The S31 Relay LED Problem

The Sonoff S31 has a red LED that is **hardware-wired** to the relay — it turns on when the relay is on, and there's no software control. But function packages expect to call `relay_led.turn_on()` and `relay_led.turn_off()`.

The solution: the S31 hardware profile provides a **dummy `relay_led`** — a template output that logs the call but does nothing. This lets function packages work identically on both hardware platforms without `#ifdef`-style conditionals.

## Cross-Platform Compatibility

The common package (`esp_common.yaml`) contains **zero platform-specific code**. It works on ESP8266 and ESP32 identically.

Hardware differences are isolated in the hardware profiles:

| Concern | Sonoff S31 (ESP8266) | SwitchBot Mini (ESP32-C3) |
|---------|---------------------|--------------------------|
| Board | `esp8266: board: esp12e` | `esp32: board: esp32-c3-devkitm-1` |
| Framework | Arduino (implicit) | ESP-IDF |
| Power chip | CSE7766 (UART) | BL0937 (HLW8012 protocol) |
| Status LED PWM | `esp8266_pwm` | `ledc` |
| Relay LED | Hardware-coupled (dummy output) | Software-controllable GPIO |
| BLE Proxy | Not available | Yes (ESP32 has Bluetooth) |
| Logger UART | `baud_rate: 0` (UART used for CSE7766) | Default (no conflict) |

## Using GitHub Remote Packages

ESPHome can pull packages directly from GitHub:

```yaml
packages:
  base:
    url: https://github.com/YOUR_USERNAME/esphome-miniplug-framework
    file: common/esp_common.yaml
    refresh: 1d
```

**Important caveat:** `!secret` references in package files resolve against your **local** `secrets.yaml`, so they work fine even with remote packages. However, `!include` paths inside a remote package resolve relative to the remote repo, not your local filesystem. Since none of the framework files use `!include` internally (only device configs do), this works correctly.

## Extending the Pattern

### Adding a new function

Create a new YAML file in `functions/` that:
1. Consumes `relay_output` and `relay_led`
2. Implements `on_button_press`
3. Optionally uses `power`, `device_time`

### Adding new hardware

Create a new YAML file in `hardware/` that:
1. Defines the board, pins, and chip config
2. Provides `relay_output`, `relay_led`, `power`, `current`, `voltage`, `energy`
3. Calls `on_button_press` from the physical button
4. Implements `update_status_led` with appropriate LED effects

See [ADDING_HARDWARE.md](ADDING_HARDWARE.md) for a step-by-step guide.

### Beyond smart plugs

The three-layer pattern isn't limited to plugs. The same architecture works for:
- **Smart switches** — hardware profile defines the switch GPIO, function defines behavior
- **Relay boards** — hardware provides multiple outputs, function coordinates them
- **Sensor nodes** — hardware defines I2C/SPI/UART, function defines what to measure

The key principle: **separate what varies (hardware, function) from what's constant (connectivity, diagnostics)**.
