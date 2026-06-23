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
  `doorlock_switch`, `claude_5h_entity`, `claude_reset_entity`, `zai_5h_entity`, `zai_reset_entity`);
  docs use placeholders (`<HA_IP>`, `<CYD_IP>`, `<your-wifi-ssid>`,
  `switch.your_garage_open`, etc.). Real values live only in local `secrets.yaml`.
- This is a plain (Syncthing-synced) dir with a git repo + GitHub remote — commit/push on changes.

## Key Technical Decisions

### Display Driver
- **Model: ILI9342** (not ILI9341) — same chip/init sequence but default 320×240 landscape dimensions
- `ili9xxx` platform (NOT `mipi_spi` — mipi_spi causes garbled output on this board)
- No `rotation` or `transform` needed — the ILI9342 model + LVGL `rotation: 0` gives correct landscape orientation
- LVGL blocks `rotation` in display config — use `model: ILI9342` to avoid needing it
- `buffer_size: 25%` (no PSRAM on CYD)
- **Colours are written BGR (R/B byte-swapped)** — the ILI9342 model sets `MADCTL=0x48` (BGR bit), so the panel swaps red/blue; the `col_*` substitutions pre-swap R/B to compensate. `color_order: BGR` has NO effect on the LVGL direct-buffer path (verified empirically — it left the output unchanged), so the swap must be handled in the colour values themselves. Without this, blue renders orange and red renders blue; green looked fine (nearly swap-symmetric), which hid the bug until blue/red were actually used (2026-06-13). The same BGR rule applies to the AI-row brand colours: `col_claude` (true `0xD97757` Claude clay → stored `0x5777D9`) and `col_zai` (true `0x1F63EC` z.ai blue → stored `0xEC631F`).

### Speaker
- Uses `ledc` output on GPIO26 (NOT `esp32_dac` — rtttl doesn't support esp32_dac)
- RTTTL gain at 10%

### Doorbell chime (2026-06-23)
- The CYD chimes a 3× ding-dong (~3.25 s) when any of the 4 front-door doorbell buttons on ESP-209 (`binary_sensor.your_doorbell_1..4`) is pressed.
- **Driven from HA, not self-subscribed**: a template `button` "Doorbell Chime" in the firmware does `rtttl.play` on press (`Doorbell:d=4,o=5,b=120: c6,g5,16p,c6,g5,16p,c6,g5` — C6 "ding" → G5 "dong", perfect 4th, ×3); HA's doorbell automation (id `1686663509022` 'TTS: deurbel ingedrukt') presses it as its **first action** so the chime fires immediately (before the ESP-221 MP3 + delays).
- HA button entity is `button.your_cyd_doorbell_chime` — the device is registered in HA as `<room>_cyd_control_display` (NOT `cyd_control_…`); confirm via `list_entities`, don't assume.
- **Additive** — ESP-221's doorbell MP3, Telegram, and the Dutch TTS in the same automation are unchanged.
- The CYD amp is RTTTL-only (no MP3), so it can't replay the ESP-221 doorbell `.mp3`; it plays a generated melody instead.
- Caveat: needs ESP-209 online to trigger on a real press (ESP-209 was offline 2026-06-23); press `button.your_cyd_doorbell_chime` directly from HA to test the chime independently.

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

### AI usage row (2026-06-19)
- 4 display-only `obj` cells between the power card and the buttons: `cc 5h` / `cc reset` (Claude clay `#D97757`, `col_claude`) and `z.ai 5h` / `z.ai reset` (z.ai blue `#1F63EC`, `col_zai`). Each provider gets a 5h-% cell + a reset-countdown cell. Brand colours sourced from the real assets (Anthropic skills repo; z.ai `logo.svg`), both written BGR (see Display Driver).
- The two `%` cells import HA usage sensors directly (`claude_5h_entity`, `zai_5h_entity`).
- **Reset cells need a computed value:** ESPHome's `homeassistant` sensor imports timestamp entities as an **ISO string**, not a usable number — so the countdowns are **HA template sensors** (`claude_reset_entity`, `zai_reset_entity`) outputting minutes-to-reset via `((states(...)|as_datetime - now()).total_seconds()/60)|round`. Defined in a YAML package (`packages/ai_usage_countdowns.yaml`, on the HA config mount + mirrored in the `hass-ai-usage-monitoring` project) — packages load only at startup, so creating/editing needs an HA restart (disarm the alarm first; alarm + Telegram + Frigate live on this box). On restart the CYD reconnects and re-subscribes, so the cells populate immediately. ESPHome formats minutes as `HuMM` (e.g. `1u23`); `isnan(x)` guard → `--`.
- Cells are display-only (not buttons) → touch routing is unaffected.
- **LVGL `obj` padding gotcha:** the cells need `pad_all: 0`. LVGL `obj` widgets have default padding that both shrinks the area children align into (so a `TOP_MID` value and a `BOTTOM_MID` caption collide) and draws a grey auto-scrollbar band along the bottom. `pad_all: 0` kills both. Final cell layout: 74×58, value `montserrat_24` at `TOP_MID`, caption `montserrat_14` at `BOTTOM_MID`.

## Current Status (2026-06-23)
- ✅ Display: correct orientation, full 320×240, clear image
- ✅ Power readout: live kW→W conversion working
- ✅ Wi-Fi + HA API: connected and stable
- ✅ OTA: working at <CYD_IP>
- ✅ **Buttons: firmware confirmed working** — diagnostic build with `logger.log` on `on_press`/`on_release`/`on_short_click`/`on_click` proved the full chain fires on a firm tap, and `homeassistant.action` runs to completion ("action sent" logged). Earlier "no log output" was a false alarm: that instrumented build had never actually been flashed+tapped. Buttons use `on_short_click` and call `switch.toggle` (one tap flips state, matching the HA dashboard — they are NOT momentary pushes). **Caveat:** ESPHome sending the action ≠ HA executing it — verify the per-device "Allow the device to perform Home Assistant actions" toggle is ON (HA → Settings → Devices → ESPHome → CYD), and confirm the switch entities (`switch.your_garage_open/close`, `switch.your_door_lock`) physically respond.
- ✅ Touch: healthy — calibration in-range and (after `mirror_x: true`) left/right taps route to the correct side-by-side button.
- ✅ UI: three buttons laid out side by side (tall rectangles) across the lower screen; power label green <200 W / red ≥200 W.
- ✅ Speaker/beep: working — boot-chime test (rtttl on GPIO26 LEDC, gain 60%) produced audible sound, log confirmed the LEDC note ramp. The over-threshold `beep_loop` alarm uses the same `rtttl.play` path so the mechanism is proven; the live >3000 W alarm hasn't been field-triggered yet (power currently reads negative).
- ✅ UI & tuning (2026-06-13): **dark theme** (screen `0x101418` + card panel `0x1A2028`); power value **3-tier** — green <200 W, blue 200–3000 W, red ≥3000 W; **colours written BGR** to compensate for the ILI9342 MADCTL red/blue swap (`color_order` is a no-op on the LVGL path — see Display Driver); buttons recolored to Home Assistant blue (`bg_color 0x03A9F4` → grad `0x0288D1`); door-lock label is now `Deurslot WACHT KAMER` (added space wraps cleanly). `buzzer_threshold` lowered 4000→3000 W; rtttl `gain` lowered 60%→10%.
- ✅ AI usage row (2026-06-19): 4-cell row (`cc 5h`/`cc reset` Claude-clay + `z.ai 5h`/`z.ai reset` z.ai-blue) inserted between the power card and the buttons; power card dropped its "Verbruik" label and shrank 96→64 px, buttons shifted down (y 110→122). `%` cells import HA usage sensors; reset cells import HA template sensors (minutes-to-reset). Cells are display-only — touch unaffected. See "AI usage row".
- ✅ Doorbell chime (2026-06-23): CYD plays a 3× ding-dong (~3.25 s) on doorbell press — template `button` "Doorbell Chime" in firmware + `button.press` as the first action in the 'TTS: deurbel ingedrukt' automation (id `1686663509022`). Additive — ESP-221 MP3, Telegram, and TTS kept. User confirmed audible. Pending: ESP-209 is currently offline, so real-press triggers wait on its return (test the chime by pressing `button.your_cyd_doorbell_chime` directly).

## ESPHome Environment
- Venv: `~/esphome-cyd-venv`
- Version: ESPHome 2026.5.3
- Serial port: `/dev/ttyUSB0` (first flash only, then OTA)
