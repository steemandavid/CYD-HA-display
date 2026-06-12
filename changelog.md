# 2026-06-12: CYD HA Control Display — First Flash & Display Fix

## Summary

Set up a Cheap Yellow Display (ESP32-2432S028) as an ESPHome-based Home Assistant
control panel. First flash over USB, then iterative OTA debugging to fix the display
driver (10+ compile/flash cycles). Display now works correctly. Button HA actions
and speaker/beep are still unresolved.

## System Info

| | Details |
|---|---------|
| **CYD Board** | ESP32-2432S028, ILI9341 display, XPT2046 touch |
| **CYD IP** | <CYD_IP> (Wi-Fi, OTA) |
| **Serial Port** | /dev/ttyUSB0 |
| **HA Instance** | Proxmox VM at <HA_IP> |
| **ESPHome** | 2026.5.3, venv at ~/esphome-cyd-venv |
| **Project dir** | ~/claudecode/projects/CYD-HA-display |

## 1. Environment Setup

- Created Python venv at `~/esphome-cyd-venv`, installed ESPHome 2026.5.3
- User `john` already in `dialout` group for serial access
- Created `secrets.yaml` with Wi-Fi creds and API key matched to HA's
  `/config/esphome/secrets.yaml` (fetched via SSH to HA VM)

## 2. First Flash (USB)

- Compiled and flashed `cyd-ha-control.yaml` over USB serial at `/dev/ttyUSB0`
- Device booted, joined Wi-Fi `<your-wifi-ssid>`, got IP <CYD_IP>
- Display showed "Verbruik --W" and three button labels — correct layout

## 3. YAML Fixes Applied (chronological)

| # | Issue | Fix |
|---|-------|-----|
| 1 | `rtttl` can't use `esp32_dac` output | Changed GPIO26 output from `esp32_dac` to `ledc` |
| 2 | LVGL blocks `rotation` in display config | Moved `rotation: 90` from ili9xxx to `lvgl:` block |
| 3 | Touch not responding | Restored `transform: swap_xy: true, mirror_y: true` on XPT2046 (hardware PCB wiring fix) |
| 4 | API key mismatch with HA | Replaced generated key with HA's existing key from `/config/esphome/secrets.yaml` |
| 5 | Power sensor reports in kW, not W | Added `* 1000.0f` in display label, threshold comparison, and color lambda |
| 6 | Display dimension mismatch (1/5th noise) | Tried `mipi_spi` with ESP32-2432S028 board preset → garbled |
| 7 | `mipi_spi` garbled output | Tried `mipi_spi` with ILI9341 chip model + transform → garbled |
| 8 | `mipi_spi` garbled (all variants) | Reverted to `ili9xxx` with `transform: swap_xy/mirror_x/mirror_y` → garbled |
| 9 | **Final fix** | Changed to `model: ILI9342` (landscape variant of ILI9341, 320×240 default) with no rotation or transform → **correct** |

### Key Discovery: ILI9342 Model

The `ILI9342` model in the `ili9xxx` driver uses the **same init sequence** as `ILI9341`
but defaults to 320×240 dimensions. Combined with the init sequence's MADCTL=0x48
(MX+BGR), this gives correct landscape output with no additional rotation needed.
The `mipi_spi` driver produces garbled output on this board regardless of configuration
— likely due to SPI protocol differences in its init sequence.

## 4. Current State of cyd-ha-control.yaml

```yaml
display:
  - platform: ili9xxx
    model: ILI9342              # key fix — landscape variant
    spi_id: spi_display
    cs_pin: GPIO15
    dc_pin: GPIO2
    invert_colors: false
    auto_clear_enabled: false
    update_interval: never

lvgl:
  rotation: 0                   # no LVGL rotation needed
  buffer_size: 25%              # no PSRAM
```

## 5. Unresolved Issues

### Buttons — HA actions not firing
- `on_click` trigger produces visual feedback (button grays out)
- But `homeassistant.action` calls produce **zero log output** (even at VERBOSE level)
- `homeassistant.action` IS the correct syntax for ESPHome 2026.x (confirmed in source)
- `on_click` IS a valid LVGL trigger (`CLICK` → `on_click` in `LV_EVENT_MAP`)
- **Next step**: Add `logger.log` to confirm automation fires; try `on_short_click` or
  `on_press`+`on_release` as alternatives; check if button widget needs `clickable: true`

### Speaker/beep — not tested
- GPIO26 LEDC output + rtttl configured but never triggered

### Touch calibration — not verified
- Using default XPT2046 calibration values from spec

## 6. Files Created/Modified

| File | Status |
|------|--------|
| `cyd-ha-control.yaml` | Modified (multiple fixes) |
| `secrets.yaml` | Created |
| `CLAUDE.md` | Created (project context) |
| Memory files in `.claude/projects/` | Created (display findings, project context) |

---

# 2026-06-12 (session 2): Buttons, Speaker & Toggle Fix

## Summary

Resolved the three remaining open items from session 1. Confirmed **buttons fire HA
actions** (the earlier "not working" was a false alarm + a missing HA-side permission),
confirmed the **speaker works**, verified **touch calibration is healthy**, and finally
changed the buttons from a momentary pulse to **`switch.toggle`** to match the existing HA
dashboard behavior. All work done over OTA at <CYD_IP>. Firmware is now feature-complete.

## 1. Buttons — root cause & fix

The session-1 symptom was "`on_click` grays the button but `homeassistant.action` produces no
log output." Diagnosis approach: instrumented button 1 (Garage OPENEN) with `logger.log` on
**all four** interaction triggers in a single flash, then tapped while capturing logs.

- First capture was empty — a **timing mismatch** (device still rebooting after OTA), not a fault.
- Second capture (firm taps) showed the **full chain firing**:
  `on_press → on_release → on_short_click → on_click → "action sent"`.
- So `on_click` *does* fire and `homeassistant.action` *does* run to completion.

Two real causes behind the original symptom:
1. The logger-instrumented build had **never actually been flashed + tapped** before — the
   session-1 conclusion was premature.
2. HA's per-device **"Allow the device to perform Home Assistant actions"** toggle was OFF,
   which silently drops action calls. User enabled it (HA → Settings → Devices → ESPHome →
   CYD) → buttons then triggered HA end-to-end.

Standardized all three buttons on **`on_short_click`** (fires on a deliberate tap, ignores
long-press) with a concise `logger.log` line each.

> **Gotcha:** ESPHome *sending* `homeassistant.action` ≠ HA *executing* it. The device's
> "Allow … to perform Home Assistant actions" integration toggle must be ON.

> **Note:** The xpt2046 `Touchscreen Update [x, y]` debug values are **raw ADC** (≈280–3860
> range), not screen pixels. They landed in-range and LVGL routed taps correctly →
> calibration is healthy.

## 2. Speaker — verified working

Power reads negative, so the `beep_loop` alarm (>1000 W for 5 s) never fires on its own to
test. Added a **temporary boot chime** to confirm the hardware:

```yaml
esphome:
  on_boot:
    priority: -100
    then:
      - delay: 3s
      - rtttl.play: "test:d=4,o=5,b=120:8c,8e,8g,8c6,8g,8c6"
```

Result: **audible chime heard**; logs confirmed GPIO26 LEDC channel 1 ramping to 1047 Hz,
rtttl gain 0.6. The over-threshold alarm uses the same `rtttl.play` path, so the mechanism is
proven. Removed the boot chime and reflashed clean.

## 3. Buttons changed momentary → toggle

User reported the actions behaved wrong vs. their HA dashboard, which uses **toggle**. The
buttons were doing a momentary pulse (`turn_on` → `delay 500ms` → `turn_off`). Changed all
three to a single `switch.toggle` call; removed the now-unused `pulse_ms` substitution and
updated the "momentary buttons" header comment. User confirmed: **fixed.**

```yaml
on_short_click:
  - logger.log: "Button: Garage OPENEN -> toggle switch.your_garage_open"
  - homeassistant.action:
      action: switch.toggle
      data:
        entity_id: switch.your_garage_open
```

Entities toggled: `switch.your_garage_open`, `switch.your_garage_close`,
`switch.your_door_lock`.

## 4. Final status

| Feature | Status |
|---------|--------|
| Display, power readout, Wi-Fi/API, OTA | ✅ (from session 1) |
| Buttons → HA actions (toggle) | ✅ confirmed end-to-end |
| Touch calibration | ✅ healthy |
| Speaker / beep | ✅ audible |
| Over-threshold alarm (>1000 W, 5 s) | ⚠️ mechanism proven, not field-triggered (power reads negative) |

## 5. Files Modified (session 2)

| File | Status |
|------|--------|
| `cyd-ha-control.yaml` | Buttons → `on_short_click` + `switch.toggle`; removed `pulse_ms`; temp boot-chime added then removed |
| `CLAUDE.md` | Status section updated (buttons/speaker/touch resolved, toggle note, HA-toggle caveat) |
| Memory `cyd-ha-project-context.md` | Updated to reflect resolved items + HA-action-permission gotcha |

---

# 2026-06-12 (session 3): Thresholds, Button Layout & Touch Mirror Fix

## Summary

UI/UX refinements. Split the single power threshold into separate label-color and buzzer
thresholds, redesigned the buttons from stacked full-width bars into a side-by-side row, and
fixed a left/right touch-axis mirror that the new layout exposed. All over OTA.

## 1. Power label color + buzzer thresholds (now independent)

Split the old single `power_threshold` substitution into two:

| Substitution | Value | Effect |
|--------------|-------|--------|
| `color_threshold` | `200` (W) | Label **green** below 200 W, **red** at/above |
| `buzzer_threshold` | `4000` (W) | Alarm beeps only above 4000 W (still 5 s debounce) |

Color lambda now `(x*1000 < color_threshold) ? green(0x34C759) : red(0xFF3B30)`; the
`power_high` binary_sensor compares against `buzzer_threshold`.

## 2. Buttons: stacked bars → side-by-side row

Old layout was three full-width (300×38) bars stacked vertically — wide, slim, hard to hit.
Replaced with three tall rectangles in a row across the lower screen:

- Each `100 × 116`, `align: TOP_LEFT`, at `x: 6 / 110 / 214`, `y: 110` (6px margins, 4px gaps)
- Labels given `width: 90` + `text_align: CENTER` so long Dutch text wraps cleanly

## 3. Touch left/right mirror fix

After the side-by-side change, the **outer two buttons triggered each other's actions**
(Garage OPENEN ↔ Deurslot WACHTKAMER); the middle one was fine. Root cause: the touch X axis
was mirrored, but that was **invisible in the old vertical layout** because only the Y
coordinate selected a full-width button. Fix: `mirror_x: false → true` in the xpt2046
`transform` (now `swap_xy: true, mirror_x: true, mirror_y: true`). Confirmed correct.

> **Lesson:** Validate touch X *and* Y mapping with widgets that actually span both axes — a
> wrong horizontal mirror hides completely behind a full-width vertical layout.

## 4. Files Modified (session 3)

| File | Status |
|------|--------|
| `cyd-ha-control.yaml` | Split thresholds (`color_threshold` 200 / `buzzer_threshold` 4000); green/red label lambda; buttons → side-by-side tall layout; `mirror_x: true` |
| `CLAUDE.md` | (carried) status reflects working state |
| Memory `cyd-display-driver-findings.md` | Added correct XPT2046 transform + the layout-hides-mirror lesson |

---

# 2026-06-12 (session 4): README, Secrets Centralization & Public GitHub Repo

## Summary

Packaged the project for a **public** GitHub repo. Wrote a full `README.md`, centralized
**all** site-specific values into git-ignored `secrets.yaml` (referenced from the firmware via
ESPHome `!secret`), sanitized every committed doc, scrubbed git history to a single clean
commit, and published publicly.

Repo: **https://github.com/steemandavid/CYD-HA-display** (public)

## 1. README + initial repo

Wrote `README.md` (features, hardware/pinout, getting-started, config, gotchas). Initialized
git, set identity `David Steeman <david@steeman.be>`, created a **private** repo and pushed.
`.gitignore` already excluded `secrets.yaml` + `.esphome/`.

## 2. Secrets centralization (user's idea)

Goal: keep sensitive data in one place. Realized via ESPHome `!secret`:

- Moved HA entity IDs + IPs into `secrets.yaml` (now the single source of truth: Wi-Fi, API
  key, `power_entity`, `garage_open_switch`, `garage_close_switch`, `doorlock_switch`,
  `ha_ip`, `cyd_ip`).
- Firmware references them with `!secret` (e.g. `entity_id: !secret garage_open_switch`);
  removed the `power_entity` substitution; genericized log strings + header comment.
- `secrets.yaml.example` mirrors it with placeholders.
- Committed `cyd-ha-control.yaml` now contains **zero** site-specific data. Config still
  validates.

> **Caveat (explained to user):** Markdown can't reference a config file (GitHub renders it
> static), so docs can't pull from `secrets.yaml` — they were sanitized in place instead.
> Nothing lost: real values live in local `secrets.yaml`.

## 3. Doc sanitization

`sed` over `README.md`, `CLAUDE.md`, `changelog.md`, `CYD-HA-Control-SPEC.md`, replacing only
real values (not generic HA services like `switch.toggle`):

| Category | Placeholder |
|----------|-------------|
| CYD / HA LAN IPs | `<CYD_IP>` / `<HA_IP>` |
| Wi-Fi SSID | `<your-wifi-ssid>` |
| Garage open/close switch entities | `switch.your_garage_open` / `..._close` |
| Door-lock switch entity | `switch.your_door_lock` |
| Power sensor entity | `sensor.your_power` |

Verified with `git grep` over the committed tree: no real IPs, SSID, entity names, Wi-Fi
password, or API key remain.

## 4. History scrub + go public

The first commit contained unsanitized values. Squashed to a single clean root commit
(`git commit --amend`) and `git push --force`. Token lacked `delete_repo` scope, so could not
delete+recreate; flipped visibility to public via `gh api -X PATCH … -F private=false`.

> **Residual caveat (user accepted):** the old commit (`6557236`) survives as a *dangling*
> object on GitHub until GC — unreachable via any ref, but fetchable by exact SHA until then.
> For a guaranteed clean slate: `gh auth refresh -s delete_repo` then delete + recreate.

## 5. Files Modified (session 4)

| File | Status |
|------|--------|
| `README.md` | Created, then revised for the secrets-centric approach |
| `secrets.yaml` | Extended to hold entity IDs + IPs (git-ignored) |
| `secrets.yaml.example` | Created/extended sanitized template |
| `cyd-ha-control.yaml` | Entities via `!secret`; generic log strings; removed `power_entity` sub |
| `CLAUDE.md` / `changelog.md` / `CYD-HA-Control-SPEC.md` | Sanitized in place; CLAUDE.md gained a public-repo/secrets convention section |
| `.gitignore` | (unchanged — already excluded `secrets.yaml`) |

---

# 2026-06-12 (session 5): Blog Post & Website Deployment

## Summary

Wrote and published a blog post about the CYD HA Control Display project on
[steeman.be](https://www.steeman.be/). Deployed to production via FTP.

## 1. Blog post

Created `/home/john/claudecode/projects/website-steeman.be/content/posts/turning-a-cheap-yellow-display-into-a-home-assistant-wall-panel.md`:

- **Title:** "Turning a Cheap Yellow Display into a Home Assistant Control Panel"
- **Categories:** Electronics, DIY, ESP32, Home Assistant
- **Image directory created:** `static/images/CYD-HA-Display/` (no photos yet — TODO placeholders in post)

### Sections written

1. **Opening** — the real motivation: kids at the front door, wanting a desk panel for power consumption and door/garage control
2. **What is a CYD?** — board overview for readers unfamiliar with it
3. **What I wanted it to do** — three requirements (power display, buttons, alarm)
4. **Why ESPHome?** — framework rationale
5. **The build** — four gotchas (ILI9342 display model, touch mirror_x, ledc vs esp32_dac for speaker, kW vs W units)
6. **The firmware** — YAML architecture with beep alarm code snippet
7. **Building this with Claude Code** — the entire firmware was AI-generated, no hand-written code; iterative debugging workflow
8. **What you need to build one** — bill of materials table (~€8 CYD, USB cable, optional 3D printer), software requirements, link to GitHub repo
9. **Mounting it on the wall (or desk)** — [MakerWorld wall mount](https://makerworld.com/en/models/2655345-wall-mount-for-cyd-cheap-yellow-display-xtouch) by kenzoteke
10. **HA permissions** — the hidden "Allow device to perform actions" toggle
11. **First flash and OTA** — USB then wireless
12. **Lessons learned**
13. **Resources** — links to GitHub repo, ESPHome docs, LVGL, HA, CYD pinout, MakerWorld case, TiltBox post

### Edits after initial draft

- Added wall mount section (MakerWorld model by kenzoteke)
- Rewrote opening to tell the real story (kids at front door, desk control panel)
- Changed "waiting room lock" → "front door lock" throughout
- Added Claude Code section — no hand-written code
- Added "What you need to build one" section with BOM table
- Added GitHub repo links throughout

## 2. Deployment

Built Hugo site (`hugo --gc --minify`, 142 pages) and uploaded 30 files to `ftp.steeman.be` via curl:

- New post page, main index, RSS/sitemap, posts index, 4 category page sets (Electronics, DIY, ESP32, Home Assistant), all pagination pages

**Live at:** https://www.steeman.be/posts/turning-a-cheap-yellow-display-into-a-home-assistant-wall-panel/

## 3. Files Modified

| File | Status |
|------|--------|
| `website-steeman.be/content/posts/turning-a-cheap-yellow-display-into-a-home-assistant-wall-panel.md` | Created (blog post) |
| `website-steeman.be/static/images/CYD-HA-Display/` | Created (empty — photos TODO) |
