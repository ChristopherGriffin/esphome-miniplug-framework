# Sonoff S31 — Hardware Profile

The [Sonoff S31](https://sonoff.tech/product/smart-plugs/s31/) is an ESP8266-based smart plug with built-in CSE7766 power monitoring. It's one of the most popular ESPHome-compatible plugs — cheap, reliable, and well-documented.

## Specs

| Feature | Detail |
|---------|--------|
| **Chip** | ESP8266 (ESP12E module) |
| **Framework** | Arduino |
| **Power Monitoring** | CSE7766 via UART (accurate, fast) |
| **Max Load** | 15A / 1800W |
| **BLE Proxy** | No (ESP8266 has no Bluetooth) |
| **Physical Button** | Yes (GPIO0) |
| **LEDs** | Green (connection status, software-controlled), Red (relay status, hardware-coupled) |

## GPIO Pinout

```
┌─────────────────────────────────┐
│          Sonoff S31             │
│                                 │
│  GPIO0  ── Button (pullup, inv) │
│  GPIO12 ── Relay                │
│  GPIO13 ── Green LED (inv)      │
│  RX     ── CSE7766 UART         │
│                                 │
│  Red LED is wired directly to   │
│  the relay — not controllable   │
│  in software.                   │
└─────────────────────────────────┘
```

## LED Behavior

### Green LED — Connection Status

| State | Behavior |
|-------|----------|
| WiFi disconnected | Fast blink (0.5s on/off) |
| WiFi connected, API disconnected | Solid on |
| WiFi + API connected | Slow sine-wave glow (~2s cycle) |

### Red LED — Relay Status

The red LED is **hardware-wired** to the relay. It turns on when the relay is energized and off when it's not. There is no software control over this LED.

To maintain compatibility with function packages that call `relay_led.turn_on()` / `relay_led.turn_off()`, this profile provides a **dummy `relay_led`** light entity backed by a template output. The calls succeed silently — the real indication comes from the hardware-coupled red LED.

## Power Monitoring

The CSE7766 communicates over UART, which means **the serial logger must be disabled** (`baud_rate: 0`). The hardware profile handles this automatically. You won't get serial debug output over USB, but the web UI and wireless logs still work fine.

### Exposed Sensors

| Sensor | ID | Unit | Description |
|--------|----|------|-------------|
| Power | `power` | W | Real-time active power |
| Current | `current` | A | RMS current draw |
| Voltage | `voltage` | V | Line voltage |
| Energy | `energy` | kWh | Cumulative energy (total increasing) |
| Power Factor | — | — | Ratio of real to apparent power |
| Daily Energy | — | kWh | Resets at midnight via `total_daily_energy` |

### Update Interval

Default: `5s` (configurable via `power_update_interval` substitution). Uses `throttle_average` filtering so you get a smoothed reading at your chosen interval without losing data between samples.

For heater monitoring where response time matters, keep at `5s` or lower. For lamps or general monitoring, `30s` or `60s` reduces network traffic.

## Flashing

The S31 requires **serial flashing** (no OTA from stock firmware). You'll need to open the case and connect a USB-to-serial adapter to the programming header. After the first ESPHome flash, all future updates are OTA.

Plenty of guides exist for this — search "Sonoff S31 ESPHome flash" for step-by-step photos.

## Usage

```yaml
substitutions:
  device_name: "my-device"
  device_description: "My Sonoff S31 device"
  friendly_name: "My Device"
  name_no_dashes: "my_device"

packages:
  base: !include common/esp_common.yaml
  hardware: !include hardware/sonoff_s31.yaml
  function: !include functions/miniplug_lamp.yaml
```

## Overridable Substitutions

| Substitution | Default | Description |
|-------------|---------|-------------|
| `pin_button` | `GPIO0` | Physical button GPIO |
| `pin_relay` | `GPIO12` | Relay GPIO |
| `pin_led_green` | `GPIO13` | Connection status LED |
| `power_update_interval` | `"5s"` | Power sensor update rate |
