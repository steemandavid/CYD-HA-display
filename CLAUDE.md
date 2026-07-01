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
  `doorlock_switch`, `claude_5h_entity`, `claude_reset_entity`, `zai_5h_entity`, `zai_reset_entity`,
  `temp_outside_entity`, `temp_inside_entity`);
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

### Multi-page UI + Temperature page (2026-06-29)
- The display is **multi-page** (4 pages; `temp_page` is now the **last**): `main_page` (power + AI row + gate buttons), then `energy_page`, `devices_page`, and `temp_page` (two side-by-side temperature cards). A **"Pagina" button on the `top_layer`** (always-on-top transparent page; top-right 74×64) is visible/tappable on *every* page; `on_short_click` → `lvgl.page.next` cycles to the next page. Adding a page = append to `lvgl: pages:`; the button keeps cycling through all of them. (`top_layer` is the cookbook's pattern to avoid repeating a nav widget on each page.)
- **Page-flip actions:** ESPHome LVGL exposes `lvgl.page.next:` / `lvgl.page.previous:` / `lvgl.page.show: <id>`. **Verify:** `lvgl.page.next` must wrap from the last page back to the first — if it doesn't, the last page's button is dead and we switch to an explicit `lvgl.page.show`.
- **Page 4 / last (`temp_page`):** `Buiten` (outside, `temp_outside_entity` = `sensor.your_outside_temperature`) and `Binnen` (inside, `temp_inside_entity` = `sensor.your_inside_temperature`) side by side — two 152×232 `col_card` panels (6-px margins, 4-px gap → 6+152+4+152+6 = 320). Value `montserrat_40` white, `align: CENTER` (sits mid-card, *below* the top-right Page button so no overlap; 48 was too wide for `-5.0°` in a 152 px card). Label `montserrat_14` `col_label` at `TOP_LEFT` to clear the button. Format `%.1f°`; `isnan(x)` → `--°`.
- **Power card shrunk** 300→230 wide, left-aligned (`align: TOP_LEFT, x:6`) on `main_page`, to leave the top-right 74×64 clear for the Page button. `power_label` re-centres inside 230. AI row + buttons unchanged.
- **° symbol:** renders in the built-in Montserrat fonts (LVGL's default font range includes 0xB0). Only a *custom* ESPHome font would drop it — there is none here. (Cookbook thermometer example uses `"%.1f°C"` with `montserrat_48`.)
- **Font note:** `montserrat_40` is a real built-in size; ESPHome compiles a built-in font on reference. If a compile ever rejects it, `montserrat_30` (used in the cookbook) is the fallback. Only one new font added → flash is fine (was 67.8 %).
- Sensors update their `temp_page` labels via `on_value` regardless of the active page, so values are current when you flip to the temp page (same as power/AI on page 1).

### Energy & Devices pages (2026-06-29)
- Two more pages appended to `lvgl: pages:` — **page 2 `energy_page`** ("Energie": NET grid power + average, two solar power readings (SolarLog + Solarman, W), home-battery charge/discharge, two EV **stop-buttons**) and **page 3 `devices_page`** ("Apparaten": washing machine + dryer power, three AC modes). The `top_layer` Pagina button cycles **4** pages (main → Energie → Apparaten → temp → main); `temp_page` is now **last** (moved from page 2 on user request); `lvgl.page.next` wraps fine.
- **NET reuses the existing `power_combined` sensor** (`!secret power_entity`, ESPHome `id: power_combined`) — a 2nd `lvgl.label.update: id: energy_net_label` was added to its `on_value`, so one sensor feeds both the main-page W card and the Energie W readout (same ×1000 kW→W conversion). **No duplicate sensor.**
- **9 new `homeassistant` sensors** (power_avg, solar_power, solar_solarman, marstek_charge, marstek_discharge, enyaq_power, vin_power, shelly0_power, shelly1_power). **Power figures shown in W** — NET, the average, and the Solarman solar reading are kW-native ×1000 → W (matching the main-page power card); SolarLog + battery + Shelly are already W. The `ZON` card therefore shows **two live solar readings** (SolarLog + Solarman) instead of live + today. `isnan(x)` → `--` (the ID.4 EV (`vin_*` entities) is currently `unavailable` → shows `--`, button is a no-op).
- **Climates need a `text_sensor`** (firmware's first): a climate entity's state is a mode *string* (`cool`/`off`/…) that a numeric `sensor` can't hold, so 3 `text_sensor: - platform: homeassistant` import them; `on_value` (`x` is `std::string`) maps mode → short Dutch code (`KOEL`/`WARM`/`AUTO`/`VENT`/`DROOG`/`UIT`) + colour (cool=`col_mid`, heat=`col_high`, else=`col_label`). This is the string-state analogue of the AI-row timestamp gotcha — verify the platform compiled (it did; all 3 subscribed in the boot log).
- **EV stop-buttons** (page 3): red (`col_high`), each shows the charger's live kW + a `STOP` label; `on_short_click` calls `switch.turn_off` (**not** `toggle`) + `rtttl.play "beep:…16e6"` (same beep note as `beep_loop`). Direct stop, no confirm (user-chosen). Same `homeassistant.action` path as the gate buttons — the per-device "Allow the device to perform Home Assistant actions" toggle is already ON.
- **Light toggle (page 2, top-right gap — 2026-07-01):** a 74×74 square button filling the gap *under the Pagina button* and right of the ZON/BATT cards (`x:240, y:70` — matches the Pagina button's width/x, 6-px right margin). `on_short_click` → `switch.toggle` on `verlichting_vs4_switch` (`switch.verlichting_vs4_v1`, a lighting switch) — same blind-toggle pattern + `homeassistant.action` path as the gate buttons (no on/off state feedback, no beep); inherits the blue button theme. It is action-only, so like the gate buttons it produces no boot-log line.
- **Layout:** page 2 (Energie) stacks NET card (w230, clears the top-right Pagina) + ZON card + BATT card + 2 EV buttons (152×78, y158) **+ a light-toggle button in the top-right gap under the Pagina button**. Page 3 (Apparaten) = Wasmachine/Droogkast cards (152×66) + a full-width AIRCO card (308×160, 3 rows). The **right-side cards pin caption+value `TOP_LEFT`** so they clear the top-right Pagina button — same trick as `temp_page`'s Binnen card (a centred value in a short card sits under the button).
- One new BGR colour: `col_solar` (`0x00B5F5` → true `0xF5B800` amber). Battery charge=`col_low` green, discharge=`col_mid` blue. **No new fonts** (reuses `montserrat_14`/`_24`) → flash only 71.8→72.4 %.
- All entity **units were confirmed live via HA** before formatting (don't assume kW/W — `power_combined`, the average, the EVs, and the Solarman solar reading are kW; SolarLog/battery/Shelly are W). 15 new `secrets.yaml` keys (9 sensors + 2 EV switches + 3 climates) + `verlichting_vs4_switch` (light toggle, added 2026-07-01); `power_combined` reuses `power_entity`.

## Current Status (2026-06-29)
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
- ✅ Multi-page UI + temperature page (2026-06-29): display flips between `main_page` (power/AI/buttons) and a new `temp_page` (Buiten/Binnen temperatures side by side) via a `top_layer` "Pagina" button (`lvgl.page.next`). Power card shrunk 300→230 to clear the button. Temps from `temp_outside_entity` (outside) + `temp_inside_entity` (inside) — real IDs only in git-ignored `secrets.yaml`. Compiled (Flash 71.8 %), OTA-flashed, **user-confirmed perfect** — page flip + `lvgl.page.next` wrap + `°` rendering all good. See "Multi-page UI + Temperature page".
- ✅ Energy & Devices pages (2026-06-29): two more pages — `energy_page` (NET power reusing `power_combined` + avg + solar + battery + 2 EV stop-buttons) and `devices_page` (washer/dryer + 3 AC modes). Power shown in W (kW-native readings ×1000); climates via the firmware's first `text_sensor` block; EV buttons `switch.turn_off` + beep. Compiled (Flash 72.4 %), OTA-flashed; boot log confirms all 9 new sensors + 3 text_sensors subscribed, clean boot, no errors. **Visual/tap verification pending user** (4-page cycle wrap, AC chips show `KOEL`, EV stop-tap — test the unavailable ID.4 (`vin_*`) button first as a safe no-op). See "Energy & Devices pages".
- ✅ Light toggle on energy page (2026-07-01): 74×74 blue button in the top-right gap under the Pagina button (right of ZON/BATT), `switch.toggle` on `switch.verlichting_vs4_v1`. Compiled (Flash 72.4 %), OTA-flashed, clean boot (all sensors live, no errors). **Visual/tap verification pending user** (button renders top-right on page 2; tap toggles the light).

## ESPHome Environment
- Venv: `~/esphome-cyd-venv`
- Version: ESPHome 2026.5.3
- Serial port: `/dev/ttyUSB0` (first flash only, then OTA)
- Build/flash: `esphome compile` then `esphome upload … --device <cyd_ip>` (NOT `run` — it hangs on the "show logs?" prompt).
- **PlatformIO `esptool` breakage (2026-06-29):** a compile failed at `bootloader.bin` with `ModuleNotFoundError: No module named 'esptool'` — the penv had a dangling editable `esptool` install pointing at a deleted `~/.platformio/packages/tool-esptoolpy`. Fix: `~/.platformio/penv/bin/python -m pip uninstall -y esptool && ~/.platformio/penv/bin/python -m pip install esptool` (→ real `esptool 5.3.1`). Repeat if it recurs.
