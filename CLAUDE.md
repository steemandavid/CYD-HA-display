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
- `secrets.yaml` — **single source of truth for ALL site-specific values** (git-ignored): Wi-Fi, API key, HA entity IDs, and IPs. Matches HA's `/config/esphome/secrets.yaml` for the API key.
- `secrets.yaml.example` — sanitized template committed in place of the real secrets
- `CYD-HA-Control-SPEC.md` — Original functional specification

## Public Repo / Secrets Convention
- Repo is **public** at https://github.com/steemandavid/CYD-HA-display
- No committed file may contain real site-specific data. The firmware pulls entities via
  ESPHome `!secret` (`!secret power_entity`, `garage_open_switch`, `garage_close_switch`,
  `doorlock_switch`); docs use placeholders (`<HA_IP>`, `<CYD_IP>`, `<your-wifi-ssid>`,
  `switch.your_garage_open`, etc.). Real values live only in local `secrets.yaml`.
- This is a plain (Syncthing-synced) dir with a git repo + GitHub remote — commit/push on changes.

## Key Technical Decisions

### Display Driver
- **Model: ILI9342** (not ILI9341) — same chip/init sequence but default 320×240 landscape dimensions
- `ili9xxx` platform (NOT `mipi_spi` — mipi_spi causes garbled output on this board)
- No `rotation` or `transform` needed — the ILI9342 model + LVGL `rotation: 0` gives correct landscape orientation
- LVGL blocks `rotation` in display config — use `model: ILI9342` to avoid needing it
- `buffer_size: 25%` (no PSRAM on CYD)
- **Colours are written BGR (R/B byte-swapped)** — the ILI9342 model sets `MADCTL=0x48` (BGR bit), so the panel swaps red/blue; the `col_*` substitutions pre-swap R/B to compensate. `color_order: BGR` has NO effect on the LVGL direct-buffer path (verified empirically — it left the output unchanged), so the swap must be handled in the colour values themselves. Without this, blue renders orange and red renders blue; green looked fine (nearly swap-symmetric), which hid the bug until blue/red were actually used (2026-06-13).

### Speaker
- Uses `ledc` output on GPIO26 (NOT `esp32_dac` — rtttl doesn't support esp32_dac)
- RTTTL gain at 10%

### Power Sensor
- `sensor.your_power` reports in **kW** (not W) — multiplied by 1000 in lambdas
- Two independent thresholds (both compared after ×1000):
  - `color_threshold` = **200 W** — power value green below this, blue up to `buzzer_threshold` (3000 W), then red (3-tier)
  - `buzzer_threshold` = **3000 W** — alarm beeps above this (5 s debounce)

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

## Current Status (2026-06-13)
- ✅ Display: correct orientation, full 320×240, clear image
- ✅ Power readout: live kW→W conversion working
- ✅ Wi-Fi + HA API: connected and stable
- ✅ OTA: working at <CYD_IP>
- ✅ **Buttons: firmware confirmed working** — diagnostic build with `logger.log` on `on_press`/`on_release`/`on_short_click`/`on_click` proved the full chain fires on a firm tap, and `homeassistant.action` runs to completion ("action sent" logged). Earlier "no log output" was a false alarm: that instrumented build had never actually been flashed+tapped. Buttons use `on_short_click` and call `switch.toggle` (one tap flips state, matching the HA dashboard — they are NOT momentary pushes). **Caveat:** ESPHome sending the action ≠ HA executing it — verify the per-device "Allow the device to perform Home Assistant actions" toggle is ON (HA → Settings → Devices → ESPHome → CYD), and confirm the switch entities (`switch.your_garage_open/close`, `switch.your_door_lock`) physically respond.
- ✅ Touch: healthy — calibration in-range and (after `mirror_x: true`) left/right taps route to the correct side-by-side button.
- ✅ UI: three buttons laid out side by side (tall rectangles) across the lower screen; power label green <200 W / red ≥200 W.
- ✅ Speaker/beep: working — boot-chime test (rtttl on GPIO26 LEDC, gain 60%) produced audible sound, log confirmed the LEDC note ramp. The over-threshold `beep_loop` alarm uses the same `rtttl.play` path so the mechanism is proven; the live >3000 W alarm hasn't been field-triggered yet (power currently reads negative).
- ✅ UI & tuning (2026-06-13): **dark theme** (screen `0x101418` + card panel `0x1A2028`); power value **3-tier** — green <200 W, blue 200–3000 W, red ≥3000 W; **colours written BGR** to compensate for the ILI9342 MADCTL red/blue swap (`color_order` is a no-op on the LVGL path — see Display Driver); buttons recolored to Home Assistant blue (`bg_color 0x03A9F4` → grad `0x0288D1`); door-lock label is now `Deurslot WACHT KAMER` (added space wraps cleanly). `buzzer_threshold` lowered 4000→3000 W; rtttl `gain` lowered 60%→10%.

## ESPHome Environment
- Venv: `~/esphome-cyd-venv`
- Version: ESPHome 2026.5.3
- Serial port: `/dev/ttyUSB0` (first flash only, then OTA)
