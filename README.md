# cc1101-transceiver
ESPHome config for CC1101 RF Transceiver

A complete ESPHome configuration for the **CC1101 Sub-1 GHz RF Transceiver** with an **ESP32-C3**, featuring signal learning, replay, and Home Assistant integration.

Learn and replay 433 MHz RF signals from remotes, blinds, outlets, and more — directly from Home Assistant.

---

## Features

- **Learn RF signals** — Capture signals from any 433 MHz remote
- **Replay signals** — Retransmit learned signals on demand (single or x3)
- **Single-line log output** — Captured codes logged as copyable arrays
- **Dual pin wiring** — Simultaneous TX/RX without mode switching
- **Home Assistant integration** — Full control via buttons, switches, and sensors
- **Web interface** — Built-in web server on port 80 for standalone use

---

## Hardware Requirements

| Component | Description |
|-----------|-------------|
| ESP32-C3-DevKitM-1 | Espressif ESP32-C3 development board (or any ESP32-C3 board) |
| CC1101 Module | 433 MHz version (common green or blue PCB modules) |
| Jumper Wires | 8 female-to-female dupont wires |
| Antenna | 433 MHz antenna (17.3 cm wire or SMA antenna) |

> **Note:** The ESP32-C3-DevKitM-1 is recommended. Other ESP32-C3 boards (e.g. SuperMini, XIAO) should also work — verify your board's available GPIO pinout before wiring.

---

## Wiring Diagram

### Dual Pin Mode (Recommended)

Uses separate pins for TX and RX — no mode switching needed.

```
 ESP32-C3-DevKitM-1         CC1101 Module
 ┌──────────┐              ┌──────────────┐
 │          │              │              │
 │     3V3  ├──── Red ─────┤ VCC          │
 │     GND  ├──── Black ───┤ GND          │
 │          │              │              │
 │   GPIO5  ├──── Orange ──┤ MOSI (SI)    │
 │   GPIO3  ├──── Yellow ──┤ MISO (SO)    │
 │   GPIO4  ├──── Green ───┤ SCK (SCLK)   │
 │   GPIO0  ├──── Purple ──┤ CSN (CS)     │
 │          │              │              │
 │   GPIO1  ├──── Blue ────┤ GDO0 (TX)    │
 │   GPIO2  ├──── Cyan ────┤ GDO2 (RX)    │
 │          │              │              │
 └──────────┘              └──────────────┘
```

| Wire | ESP32-C3 Pin | CC1101 Pin | Function |
|------|-------------|------------|----------|
| Red | 3V3 | VCC | Power (3.3V only!) |
| Black | GND | GND | Ground |
| Orange | GPIO5 | MOSI | SPI Data Out |
| Yellow | GPIO3 | MISO | SPI Data In |
| Green | GPIO4 | SCK | SPI Clock |
| Purple | GPIO0 | CSN | SPI Chip Select |
| Blue | GPIO1 | GDO0 | TX Data |
| Cyan | GPIO2 | GDO2 | RX Data |

> ⚠️ The CC1101 is a **3.3V device**. Do not connect to 5V — it will damage the module.

> **Why these pins?** On ESP32-C3, GPIO6–GPIO11 are internally connected to the SPI flash chip and must never be used for external peripherals. GPIO0–GPIO5 are the safest, most accessible alternative. See [ESP32-C3 Pin Notes](#esp32-c3-pin-notes) for details.

> **GPIO2 note:** GPIO2 is a strapping pin on ESP32-C3, but it only controls JTAG debug port selection — it does **not** affect boot mode. It is safe to use for GDO2 in normal operation. If you experience issues, substitute GPIO10 (check your board's pinout first).

Pin numbering may vary between modules. Always verify with your module's datasheet.

---

## ESP32-C3 Pin Notes

### Why GPIO6–GPIO11 Are Forbidden

The ESP32-C3 routes its internal SPI flash through GPIO6–GPIO11 on the chip. Using any of these for an external peripheral will corrupt flash access, cause crashes, or prevent boot entirely.

| GPIO Range | Status |
|-----------|--------|
| GPIO0–GPIO5 | ✅ Safe — general purpose, broken out on DevKitM-1 header |
| GPIO6–GPIO11 | ❌ Reserved — connected to SPI flash chip internally |
| GPIO12–GPIO21 | ✅ Safe — general purpose (GPIO18/GPIO19 are USB D-/D+ on DevKitM-1) |

### Strapping Pins

ESP32-C3 samples three pins at boot to decide operating mode. If a connected peripheral drives these pins at an unexpected level during power-up it can affect boot or debugging.

| GPIO | Strapping Function | Impact | Safe as GDO2/CS? |
|------|-------------------|--------|-----------------|
| GPIO2 | JTAG source selection | Affects debug port only — **not boot mode** | ✅ Yes |
| GPIO8 | ROM serial messages | Cosmetic only | ✅ Yes |
| GPIO9 | Boot mode (HIGH = normal, LOW = download) | **Can enter download mode** | ⚠️ Avoid |

This config uses GPIO2 for GDO2 (RX). CC1101's GDO2 goes low after power-up by default (chip-ready signal). This sets JTAG to the USB-JTAG interface — harmless for normal use; only relevant if you use hardware JTAG debugging.

### Flashing

The ESP32-C3-DevKitM-1 has two USB ports:

| Port | Chip | Use |
|------|------|-----|
| **UART** (most boards: left port) | CP2102N USB-to-UART | First-time flashing, serial logs |
| **USB** (most boards: right port) | Built-in USB-JTAG | Alternative flashing, JTAG debugging |

Connect via the **UART port** for `esphome run` or initial flashing. After first flash, OTA updates work over WiFi.

---

## Installation

### Prerequisites

- [ESPHome](https://esphome.io/) installed (via Home Assistant Add-on or standalone)
- ESP32-C3 board with USB cable (connect to the UART/CP2102N port for first flash)
- CC1101 module wired to ESP32-C3 per the table above


## Usage

### Learning a Signal

1. Press **"Learn Signal"** in Home Assistant (or toggle **"Learning Mode"** on)
2. Status shows *"Waiting for signal... Press remote now!"*
3. Press the button on your RF remote near the CC1101
4. Status updates to *"Signal learned! X pulses captured"*

### Replaying a Signal

- **"Replay Learned Signal"** — Transmit once
- **"Replay x3"** — Transmit 3 times with 50ms gaps (more reliable for some devices)

### Clearing the Signal

- Press **"Clear Learned Signal"** to reset and learn a new one

> **Note:** The learned signal is stored in RAM. It will be lost on reboot. See [Adding Permanent Buttons](#adding-permanent-buttons) to save signals permanently.

### Copying Raw Codes from Logs

Every received signal is logged as a single copyable line in the ESPHome console:

```
[I][raw_code:xxx]: Pulses: 122
[I][raw_code:xxx]: Copy this line:
[I][raw_code:xxx]: [592, -635, 266, -295, 617, -615, 268, -309, ...]
```

You can paste this array directly into the YAML config as a permanent button.

---

## Home Assistant Entities

### Buttons

| Entity | Icon | Description |
|--------|------|-------------|
| Learn Signal | `mdi:record-rec` | Start learning mode |
| Replay Learned Signal | `mdi:replay` | Transmit learned signal once |
| Replay x3 | `mdi:replay` | Transmit 3 times with 50ms gaps |
| Clear Learned Signal | `mdi:delete` | Clear signal from memory |
| TX Test | `mdi:access-point` | Send a test signal |
| Restart | | Reboot the ESP32-C3 |

### Switch

| Entity | Icon | Description |
|--------|------|-------------|
| Learning Mode | `mdi:school` | Toggle learning on/off |

### Sensors

| Entity | Type | Description |
|--------|------|-------------|
| Learn Status | Text | Current status message |
| Last Received Signal | Text | Preview of last captured signal |
| Learned Signal Length | Number | Pulse count of learned signal |
| Signal Learned | Binary | Whether a signal is currently stored |

---

## Adding Permanent Buttons

Once you've captured a signal from the logs, add it to the `button:` section in your YAML to make it permanent:

```yaml
- platform: template
  name: "Living Room Blinds Up"
  icon: "mdi:blinds-open"
  on_press:
    - remote_transmitter.transmit_raw:
        carrier_frequency: 0Hz
        code: [592, -635, 266, -295, 617, -615, ...]

- platform: template
  name: "Living Room Blinds Down"
  icon: "mdi:blinds"
  on_press:
    - remote_transmitter.transmit_raw:
        carrier_frequency: 0Hz
        code: [605, -622, 280, -310, ...]
```

You can also wrap them in a `repeat` for reliability:

```yaml
- platform: template
  name: "Garage Light"
  icon: "mdi:lightbulb"
  on_press:
    - repeat:
        count: 3
        then:
          - remote_transmitter.transmit_raw:
              carrier_frequency: 0Hz
              code: [500, -500, 500, -500, ...]
          - delay: 50ms
```

---

## Automation Examples

### Close Blinds at Sunset

```yaml
automation:
  - alias: "Close Blinds at Sunset"
    trigger:
      - platform: sun
        event: sunset
    action:
      - button.press:
          entity_id: button.cc1101_rf_transceiver_living_room_blinds_down
```

### Toggle Lights via Motion Sensor

```yaml
automation:
  - alias: "RF Light on Motion"
    trigger:
      - platform: state
        entity_id: binary_sensor.hallway_motion
        to: "on"
    action:
      - button.press:
          entity_id: button.cc1101_rf_transceiver_hallway_light
```

---

## Configuration Reference

### CC1101 Settings

Adjust in the `cc1101:` section of the YAML:

| Setting | Value | Description |
|---------|-------|-------------|
| `frequency` | `433.92MHz` | Operating frequency (300–928 MHz) |
| `output_power` | `10` | TX power in dBm (-30 to +11) |
| `modulation_type` | `ASK/OOK` | Modulation scheme |
| `symbol_rate` | `5000` | Symbol rate in Baud |
| `filter_bandwidth` | `200kHz` | Receive filter bandwidth |

### Remote Receiver Tuning

| Setting | Value | Description |
|---------|-------|-------------|
| `tolerance` | `50%` | How much pulse timing can deviate |
| `filter` | `200us` | Ignore pulses shorter than this |
| `idle` | `10ms` | Gap that marks end of a signal |

**Tuning tips:**
- Too much noise → increase `filter` to `500us` or reduce `filter_bandwidth`
- Missing signals → increase `tolerance` or `idle`

---

## Troubleshooting

### CC1101 Not Detected

- **"FF0F was found" error** — SPI wiring issue. Double-check MISO (GPIO3), MOSI (GPIO5), SCK (GPIO4), and CSN (GPIO0) connections.
- Verify you're using **3.3V**, not 5V.
- Use shorter jumper wires — long wires cause SPI failures.
- On ESP32-C3: double-check that none of the wires are accidentally on GPIO6–GPIO11 (they share labels with adjacent header pins).

### Not Receiving Signals

- Confirm the remote operates on **433.92 MHz** (some use 315 or 868 MHz).
- Attach a proper antenna — a **17.3 cm** straight wire works for 433 MHz.
- On ESP32-C3: verify none of your wires are connected to GPIO6–GPIO11 (reserved for flash).
- Try increasing `filter_bandwidth` to capture wider frequency deviations.

### Signals Captured But Replay Doesn't Work

- Use **"Replay x3"** — some devices need multiple transmissions.
- Move the ESP32 + CC1101 **closer** to the target device.
- Increase `output_power` to `11` (maximum).
- **Rolling code devices** (garage doors, car key fobs) cannot be replayed — each press generates a unique code.

### Too Much Noise in Logs

- Increase `filter` from `200us` to `500us`
- Reduce `filter_bandwidth` from `200kHz` to `100kHz`
- Move the antenna away from the ESP32 board and USB cable

---

## Compatible Devices

| ✅ Works (Fixed Code) | ❌ Won't Work (Rolling/Encrypted) |
|------------------------|-----------------------------------|
| Motorized blinds/shades | Car key fobs |
| RF power outlets | Most garage door openers |
| Ceiling fan remotes | Security alarm systems |
| Doorbells | |
| RF light switches | |
| Fixed-code gate remotes | |


---

## References

- [ESPHome CC1101 Component](https://esphome.io/components/cc1101/)
- [ESPHome Remote Receiver](https://esphome.io/components/remote_receiver/)
- [ESPHome Remote Transmitter](https://esphome.io/components/remote_transmitter/)
- [CC1101 Datasheet (TI)](https://www.ti.com/lit/ds/symlink/cc1101.pdf)
- [ESP32-C3-DevKitM-1 Getting Started Guide](https://docs.espressif.com/projects/esp-idf/en/latest/esp32c3/hw-reference/esp32c3/user-guide-devkitm-1.html)
- [ESP32-C3 Technical Reference Manual](https://www.espressif.com/sites/default/files/documentation/esp32-c3_technical_reference_manual_en.pdf)


---

## License

MIT License — free to use, modify, and share.














