# Villavent VR 400/E3 — ESPHome Control via ESP32

Control your Villavent VR 400/E3 ventilation unit using an ESP32 and ESPHome, integrated with Home Assistant. Instead of modifying the original electronics, this project uses **optocouplers** to simulate button presses and read LED status indicators — keeping the original control board fully intact.

## How It Works

The Villavent VR 400/E3 has a physical control panel with buttons (fan speed up/down, temperature up/down) and LEDs indicating the current mode. This project taps into those signals using optocouplers in two ways:

**Outputs (simulating button presses):** An optocoupler is connected in parallel with each button. When the ESP32 briefly pulls the GPIO high (300ms pulse), the optocoupler closes the circuit — identical to a physical button press.

**Inputs (reading LED state):** An optocoupler is connected across each status LED. When the LED lights up, it drives the optocoupler which pulls the ESP32 GPIO low (inputs are configured with pullup + inverted).

This approach means zero direct electrical connection between the ESP32 and the ventilation unit's PCB.

## Features

- Fan speed control (UP / DOWN) with current state feedback (MIN / NORMAL / MAX)
- Temperature setpoint control (UP / DOWN) with current state feedback (levels 0–5)
- Optional status sensors: Electric heater active, Summer mode, Filter change alert
- Full Home Assistant integration via ESPHome API
- OTA firmware updates

## Pin Mapping

### Outputs — Button Simulation

| Function       | GPIO |
|----------------|------|
| Fan UP         | 5    |
| Fan DOWN       | 17   |
| Temp UP        | 16   |
| Temp DOWN      | 4    |

### Inputs — LED Reading

| Function              | GPIO | Notes                        |
|-----------------------|------|------------------------------|
| Fan speed MAX         | 25   | Pullup, inverted             |
| Fan speed NORMAL      | 26   | Pullup, inverted             |
| Fan speed MIN         | 27   | Pullup, inverted             |
| Temp lamp 6 (left)    | 32   | Pullup, inverted             |
| Temp lamp 7 (center)  | 33   | Pullup, inverted             |
| Temp lamp 8 (right)   | 15   | Pullup, inverted             |
| Electric heater active| 13   | Optional — comment out if not wired |
| Summer mode           | 14   | Optional — comment out if not wired |
| Filter change alert   | 12   | Optional — comment out if not wired |

## Temperature Level Decoding

The VR 400/E3 uses a 3-LED combination to indicate one of 6 temperature setpoints:

| Level | Lamp 6 | Lamp 7 | Lamp 8 |
|-------|--------|--------|--------|
| 0     | OFF    | OFF    | OFF    |
| 1     | ON     | OFF    | OFF    |
| 2     | ON     | ON     | OFF    |
| 3     | OFF    | ON     | OFF    |
| 4     | OFF    | ON     | ON     |
| 5     | OFF    | OFF    | ON     |

This decoding runs as a template sensor in ESPHome, updating every second.

## Getting Started

### Prerequisites

- [ESPHome](https://esphome.io) installed (via Home Assistant add-on or CLI)
- ESP32 development board (e.g. ESP32-WROOM)
- Optocouplers (e.g. PC817 or similar 4N35)
- Home Assistant instance

### Installation

1. Clone this repository:
   ```bash
   git clone https://github.com/YOUR_USERNAME/villavent-vr400-esphome.git
   cd villavent-vr400-esphome
   ```

2. Create a `secrets.yaml` file in the same directory:
   ```yaml
   wifi_ssid: "YourWiFiName"
   wifi_password: "YourWiFiPassword"
   ```

3. Replace the placeholder values in `vr.yaml`:
   - `YOUR_API_KEY_HERE` with your ESPHome API encryption key (generate one with `esphome generate-api-key`)
   - `YOUR_OTA_PASSWORD_HERE` with a password for OTA updates
   - `YOUR_FALLBACK_PASSWORD_HERE` with a fallback hotspot password

4. Flash the ESP32:
   ```bash
   esphome run vr.yaml
   ```

5. Add the device to Home Assistant via Settings > Integrations > ESPHome.

### Optional Sensors

Three additional binary sensors (Electric heater, Summer mode, Filter change) are included in the config but **comment them out** if you have not wired those optocouplers yet:

```yaml
# Comment these out if not yet connected:
  - platform: gpio
    pin:
      number: GPIO13
    name: "Elpatron Aktiv"
    ...
```

## Hardware Notes

- Use a **current-limiting resistor** in series with each optocoupler LED side, sized for the LED voltage on the VR 400/E3 panel.
- The output optocouplers (button simulation) switch the collector/emitter across the original button contacts.
- The input optocouplers (LED reading) have their LED side across the status LED, and the output side connected to the ESP32 GPIO and GND.
- The ESP32 and the ventilation unit share **no common ground** — the optocouplers provide full galvanic isolation.

## License

MIT
