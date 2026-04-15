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
   git clone https://github.com/edwardhallgren/villavent-vr400-esphome.git
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

## Home Assistant Dashboard

A ready-made Lovelace card is included in [`lovelace-card.yaml`](lovelace-card.yaml). It provides:

- Status chips for Summer mode, Electric heater, and Filter alert
- One-tap fan speed presets (Min / Normal / Max) with active-state highlighting
- Current fan speed display
- Step-by-step fan and temperature nudge buttons
- Boost mode (20 min at MAX) with a live countdown bar that cancels on tap

![Dashboard preview showing fan speed control, boost button, and temperature level selector](https://raw.githubusercontent.com/edwardhallgren/villavent-vr400-esphome/main/lovelace-card.yaml)

### Required HACS Frontend Integrations

Install these via HACS → Frontend before adding the card:

| Integration | Repository |
|---|---|
| Mushroom | `piitaya/lovelace-mushroom` |
| card-mod | `thomasloven/lovelace-card-mod` |
| timer-bar-card | `rianadon/timer-bar-card` |

### Entity Naming

The card expects entity IDs with a `vr_` prefix (e.g. `binary_sensor.vr_flakt_lage_max`). This means the ESPHome device must be named **`vr`** in `vr.yaml`:

```yaml
esphome:
  name: vr
```

If you use a different device name, do a find-and-replace on the `vr_` prefix in `lovelace-card.yaml`.

### Additional HA Helpers Required

The card calls scripts and a timer that you need to create manually in Home Assistant (Settings → Automations & Scenes):

**Timer**

| Entity | Duration |
|---|---|
| `timer.villavent_boost_timer` | 20 minutes |

**Scripts**

| Entity | Purpose |
|---|---|
| `script.set_fan_to_min` | Press Fan DOWN until MIN LED is on |
| `script.set_fan_to_normal` | Press until NORMAL LED is on |
| `script.set_fan_to_max_2` | Press Fan UP until MAX LED is on |
| `script.set_temp_to_0` … `script.set_temp_to_5` | Press Temp UP/DOWN until the target level sensor matches |
| `script.villavent_boost_20min` | Set fan to MAX and start `timer.villavent_boost_timer` |

Each script reads the current sensor state and presses the UP or DOWN button the required number of times to reach the target level.

### Adding the Card

1. Open your dashboard in edit mode.
2. Click **Add Card → Manual**.
3. Paste the full contents of `lovelace-card.yaml` (excluding the comment header lines starting with `##`).

## Hardware Notes

- Use a **current-limiting resistor** in series with each optocoupler LED side, sized for the LED voltage on the VR 400/E3 panel.
- The output optocouplers (button simulation) switch the collector/emitter across the original button contacts.
- The input optocouplers (LED reading) have their LED side across the status LED, and the output side connected to the ESP32 GPIO and GND.
- The ESP32 and the ventilation unit share **no common ground** — the optocouplers provide full galvanic isolation.

## License

MIT
