# CYD Home Assistant Control Display

## Overview
ESPHome firmware for a Cheap Yellow Display (CYD / ESP32-2432S028) acting as a wall control panel for Home Assistant.

## Hardware
- **Board**: ESP32-2432S028 ("Cheap Yellow Display"), ESP32 dual-core, 240 MHz
- **Display**: 2.8" 320×240 ILI9341, SPI
- **Touch**: XPT2046 resistive, separate SPI bus
- **Audio**: Speaker amp on GPIO26
- **Backlight**: GPIO21

## Project Files
- `cyd-ha-control.yaml` — Main ESPHome firmware config
- `secrets.yaml` — Wi-Fi credentials + API encryption key (matches HA's `/config/esphome/secrets.yaml`)
- `CYD-HA-Control-SPEC.md` — Original functional specification

## Key Technical Decisions

### Display Driver
- **Model: ILI9342** (not ILI9341) — same chip/init sequence but default 320×240 landscape dimensions
- `ili9xxx` platform (NOT `mipi_spi` — mipi_spi causes garbled output on this board)
- No `rotation` or `transform` needed — the ILI9342 model + LVGL `rotation: 0` gives correct landscape orientation
- LVGL blocks `rotation` in display config — use `model: ILI9342` to avoid needing it
- `buffer_size: 25%` (no PSRAM on CYD)

### Speaker
- Uses `ledc` output on GPIO26 (NOT `esp32_dac` — rtttl doesn't support esp32_dac)
- RTTTL gain at 60%

### Power Sensor
- `sensor.your_power` reports in **kW** (not W) — multiplied by 1000 in lambdas
- Two independent thresholds (both compared after ×1000):
  - `color_threshold` = **200 W** — label green below, red at/above
  - `buzzer_threshold` = **4000 W** — alarm beeps above this (5 s debounce)

### Touch (XPT2046)
- Transform: `swap_xy: true, mirror_x: true, mirror_y: true` — correct combo for this PCB
- A wrong `mirror_x` stays hidden while widgets are full-width-stacked (only Y selects the
  widget); it only surfaces with side-by-side widgets, as flipped left/right taps
- This is a *touch* transform, separate from the display (the display needs no transform)

### Home Assistant
- HA runs on Proxmox VM at `<HA_IP>`
- ESPHome secrets at `/config/esphome/secrets.yaml` on HA
- CYD adopted in HA ESPHome integration, API connected
- Wi-Fi SSID: `<your-wifi-ssid>`

## Current Status (2026-06-12)
- ✅ Display: correct orientation, full 320×240, clear image
- ✅ Power readout: live kW→W conversion working
- ✅ Wi-Fi + HA API: connected and stable
- ✅ OTA: working at <CYD_IP>
- ✅ **Buttons: firmware confirmed working** — diagnostic build with `logger.log` on `on_press`/`on_release`/`on_short_click`/`on_click` proved the full chain fires on a firm tap, and `homeassistant.action` runs to completion ("action sent" logged). Earlier "no log output" was a false alarm: that instrumented build had never actually been flashed+tapped. Buttons use `on_short_click` and call `switch.toggle` (one tap flips state, matching the HA dashboard — they are NOT momentary pushes). **Caveat:** ESPHome sending the action ≠ HA executing it — verify the per-device "Allow the device to perform Home Assistant actions" toggle is ON (HA → Settings → Devices → ESPHome → CYD), and confirm the switch entities (`switch.your_garage_open/sluiten`, `switch.your_door_lock`) physically respond.
- ✅ Touch: healthy — calibration in-range and (after `mirror_x: true`) left/right taps route to the correct side-by-side button.
- ✅ UI: three buttons laid out side by side (tall rectangles) across the lower screen; power label green <200 W / red ≥200 W.
- ✅ Speaker/beep: working — boot-chime test (rtttl on GPIO26 LEDC, gain 60%) produced audible sound, log confirmed the LEDC note ramp. The over-threshold `beep_loop` alarm uses the same `rtttl.play` path so the mechanism is proven; the live >4000 W alarm hasn't been field-triggered yet (power currently reads negative).

## ESPHome Environment
- Venv: `~/esphome-cyd-venv`
- Version: ESPHome 2026.5.3
- Serial port: `/dev/ttyUSB0` (first flash only, then OTA)
