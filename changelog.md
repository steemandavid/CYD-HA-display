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

---

# 2026-06-13 (session 6): Dark Theme, 3-Tier Power Colours & the ILI9342 Red/Blue Swap

## Summary

UI overhaul of the CYD panel: retuned the alarm/gain, redesigned the power readout into a
dark **Home Assistant-style** card with a **3-tier green/blue/red** value, recoloured the
buttons to HA blue, and chased down a stubborn **red/blue colour-channel swap** that turned
out to be unrecoverable from config (the real fix is BGR-encoded colour constants). Also a
full documentation review/sync. All work over OTA at <CYD_IP>.

## 1. Tuning

| Setting | Before | After |
|---------|--------|-------|
| `buzzer_threshold` (over-power alarm) | 4000 W | **3000 W** |
| rtttl `gain` (speaker volume) | 60% | **10%** |

## 2. Power value — vibrant colours → 3-tier

Original label colours were "very bland". Moved to a 3-tier scheme driven by two
substitutions — red intentionally tracks `buzzer_threshold`, so the number turns red exactly
when the alarm is about to arm:

| Power | Colour | Hex (true RGB) |
|-------|--------|----------------|
| < 200 W (`color_threshold`) | vibrant green | `0x39FF14` |
| 200–3000 W | HA blue | `0x03A9F4` |
| ≥ 3000 W (`buzzer_threshold`) | vibrant red | `0xFF1744` |

Implemented as an `if / else if / else` lambda (was a 2-way ternary).

## 3. Buttons — Home Assistant blue + label fix

- Button theme recoloured to HA blue: `bg_color 0x03A9F4` → grad `0x0288D1`, white text.
- Door-lock label `"Deurslot WACHTKAMER"` → `"Deurslot WACHT KAMER"` (added space so the long
  Dutch text wraps cleanly in the 90 px-wide label). Updated the matching `logger.log` string
  and the README button table.

## 4. Dark theme

White (LVGL default) screen background replaced with a dark HA-style surface:

- Screen background: `0x101418` (deep blue-black) — set on `main_page`
- Card panel behind the power readout: `0x1A2028`, `bg_opa: COVER`, `radius: 8`
- "Verbruik" subheader text: grey `0xAAAAAA` → soft blue-grey `0xB0BEC5`

## 5. The red/blue colour-swap saga (key gotcha)

After the dark-theme flash, colours were wrong: **blue rendered orange, red rendered blue,
green looked fine.** Diagnosis:

- It's a **clean R/B byte-swap**: `0x03A9F4` (blue) → `0xF4A903` (orange); `0xFF1744` (red) →
  `0x4417FF` (blue). Green is nearly swap-symmetric, which is why nobody noticed earlier
  (low power is always green).
- Root cause: the `ili9xxx` `model: ILI9342` preset sets **`MADCTL=0x48` with the BGR bit**,
  so the panel swaps red/blue of every pixel it receives.

**First fix attempt — `color_order: BGR` on the display — FAILED.** It left the output
identical. Proven by elimination: for `color_order` to be effective, *some* assumption about
the default would have to make "default" and "BGR" differ — but both produced the same
swapped output. So `color_order:` is a **no-op on the LVGL direct-buffer path** (LVGL owns
the screen via `update_interval: never`, bypassing the ili9xxx colour-order handling).

**Real fix — write all colours BGR (R/B byte-swapped).** An R/B swap is its own inverse, so
pre-swapping each constant makes the panel's swap cancel out. Centralised as `col_*`
substitutions with the true colour commented, so they stay maintainable:

```yaml
col_bg: "0x181410"        # -> true 0x101418  dark screen
col_card: "0x28201A"      # -> true 0x1A2028  power card panel
col_label: "0xC5BEB0"     # -> true 0xB0BEC5  "Verbruik" subheader
col_low: "0x14FF39"       # -> true 0x39FF14  green  (power < 200 W)
col_mid: "0xF4A903"       # -> true 0x03A9F4  HA blue (200-3000 W, buttons)
col_high: "0x4417FF"      # -> true 0xFF1744  red    (power >= 3000 W)
col_btn_bot: "0xD18802"   # -> true 0x0288D1  button gradient bottom
```

Confirmed visually: 171 W / −30 W green, 2314 W blue, 3774 W red, buttons HA blue.

## 6. OTA workflow — clearing stuck processes

The first OTA attempt hung. Cause: **three `esphome run` processes left over from Jun 12**
were still alive and holding the ESPHome build lock — the new compile was blocked (2 s CPU,
not compiling), not slow. Device itself was fine (ping 0% loss, OTA port 3232 open).

- `pkill -f "bin/esphome run"` to clear them; confirmed no stale build mutex (the only
  `.lock` is `dependencies.lock`, a manifest, not a process lock).
- Switched from `esphome run` (which waits on an interactive "show logs?" prompt — the likely
  reason the Jun 12 processes hung) to **`esphome compile` then `esphome upload --device`**,
  output redirected to a live log file (`< /dev/null` to satisfy any prompt). Reliable.

## 7. Documentation review

Full review of `CLAUDE.md`, `README.md`, `CYD-HA-Control-SPEC.md`, `changelog.md`:

- Fixed stale values from this session (4000→3000 W, 60%→10% gain) across CLAUDE.md/README.
- Added a "Historical note" banner to the SPEC pointing to CLAUDE.md/changelog as the source
  of truth (the SPEC is the original, pre-build spec and intentionally diverges).
- Recorded the BGR red/blue-swap gotcha in CLAUDE.md (Display Driver) and README (gotchas) —
  corrected twice, since the first version wrongly claimed `color_order: BGR` was the fix.
- Verified the public-repo secret convention: `git grep` over tracked files shows no real
  IPs/SSIDs/entities leaked.

## 8. Files Modified

| File | Status |
|------|--------|
| `cyd-ha-control.yaml` | Threshold 3000 W; gain 10%; dark theme (screen + card); 3-tier colour lambda; HA-blue buttons; `Deurslot WACHT KAMER`; **`col_*` BGR colour substitutions** (the R/B-swap fix) |
| `CLAUDE.md` | Status → 2026-06-13; gain 10%; BGR red/blue-swap gotcha; 3-tier thresholds; fixed `your_garage_open/close` typo |
| `README.md` | Alarm 3000 W; 3-tier colour + dark card features; BGR gotcha; `Deurslot WACHT KAMER` |
| `CYD-HA-Control-SPEC.md` | Added "Historical note" divergence banner |
| `changelog.md` | This session-6 entry |

---

## 9. Blog post update & deploy (steeman.be)

Updated the existing steeman.be post (created in session 5) to reflect this session's
changes, then rebuilt and deployed.

Post edits (`website-steeman.be/content/posts/turning-a-cheap-yellow-display-into-a-home-assistant-wall-panel.md`):

- Alarm threshold 4000→3000 W; speaker gain 60%→10%.
- Added a **"Colours — the fourth gotcha (and the sneakiest)"** section telling the ILI9342
  red/blue-swap story for a general reader (blue→orange, red→blue, green fine; `color_order`
  a no-op; the BGR fix).
- UI description rewritten: dark theme + 3-tier green/blue/red value + HA-blue buttons.
- Line count "260" → "270-odd".

Deploy: `hugo --gc --minify` (142 pages), then curl FTP upload of **22 changed files** to
ftp.steeman.be — the post page plus homepage, RSS, sitemap, posts index, and the 4 category
pages (diy/electronics/esp32/home-assistant) with their pagination. 0 failures. Verified live:
HTTP 200, "3000 W" present, "fourth gotcha" present, "4000 W" gone. (Confirmed the post
appears only on the homepage, so no `page/2+` upload was needed.)

Git: the post was **untracked** in the website repo (session 5 deployed it but never committed
it). Committed just that one file — explicit `git add` of the single `.md`, never `git add .`,
because the website repo has 1101 modified files (the `old-site/` binary churn) — and pushed
(`5e2fbb6` → `website-steeman.be` main).

---

# 2026-06-19 (session 7): AI Usage Row (cc + z.ai quota cells)

## Summary

Added a 4-cell AI-usage row between the power card and the buttons, mirroring the new HA "AI
usage" dashboard: each provider (Claude, z.ai) gets a 5h-% cell and a reset-countdown cell,
colour-coded with the real brand colours. To make room the power card dropped its "Verbruik"
label and shrank; the buttons shifted down. The two reset cells read HA **template sensors**
because ESPHome can't import timestamp entities as numbers. All work over OTA at <CYD_IP>.

## 1. Layout rework (320×240)

Removed the "Verbruik" subheader to free a row, then redistributed vertically:

| Region | Before | After |
|---|---|---|
| Power card | y6 h96 (label + value) | y4 h64 (value only) |
| AI row | — | y70 h58 (NEW) |
| Buttons | y110 h116 | y130 h106 |

AI row = 4 display-only `obj` cells, 74×58, at x 6/84/162/240 (6px side margins, 4px gaps).

## 2. The four cells

| Cell | Colour | Source |
|---|---|---|
| cc 5h      | Claude clay #D97757 | `sensor.your_claude_5h_usage` (%) |
| cc reset   | Claude clay #D97757 | NEW HA template: minutes to session reset |
| z.ai 5h    | z.ai blue #1F63EC   | `sensor.your_zai_5h_quota` (%) |
| z.ai reset | z.ai blue #1F63EC   | NEW HA template: minutes to 5h reset |

Provider name is in the caption (`cc` / `z.ai`) as well as the colour.

## 3. Brand colours (sourced from real assets)

- Claude **#D97757** — confirmed from Anthropic's official Claude Code skills repo.
- z.ai **#1F63EC** — the dominant blue in the official `logo.svg`
  (`https://z-cdn.chatglm.cn/z-ai/static/logo.svg`); the logo has no purple despite the
  "blue-purple gradient" reputation, which comes from marketing UI.
- Both written BGR per the existing convention: `col_claude` 0x5777D9, `col_zai` 0xEC631F.

## 4. Reset countdowns — HA template sensors (the timestamp gotcha)

ESPHome's `homeassistant` sensor imports a timestamp entity as an **ISO string** (verified — not
a usable epoch), so the reset cells can't compute "time until reset" on-device from the raw
reset-time entities. Chosen approach: HA template sensors that output minutes-to-reset, which
ESPHome reads as a plain number and formats as `HuMM` (e.g. `1u23`).

- `sensor.your_claude_reset_minutes` ← the Claude session-reset-time entity
- `sensor.your_zai_reset_minutes` ← the z.ai 5h-reset-time entity
- Jinja: `{% set r = states('<src>')|as_datetime %}{% if r %}{{ ((r-now()).total_seconds()/60)|round(0)|int }}{% else %}unknown{% endif %}`

Defined in a YAML package (`packages/ai_usage_countdowns.yaml`), written to the HA config SMB
mount + mirrored in the `hass-ai-usage-monitoring` project. Packages load only at startup, so this
needs an HA restart (disarm the alarm first — alarm + Telegram + Frigate live on this box, per the
AI-usage-monitoring caveat). That restart conveniently also drops the CYD's API connection so
ESPHome re-subscribes and the reset cells populate immediately (no separate device reboot).
Verified post-restart: `claude_session_minutes_to_reset`=14, `z_ai_5h_minutes_to_reset`=94.
(The `ha_mcp_tools` services exist over MCP but their returns didn't serialize through the
wrapper; writing the package file over the SMB mount was the clean path. UI Helpers would also
have worked without a restart, but the YAML package is reproducible + version-mirrored.)

## 5. Firmware (cyd-ha-control.yaml)

- `col_claude`/`col_zai` BGR substitutions.
- 4 homeassistant sensors (`claude_5h`, `claude_reset`, `zai_5h`, `zai_reset`), each `on_value`
  → `lvgl.label.update`; `%` cells `%.0f%%`, reset cells `HuMM`, `isnan(x)`→`--`, negatives clamped to 0.
- Power card: label removed, h 64, value `CENTER` (**gotcha:** `CENTER_MID` is not a valid LVGL
  align token — caught at compile; the token is just `CENTER`).
- AI cells: value `montserrat_24` at `TOP_MID y:0`, caption `montserrat_14` at `BOTTOM_MID y:0`,
  `pad_all: 0` (see §8). Buttons shifted to y 130 / h 106.

## 6. Secrets

4 new `!secret` entity keys (`claude_5h_entity`, `claude_reset_entity`, `zai_5h_entity`,
`zai_reset_entity`) in `secrets.yaml` + sanitized `secrets.yaml.example`. Site-specific entity
IDs (incl. the Claude `david_pro` one) live only in the git-ignored `secrets.yaml`.

## 7. Files Modified

| File | Status |
|---|---|
| `cyd-ha-control.yaml` | `col_claude`/`col_zai`; 4 sensors + AI row; power card 64/no-label; buttons shifted |
| `secrets.yaml` / `secrets.yaml.example` | +4 AI entity keys |
| `CLAUDE.md` | AI usage row subsection; `col_claude`/`col_zai`; status 2026-06-19; secrets list |
| `README.md` | Features (AI row); secrets block; gotcha (template/timestamp) |
| `changelog.md` | This session-7 entry |
| HA `packages/ai_usage_countdowns.yaml` (+ `hass-ai-usage-monitoring` mirror) | 2 template sensors: minutes-to-reset for Claude + z.ai (loads at HA restart) |

## 8. Layout tuning after first flash (two iterations)

The first flash (cells 74×50, value `montserrat_28`) showed the value and caption **overlapping**
on every card, plus a faint **grey horizontal strip** along the bottom of each card. Two rounds:

1. Grew cells 50→58 px (taking 8 px from the buttons: 114→106), reduced the value font 28→24
   (~10% as requested), pinned the number to the cell top. Still overlapped.
2. Root cause: **LVGL `obj` default padding** — it both shrinks the area children align into (so
   `TOP_MID`/`BOTTOM_MID` collided) and draws an auto-scrollbar band (the "grey strip") when content
   nears the padded inset. **Fix: `pad_all: 0`** on each card — documented to remove the `obj`
   scrollbar band. With padding gone the value anchors to the true top and the caption to the true
   bottom with a clear gap. (ESPHome LVGL widgets docs; HA community: "Setting `pad_all: 0` removes
   the scrollbar.")

Final cell layout: `74×58, pad_all: 0`, value `montserrat_24` `TOP_MID y:6`, caption
`montserrat_14` `BOTTOM_MID y:-6` — a 3rd tuning round nudged both 6 px toward center
(value ↓6, caption ↑6) to kill the edge-hugging + excess gap left by `pad_all: 0`;
~6 px top/bottom margins, ~8 px gap between value and caption. Confirmed good by user.

## 9. Status
- ✅ Firmware compiled + OTA-flashed (final layout with `pad_all: 0` addressing the reported overlap + scrollbar band).
- ✅ HA template sensors live (Claude 14 / z.ai 94 min verified post-restart); CYD re-subscribed on the restart, all 4 cells populated.
- ✅ Claude values cross-checked against claude.ai → Settings → Usage (agree). z.ai has no public usage dashboard to cross-check; values come from the z.ai API package (`packages/zai_usage.yaml`).
- ✅ Layout confirmed good by user after 3 tuning rounds (overlap → scrollbar band → edge-hugging/excess gap). Final: value `TOP_MID y:6`, caption `BOTTOM_MID y:-6`. Committed as a follow-up after the initial `/finished`.

---

# 2026-06-19 (session 8): Blog post — AI usage row + photos

## Summary
Updated the steeman.be CYD blog post to document session 7's AI-usage row, added photos, and
deployed. **No firmware changes** — `cyd-ha-control.yaml` untouched.

## 1. Post content (website-steeman.be)
- New **"An AI usage row"** section: the cc/z.ai 5h-% + reset-countdown cells, brand colours
  (Claude clay `#D97757`, z.ai blue `#1F63EC`), with two new gotchas continuing the post's
  through-line — **"Timestamps — the fifth gotcha"** (ESPHome imports HA timestamp entities as
  ISO strings → countdowns computed by HA template sensors) and **"LVGL object padding"**
  (`pad_all: 0` kills the value/caption collision + the grey scrollbar band). Cross-link to the
  AI-usage-monitoring post.
- Wove the AI row into the "The firmware" UI description; bumped post date 2026-06-12 →
  2026-06-19; "270-odd" → "450-odd" lines; added a lessons bullet (watch how ESPHome imports an
  entity's type) and a Resources cross-link.

## 2. Photos (`static/images/CYD-HA-Display/`)
| File | Placement |
|---|---|
| `cyd-ai-usage-row.jpg` (today — shows the 4 cells) | AI section + hero `image:` |
| `cyd-power-and-buttons.jpg` | "The firmware" UI shot |
| `cyd-in-case.jpg` | "Mounting" / 3D-printed case |
| `cyd-board.png` (bare ESP32-2432S028 render) | "What is a CYD?" intro |
- Each full + 600 px `thumb`; board thumb saved as **JPEG (39 KB)** — the PNG thumb was 298 KB
  (RGBA flattened on white, visually identical).
- Held back `IMG_20260613_191109` (softest, near-duplicate of the other June-13 shots).

## 3. Deploy (steeman.be)
- `hugo --gc --minify` (143 pages) + `scripts/deploy.sh --no-build` (lftp mirror via `~/.netrc`).
- Two deploy rounds (second added the board render). Verified live: post + all images HTTP 200.
- **Gotcha:** steeman.be is CDN-cached (Anubis; `x-cache-status: HIT`). Right after a deploy the
  bare URL serves the *previous* version for minutes — verify with a `?v=` cache-bust or
  `Cache-Control: no-cache` header (→ `MISS`, true `last-modified`). Saved to memory
  `steeman-be-deploy-cache-verification`. Also: Hugo does not clean `public/`, so a renamed
  thumbnail left an orphan `cyd-board thumb.png` that `lftp -n` re-uploaded; removed it locally +
  via FTP `DELE` (now 404).

## 4. Files
| File | Status |
|---|---|
| `website-steeman.be/content/posts/turning-…wall-panel.md` | Updated — committed `3c02bb3` → website repo `main` |
| `website-steeman.be/static/images/CYD-HA-Display/*` | 8 image files added — committed `3c02bb3` |
| CYD `changelog.md` | This session-8 entry |

---

# 2026-06-23 (session 9): Doorbell Chime — CYD Plays on Doorbell Press

## Summary

The CYD's speaker now chimes when any of the 4 front-door doorbell buttons
(`binary_sensor.your_doorbell_1..4` on ESP-209) is pressed. The CYD amp is
RTTTL-only (no MP3), so it answers with a generated ~3 s **3× ding-dong** rather
than the doorbell MP3 that ESP-221 plays. The chime is driven from the existing
HA doorbell automation and is **purely additive** — ESP-221 MP3, Telegram ping,
and Dutch TTS all stay as-is.

## 1. Design decision (user-chosen)

Two options were offered; user picked **HA-automation-driven + additive**:
- **HA automation drives the CYD** (not CYD self-subscribing): the CYD exposes a
  "Doorbell Chime" button; HA presses it.
- **Keep the ESP-221 chime** (not replace): CYD is an additional chime.

## 2. Firmware (cyd-ha-control.yaml)

Added a template `button` that plays the tune on press:

```yaml
button:
  - platform: template
    name: "Doorbell Chime"
    icon: "mdi:bell-ring"
    on_press:
      - logger.log: "Doorbell chime (from HA)"
      - rtttl.play: "Doorbell:d=4,o=5,b=120: c6,g5,16p,c6,g5,16p,c6,g5"
```

Tune: 3× ding-dong — C6 (1047 Hz) "ding" → G5 (784 Hz) "dong" (perfect fourth),
repeated three times over **~3.25 s** (@120 BPM: 2×(0.5+0.5+0.125) + (0.5+0.5)).
Three repeats so it's insistent enough to notice, vs. a single easy-to-miss chime.
No new secrets — the CYD doesn't subscribe to anything; HA tells it when.

Compiled + OTA-flashed with ESPHome 2026.5.3 (`~/esphome-cyd-venv`): 65 s compile,
8 s OTA to `<CYD_IP>`. Flash 67.8 %. `esphome compile` then `upload --device`
(not `run` — hangs on the logs prompt).

## 3. HA automation (id `1686663509022` 'TTS: deurbel ingedrukt')

Added `button.press` as the **first action** so the chime fires immediately on
press (before the ESP-221 play_media sequence + delays):

```yaml
actions:
  - target:
      entity_id: button.your_cyd_doorbell_chime
    data: {}
    action: button.press
  - data: ...            # telegram, esp_221 play_media x4, TTS — unchanged
```

> **Entity-id gotcha:** the CYD device is registered in HA as
> `<room>_cyd_control_display` (HA registers it with a location/room prefix), so the
> button entity is `button.your_cyd_doorbell_chime` — **not**
> the `button.cyd_control_...` the firmware name alone would suggest. Confirmed
> by `list_entities`, not assumed.

Backed up `automations.yaml` (`.bak-…-cyd-chime`) before editing; `automation.reload`
after.

## 4. Verification

- `button.press` on the chime entity returns ok; **user heard the ding-dong** —
  HA → CYD → rtttl path confirmed end-to-end.
- `automation.your_doorbell_announcement` reloaded, state `on`.

## 5. Caveat — ESP-209 (doorbell source) offline

`device_tracker.your_doorbell_node` = `unavailable` since 2026-06-22, so the 4 doorbell
entities are not currently in HA's state machine. The automation (and therefore
the chime) will fire on a real press the moment ESP-209 is back online. The
button-press test above proves the CYD half independently.

## 6. Files Modified

| File | Status |
|---|---|
| `cyd-ha-control.yaml` | + template `button` "Doorbell Chime" → rtttl 3× ding-dong |
| HA `automations.yaml` | + `button.press` first action in 'TTS: deurbel ingedrukt' (id 1686663509022); backed up + reloaded |
| `secrets.yaml` | (unchanged — no new secrets) |
| `CLAUDE.md` | + Doorbell-chime section + status |
| `changelog.md` | This session-9 entry |
