# SwitchBot Mini Plug W1901400 — Hardware Profile

The [SwitchBot Mini Plug W1901400](https://www.switchbot.com/) is an ESP32-C3 based smart plug with BL0937 power monitoring and Bluetooth LE support. The ESP32-C3 gives it significantly more headroom than ESP8266-based plugs — more RAM, better WiFi, and BLE proxy capability.

## Specs

| Feature | Detail |
|---------|--------|
| **Chip** | ESP32-C3 |
| **Framework** | ESP-IDF |
| **Power Monitoring** | BL0937 (HLW8012 protocol) |
| **Max Load** | 15A / 1800W |
| **BLE Proxy** | Yes — acts as a Bluetooth relay for Home Assistant |
| **Physical Button** | Yes (GPIO02) |
| **LEDs** | Blue (connection status, PWM), White (relay status, software-controlled) |

## GPIO Pinout

```
┌──────────────────────────────────────┐
│      SwitchBot Mini Plug W1901400    │
│                                      │
│  GPIO02 ── Button (pullup, inv)      │
│  GPIO06 ── Relay                     │
│  GPIO07 ── White LED (inv) - Relay   │
│  GPIO08 ── Blue LED (inv) - Status   │
│  GPIO18 ── BL0937 CF                 │
│  GPIO19 ── BL0937 CF1               │
│  GPIO20 ── BL0937 SEL (inv)         │
└──────────────────────────────────────┘
```

## LED Behavior

### Blue LED — Connection Status

| State | Behavior |
|-------|----------|
| WiFi disconnected | Fast blink (0.5s on/off) |
| WiFi connected, API disconnected | Solid on |
| WiFi + API connected | Slow sine-wave glow (~2s cycle) |

### White LED — Relay Status

The white LED is **software-controllable** via the `relay_led` light entity. Function packages call `relay_led.turn_on()` and `relay_led.turn_off()` to sync it with the relay state. Unlike the S31's hardware-coupled red LED, this one is fully under software control.

## Power Monitoring

The BL0937 uses pulse-counting via GPIO (not UART), so there's no conflict with the serial logger. The HLW8012 ESPHome component handles the protocol.

### Exposed Sensors

| Sensor | ID | Unit | Description |
|--------|----|------|-------------|
| Power | `power` | W | Real-time active power |
| Current | `current` | A | RMS current draw |
| Voltage | `voltage` | V | Line voltage |
| Energy | `energy` | kWh | Cumulative energy (total increasing) |
| Daily Energy | — | kWh | Resets at midnight via `total_daily_energy` |
| Internal Temp | — | °C | ESP32-C3 die temperature (diagnostic) |

### Calibration

The BL0937 accuracy depends on two hardware-specific calibration values:

| Substitution | Default | Description |
|-------------|---------|-------------|
| `voltage_divider` | `"1467"` | Resistive divider ratio for voltage measurement |
| `current_resistor` | `"0.001"` | Current sense resistor value in ohms |

The defaults work reasonably well out of the box. For better accuracy, measure with a known load (e.g., a 100W incandescent bulb on a Kill-A-Watt) and adjust. The ESPHome docs on [HLW8012 calibration](https://esphome.io/components/sensor/hlw8012.html#calibration) cover the process.

### Update Interval

Default: `2s` (configurable via `power_update_interval` substitution). The BL0937 alternates between measuring voltage and current (`change_mode_every: 4` — switches every 4 update cycles), so power readings update at the configured interval but voltage/current alternate.

## Button Behavior

The physical button (GPIO02) uses ESPHome's `on_multi_click` to distinguish short and long presses. It calls scripts defined by the function package — the hardware only detects the gesture and delegates.

| Action | Script Called | Typical Result |
|--------|--------------|----------------|
| Tap and release (< 2.9s) | `on_button_press` | Toggle lamp |
| Hold 3s (while held) | `on_button_hold` | Toggle schedule or other function-defined action |

Short press fires on release. Long hold fires at the 3-second mark while the button is still held — the function package flashes the LED to signal it's safe to release. Release after the 3s mark is ignored.

The actual behavior depends on which function package is loaded. See the function README for details.

## BLE Proxy

This plug doubles as a **Bluetooth LE proxy** for Home Assistant. Any BLE devices in range (SwitchBot sensors, Xiaomi thermometers, plant sensors, etc.) can be read by HA through this plug without a dedicated BLE adapter on the HA host.

The BLE scanner runs alongside WiFi with minimal impact. If you don't need BLE proxy functionality, you can disable it by adding this to your device config:

```yaml
# Disable BLE proxy (saves some memory and radio time)
esp32_ble_tracker:
  scan_parameters:
    active: false
    interval: 1200ms
    window: 100ms

bluetooth_proxy:
  active: false
```

## Flashing

The W1901400 can typically be flashed via **Tuya Convert** (OTA from stock firmware) or by serial if Tuya Convert doesn't work. Check the current status of Tuya Convert compatibility before attempting — some newer production runs have patched it out.

For serial flashing, the board has test pads for 3.3V, GND, TX, and RX. You'll need to open the case and solder temporary leads.

## ESP-IDF Framework

This profile uses the **ESP-IDF framework** (not Arduino) for better stability and memory management on the ESP32-C3. The ESP-IDF build includes these sdkconfig options:

```yaml
sdkconfig_options:
  CONFIG_BT_BLE_50_FEATURES_SUPPORTED: y   # BLE 5.0 features
  CONFIG_BT_BLE_42_FEATURES_SUPPORTED: y   # BLE 4.2 compatibility
  CONFIG_ESP_TASK_WDT_TIMEOUT_S: "10"      # Watchdog timeout (generous for BLE operations)
```

## Usage

```yaml
substitutions:
  device_name: "my-device"
  device_description: "My SwitchBot Mini Plug device"
  friendly_name: "My Device"
  name_no_dashes: "my_device"

packages:
  base: !include common/esp_common.yaml
  hardware: !include hardware/switchbot_miniplug_W1901400.yaml
  function: !include functions/miniplug_lamp.yaml
```

## Overridable Substitutions

| Substitution | Default | Description |
|-------------|---------|-------------|
| `pin_button` | `GPIO02` | Physical button GPIO |
| `pin_relay` | `GPIO06` | Relay GPIO |
| `pin_led_white` | `GPIO07` | Relay status LED |
| `pin_led_blue` | `GPIO08` | Connection status LED |
| `pin_hlw_cf` | `GPIO18` | BL0937 CF pin |
| `pin_hlw_cf1` | `GPIO19` | BL0937 CF1 pin |
| `pin_hlw_sel` | `GPIO20` | BL0937 SEL pin |
| `voltage_divider` | `"1467"` | BL0937 voltage calibration |
| `current_resistor` | `"0.001"` | BL0937 current calibration |
| `power_update_interval` | `"2s"` | Power sensor update rate |
