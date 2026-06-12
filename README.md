# CYD Home Assistant Control Display

ESPHome firmware that turns a **Cheap Yellow Display (CYD / ESP32-2432S028)** into a
wall-mounted control panel for [Home Assistant](https://www.home-assistant.io/). It shows
live power consumption and provides three touch buttons that toggle Home Assistant switches,
plus an audible over-power alarm.

![platform: ESPHome](https://img.shields.io/badge/platform-ESPHome%202026.5-blue)
![board: ESP32-2432S028](https://img.shields.io/badge/board-ESP32--2432S028-yellow)

---

## Features

- **Live power readout** — displays `sensor.your_power` in watts, large and centered.
  - Label turns **green below 200 W** and **red at/above 200 W**.
- **Three toggle buttons** laid out side by side (tall, easy-to-hit rectangles):
  | Button | Action |
  |--------|--------|
  | Garage OPENEN | toggles `switch.your_garage_open` |
  | Garage SLUITEN | toggles `switch.your_garage_close` |
  | Deurslot WACHTKAMER | toggles `switch.your_door_lock` |
- **Over-power alarm** — short repeating beep while power stays **above 4000 W** for 5 s
  (5 s debounce), via the onboard speaker.

## Hardware

| | |
|---|---|
| **Board** | ESP32-2432S028 ("Cheap Yellow Display"), ESP32 dual-core @ 240 MHz |
| **Display** | 2.8" 320×240 ILI9341, SPI |
| **Touch** | XPT2046 resistive, separate SPI bus |
| **Audio** | Speaker/amp on GPIO26 |
| **Backlight** | GPIO21 |

### Pinout used

| Function | Pins |
|----------|------|
| Display SPI | CLK `GPIO14`, MOSI `GPIO13`, MISO `GPIO12`, CS `GPIO15`, DC `GPIO2` |
| Touch SPI | CLK `GPIO25`, MOSI `GPIO32`, MISO `GPIO39`, CS `GPIO33`, IRQ `GPIO36` |
| Backlight | `GPIO21` (LEDC PWM) |
| Speaker | `GPIO26` (LEDC PWM) |

## Repository layout

| File | Purpose |
|------|---------|
| `cyd-ha-control.yaml` | Main ESPHome firmware config |
| `secrets.yaml.example` | Template for all site-specific values (Wi-Fi, API key, HA entity IDs) |
| `secrets.yaml` | **Not committed** — your real secrets + entity IDs (git-ignored) |
| `CYD-HA-Control-SPEC.md` | Original functional specification |
| `CLAUDE.md` | Project notes, key technical decisions, and gotchas |
| `changelog.md` | Development changelog |

## Getting started

### 1. Prerequisites

- Python with [ESPHome](https://esphome.io/) installed (developed against **ESPHome 2026.5.3**):
  ```bash
  python3 -m venv ~/esphome-cyd-venv
  source ~/esphome-cyd-venv/bin/activate
  pip install esphome
  ```
- A Home Assistant instance with the ESPHome integration.

### 2. Configure secrets

Copy the template and fill in your values:

```bash
cp secrets.yaml.example secrets.yaml
```

`secrets.yaml` is the **single source of truth** for every site-specific value — Wi-Fi, the
API key, *and* the Home Assistant entity IDs the panel reads/controls. The firmware pulls
them via ESPHome `!secret`, so `cyd-ha-control.yaml` itself contains no site-specific data:

```yaml
wifi_ssid: "your-wifi-ssid"
wifi_password: "your-wifi-password"
api_encryption_key: "base64-32-byte-key"   # must match HA's esphome secrets

power_entity: sensor.your_power            # sensor shown on screen (reports kW)
garage_open_switch: switch.your_garage_open
garage_close_switch: switch.your_garage_close
doorlock_switch: switch.your_door_lock
```

> The `api_encryption_key` must match the key Home Assistant has for this device
> (HA stores it in `/config/esphome/secrets.yaml`). If you let ESPHome generate a new key,
> update HA to match, or the API won't connect.

### 3. First flash (USB)

Connect the CYD over USB and flash once over serial:

```bash
esphome run cyd-ha-control.yaml --device /dev/ttyUSB0
```

### 4. Subsequent updates (OTA)

After the first flash, the device joins Wi-Fi and accepts over-the-air updates:

```bash
esphome run cyd-ha-control.yaml --device <device-ip>
```

### 5. Adopt in Home Assistant

The device should appear in **Settings → Devices & Services → ESPHome**. Adopt it, then —
importantly — enable the per-device toggle **"Allow the device to perform Home Assistant
actions"**, otherwise the buttons' service calls are silently dropped (see gotchas below).

## Configuration

**HA entities** (the power sensor and the three switches) are set in `secrets.yaml` — see
above. To point the panel at different entities, just edit those values.

**Display/alarm tunables** live in the `substitutions:` block at the top of
`cyd-ha-control.yaml`:

| Substitution | Default | Meaning |
|--------------|---------|---------|
| `color_threshold` | `200` | Watts; label green below, red at/above |
| `buzzer_threshold` | `4000` | Watts; alarm beeps above this (5 s debounce) |

## Key technical decisions & gotchas

These were hard-won during bring-up — see `CLAUDE.md` for the full story.

- **Display driver: use `model: ILI9342` with the `ili9xxx` platform.** It shares the
  ILI9341 init sequence but defaults to 320×240 landscape, giving correct orientation with
  LVGL `rotation: 0` and **no** display transform. The `mipi_spi` platform produces garbled
  output on this board regardless of configuration.
- **Touch transform: `swap_xy: true, mirror_x: true, mirror_y: true`.** A wrong `mirror_x`
  is invisible while widgets are full-width-stacked (only the Y axis selects a widget) — it
  only shows up with side-by-side widgets as flipped left/right taps.
- **Power sensor reports kW, not W** — every comparison and label multiplies by 1000.
- **Speaker uses `ledc` output (not `esp32_dac`)** — the `rtttl` component requires it.
- **`buffer_size: 25%`** — the CYD has no PSRAM.
- **HA-side permission:** ESPHome *sending* a `homeassistant.action` is not the same as HA
  *executing* it — the device's "Allow the device to perform Home Assistant actions" toggle
  in the ESPHome integration must be ON.

## License

Personal project; no warranty. Use at your own risk.
