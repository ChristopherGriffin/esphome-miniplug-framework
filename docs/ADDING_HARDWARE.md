# Adding a New Hardware Profile

This guide walks through creating a hardware profile for a new smart plug so it works with all existing function packages.

## Prerequisites

Before starting, you'll need:
- The plug flashed with ESPHome (or a way to get there — Tuya Convert, serial flash, etc.)
- The GPIO pinout for your plug (check [templates.blakadder.com](https://templates.blakadder.com/) or open it up)
- Knowledge of which power monitoring chip it uses (if any)

## The Interface Contract

Your hardware profile **must provide** these IDs:

| ID | Type | Required | Description |
|----|------|----------|-------------|
| `relay_output` | `output` | **Yes** | GPIO output for the relay |
| `relay_led` | `light` (binary, internal) | **Yes** | Status LED for relay state |
| `power` | `sensor` | **Yes*** | Power consumption in watts |
| `current` | `sensor` | No | Current in amps |
| `voltage` | `sensor` | No | Voltage in volts |
| `energy` | `sensor` | No | Cumulative energy in kWh |

*\*Required if using power-monitoring function packages (heater, appliance). Not needed for lamp.*

Your hardware profile **must call**:

| ID | Type | Description |
|----|------|-------------|
| `on_button_press` | `script` | Call this when the physical button is pressed |

Your hardware profile **should provide**:

| ID | Type | Description |
|----|------|-------------|
| `update_status_led` | `script` | Connection status LED management |

## Step-by-Step

### 1. Start with the template

```yaml
#==============================================================================
# [Plug Name] - Hardware Package
#==============================================================================
# File: hardware/[plug_name].yaml
#
# Hardware-specific configuration for [Plug Name] ([chip type]).
#
# HARDWARE PINOUT:
#   - GPIOxx : Button
#   - GPIOxx : Relay
#   - GPIOxx : LED (status)
#   - GPIOxx : LED (relay) or note if hardware-coupled
#   - GPIOxx : Power monitoring pins
#
# PROVIDES: relay_output, relay_led, power, current, voltage, energy
# REQUIRES: on_button_press (from function package)
#==============================================================================

substitutions:
  pin_button: GPIOxx
  pin_relay: GPIOxx
  pin_led_status: GPIOxx
  # Add power monitoring pins as needed
  power_update_interval: "5s"
```

### 2. Define the board

```yaml
# For ESP8266-based plugs:
esp8266:
  board: esp01_1m  # or esp12e, etc.
  early_pin_init: false
  restore_from_flash: false

# For ESP32-based plugs:
esp32:
  board: esp32dev  # or esp32-c3-devkitm-1, etc.
  framework:
    type: esp-idf   # or arduino
```

### 3. Define the relay output

```yaml
output:
  - platform: gpio
    id: relay_output
    pin: ${pin_relay}
```

### 4. Define the relay LED

If the LED is **software-controllable** (like the SwitchBot):
```yaml
output:
  - platform: gpio
    id: relay_led_output
    pin:
      number: ${pin_led_relay}
      inverted: true  # Most LEDs are active-low

light:
  - platform: binary
    id: relay_led
    output: relay_led_output
    internal: true
```

If the LED is **hardware-coupled** to the relay (like the S31):
```yaml
output:
  - platform: template
    id: relay_led_output
    type: binary
    write_action:
      - logger.log:
          level: DEBUG
          format: "Relay LED state: %d (hardware-coupled)"
          args: ['state']

light:
  - platform: binary
    id: relay_led
    output: relay_led_output
    internal: true
```

### 5. Define the physical button

```yaml
binary_sensor:
  - platform: gpio
    id: physical_button
    internal: true
    pin:
      number: ${pin_button}
      mode: INPUT_PULLUP
      inverted: true
    filters:
      - delayed_on: 10ms
    on_press:
      - script.execute: on_button_press
```

### 6. Define power monitoring

This varies by chip. Common options:

**CSE7766** (Sonoff S31, Sonoff POW):
```yaml
uart:
  rx_pin: RX
  baud_rate: 4800
  parity: EVEN

sensor:
  - platform: cse7766
    power:
      name: "Power"
      id: power
      # ...
```

**BL0937 / HLW8012** (SwitchBot, Athom, many Tuya plugs):
```yaml
sensor:
  - platform: hlw8012
    model: BL0937  # or HLW8012
    sel_pin: ...
    cf_pin: ...
    cf1_pin: ...
    power:
      name: "Power"
      id: power
      # ...
```

**No power monitoring** (basic relay plugs):
If the plug has no power monitoring chip, provide a dummy sensor:
```yaml
sensor:
  - platform: template
    name: "Power"
    id: power
    unit_of_measurement: W
    lambda: 'return 0;'
    update_interval: never
```

### 7. Implement the status LED

Copy the `update_status_led` script and LED effects from an existing hardware profile. The pattern (fast blink / solid / slow glow) should stay consistent across all hardware.

### 8. Test

1. Create a test device config using your new hardware profile + the lamp function
2. Compile and verify no errors
3. Flash and verify: button toggles lamp, LEDs behave correctly, power readings are sane
4. Test with heater and appliance functions to verify power monitoring IDs work

## Common Pitfalls

- **UART conflicts**: If the power monitoring chip uses UART, set `logger: baud_rate: 0`
- **PWM platform**: ESP8266 uses `esp8266_pwm`, ESP32 uses `ledc` for LED PWM
- **Pin modes**: Some chips need specific pin modes — check the datasheet
- **Power calibration**: BL0937/HLW8012 plugs often need `voltage_divider` and `current_resistor` calibration. Measure with a known load and adjust.
