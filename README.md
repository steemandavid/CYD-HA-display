# CYD Home Assistant Control Display

ESPHome firmware that turns a **Cheap Yellow Display (CYD / ESP32-2432S028)** into a
wall-mounted control panel for [Home Assistant](https://www.home-assistant.io/). It shows
live power consumption and provides three touch buttons that toggle Home Assistant switches,
plus an audible over-power alarm. A top-right **Pagina** button cycles through the other
pages: **temperatures**, an **energy monitor** (grid, solar, home battery, two EV chargers
with stop-buttons), and **devices** (appliances + airco modes).

![platform: ESPHome](https://img.shields.io/badge/platform-ESPHome%202026.5-blue)
![board: ESP32-2432S028](https://img.shields.io/badge/board-ESP32--2432S028-yellow)

---

## Features

- **Live power readout** — displays `sensor.your_power` in watts, large and centered on a dark **Home Assistant-style** card.
  - **Three-tier colour:** green below 200 W, blue 200–3000 W, red at/above 3000 W.
- **AI usage row** — four cells between the power card and the buttons, colour-coded by provider:
  - **Claude** (clay `#D97757`): `cc 5h` % and `cc reset` countdown.
  - **z.ai** (blue `#1F63EC`): `z.ai 5h` % and `z.ai reset` countdown.
  - `%` cells read the usage sensors directly; reset cells read HA **template sensors** that compute minutes-to-reset (ESPHome can't read timestamp entities as numbers).
- **Three toggle buttons** laid out side by side (tall, easy-to-hit rectangles):
  | Button | Action |
  |--------|--------|
  | Garage OPENEN | toggles `switch.your_garage_open` |
  | Garage SLUITEN | toggles `switch.your_garage_close` |
  | Deurslot WACHT KAMER | toggles `switch.your_door_lock` |
- **Over-power alarm** — short repeating beep while power stays **above 3000 W** for 5 s
  (5 s debounce), via the onboard speaker.
- **Temperature page (multi-page UI)** — tap the top-right **Pagina** button to cycle to the
  **temperature** page: two temperatures side by side, **Buiten** (outside) and **Binnen**
  (inside), each a large readout on a dark card. The button lives on the `top_layer`, so it
  shows on every page and cycles through all of them.
- **Energy monitor page** — grid **NET** power + running average (both in W, side by side),
  two solar power readings (SolarLog + Solarman, W), home-battery charge/discharge, and two
  **EV stop-buttons**: each shows the car's charging power and, on tap, calls `switch.turn_off`
  on the charger and plays a short beep. NET, the average, and the Solarman solar reading are
  kW-native × 1000 → W; the rest are already W. A small **light toggle** button (`VS4-V1`) fills
  the top-right gap under the Pagina button — tap toggles `switch.verlichting_vs4_v1`.
- **Devices page** — washing-machine & dryer live power, plus the current mode of three
  air conditioners (`KOEL` / `WARM` / `UIT` …), coloured by mode.

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

# AI usage row (the reset ones are HA template sensors computing minutes-to-reset)
claude_5h_entity: sensor.your_claude_5h_usage
claude_reset_entity: sensor.your_claude_reset_minutes
zai_5h_entity: sensor.your_zai_5h_quota
zai_reset_entity: sensor.your_zai_reset_minutes

# Temperature page (temp_page)
temp_outside_entity: sensor.your_outside_temperature
temp_inside_entity: sensor.your_inside_temperature

# Energy monitor page (energy_page) — power_combined is reused via power_entity above
power_avg_entity: sensor.your_power_average
solar_power_entity: sensor.your_solar_power_ac
solar_solarman_entity: sensor.your_solarman_power_output
marstek_charge_entity: sensor.your_battery_charge_power
marstek_discharge_entity: sensor.your_battery_discharge_power
enyaq_power_entity: sensor.your_ev1_charging_power
vin_power_entity: sensor.your_ev2_charging_power
enyaq_charge_switch: switch.your_ev1_charging     # stop-button -> switch.turn_off
vin_charge_switch: switch.your_ev2_charging       # stop-button -> switch.turn_off
verlichting_vs4_switch: switch.your_lighting_vs4  # light toggle (top-right gap on energy page)

# Devices page (devices_page)
shelly0_power_entity: sensor.your_washer_power
shelly1_power_entity: sensor.your_dryer_power
climate_woonkamer_entity: climate.your_ac_living
climate_overloop_entity: climate.your_ac_landing
climate_slaapkamer_entity: climate.your_ac_bedroom
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
| `buzzer_threshold` | `3000` | Watts; alarm beeps above this (5 s debounce) |

## Key technical decisions & gotchas

These were hard-won during bring-up — see `CLAUDE.md` for the full story.

- **Display driver: use `model: ILI9342` with the `ili9xxx` platform.** It shares the
  ILI9341 init sequence but defaults to 320×240 landscape, giving correct orientation with
  LVGL `rotation: 0` and **no** display transform. The `mipi_spi` platform produces garbled
  output on this board regardless of configuration. **Colour constants are written BGR** (red/blue byte-swapped) — the ILI9342 preset enables the display's BGR mode, so the panel swaps red/blue; the `col_*` substitutions pre-swap to compensate (`color_order:` has no effect on the LVGL path). Without this, blue shows orange and red shows blue (green looks fine, which hides it).
- **Touch transform: `swap_xy: true, mirror_x: true, mirror_y: true`.** A wrong `mirror_x`
  is invisible while widgets are full-width-stacked (only the Y axis selects a widget) — it
  only shows up with side-by-side widgets as flipped left/right taps.
- **Power sensor reports kW, not W** — every comparison and label multiplies by 1000.
- **Speaker uses `ledc` output (not `esp32_dac`)** — the `rtttl` component requires it.
- **`buffer_size: 25%`** — the CYD has no PSRAM.
- **HA-side permission:** ESPHome *sending* a `homeassistant.action` is not the same as HA
  *executing* it — the device's "Allow the device to perform Home Assistant actions" toggle
  in the ESPHome integration must be ON.
- **AI-row reset cells need template sensors:** ESPHome imports timestamp entities (the
  Claude/z.ai reset times) as ISO strings, not numbers, so the reset countdowns are computed by
  HA template sensors (minutes-to-reset) and read in as plain numbers. The brand colours
  (`#D97757` Claude clay, `#1F63EC` z.ai blue) are written BGR like every other `col_*` (above).
- **Multi-page navigation:** the display has four pages (`main_page`, `energy_page`,
  `devices_page`, `temp_page` — temperatures last) cycled by a "Pagina" button on the `top_layer` (LVGL's
  always-on-top page) via `lvgl.page.next`. Putting the button on `top_layer` is the clean way
  to have one nav control on every page without repeating it per page. The `°` in the
  temperatures relies on the built-in Montserrat font (LVGL's default glyph range includes
  0xB0); if you ever set a custom `text_font`, add the glyph explicitly or it shows as a box.
- **Climate modes need a `text_sensor`:** an AC's state is a mode *string* (`cool`, `off`, …),
  which ESPHome's numeric `homeassistant` `sensor` can't hold (it'd come through as `NaN`).
  Import them with `text_sensor: - platform: homeassistant` instead, and map the string to a
  display code in `on_value`. (Same shape of problem as the AI-row reset timestamps, which are
  computed by HA template sensors instead.) The energy-page power figures are shown in **watts**
  (NET, the average, and the Solarman solar reading are kW-native × 1000; the SolarLog, battery,
  and Shelly readings are already W) — still check each entity's `unit_of_measurement` before
  formatting.

## License

Personal project; no warranty. Use at your own risk.
