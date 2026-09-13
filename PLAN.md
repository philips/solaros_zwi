# Port Plan: SolarOS → ZeroWriter Ink

**Goal:** Run SolarOS as the operating environment on the ZeroWriter Ink
(Inkplate 5 V2-based e-ink writing device), as a first-class SolarOS board
target built through the normal board-manifest pipeline.

This plan is derived from three local sources:

| Repo | Role | License | How we may use it |
|---|---|---|---|
| `solar_os/` | The OS being ported (Apache-2.0, ESP-IDF via PlatformIO) | Apache-2.0 | Port target — all changes land here |
| `ize-compose/` | 3rd-party OSS firmware for ZW Ink; **technical reference only** | PolyForm Noncommercial 1.0.0 | Read for hardware facts. **Do not copy code** into SolarOS |
| `zerowriter_ink/` | Official (closed) firmware binaries, keymaps, fonts, CAD + KiCad design files | GPL-3.0 (keyboard fw) / mixed | Hardware reference, protocol facts, validation baseline |

Hardware facts below are extracted from `ize-compose/lib/InkplateLibrary/`
(vendored, LGPL-3.0) and `zerowriter_ink/` design files. We **re-implement,
not copy**: the Inkplate library is LGPL-3.0 and ize-compose is
noncommercial-licensed, so all SolarOS code must be written fresh (register
maps and protocol constants are facts; code is not).

---

## 1. Target hardware dossier — ZeroWriter Ink

The device is an **Inkplate 5 V2 carrier**: classic ESP32 (WROVER, **not**
S3) + 1280×720 parallel-driven e-ink panel + separate keyboard MCU.

### 1.1 Compute
- MCU: classic ESP32-WROVER, 240 MHz, QIO flash @ 80 MHz
  (`ize-compose/platformio.ini`: `board = esp32dev`, `-DBOARD_HAS_PSRAM`,
  `-mfix-esp32-psram-cache-issue`)
- PSRAM required for the 1280×720 framebuffer mirror, fonts, docs.
- Flash size: **verify on hardware** (stock fw uses `min_spiffs.csv`, i.e.
  4 MB layout; run `esptool flash_id` in Phase 0).
- Closest existing SolarOS base profile: `esp32_devkitc_v4_wrover`
  (classic ESP32 + PSRAM) and its
  `sdkconfig.defaults.freenove_esp32_wrover_v3` (SPIRAM quad 40 MHz,
  4 MB mapped of 8 MB — matches classic-ESP32 cache limits).

### 1.2 Display — the core porting problem
1280×720 e-ink panel driven **in parallel via I2S + GPIO bit-banging**
(no SPI EPD controller like SolarOS's existing `ssd1683`/`st7305` drivers):

- Pixel data: **I2S1** in 8-bit mode, DMA line buffer (160 bytes/line @ 1bpp)
  — `InkplateLibrary/src/system/UtilI2S/UtilI2S.h`
- I2S pin mux (from `Inkplate5V2Driver.cpp: pinsAsOutputs()`):
  - CL (pixel clock, I2S1_BCK): **GPIO0**
  - D0–D7: **GPIO4, 5, 18, 19, 23, 25, 26, 27**
- Bit-banged control lines (from `boards/Inkplate5V2/pins.h`):
  - LE: **GPIO2**, CKV: **GPIO32**, SPH: **GPIO33**
- Panel control + power via **PCAL6416A I2C expander @ 0x20**:
  OE=P0.0, GMOD=P0.1, SPV=P0.2, WAKEUP=P0.3, PWRUP=P0.4, VCOM=P0.5,
  **SD_PMOS=P1.2** (SD card rail switch)
- **TPS65186 PMIC** on I2C: e-paper rails (+15/−15/+20/−20 V), power-good,
  panel temperature read, VCOM EEPROM
- Waveform tables (1bpp, multi-mode): `Inkplate5V2/waveforms.h`
  (`E_INK_WIDTH 1280`, `E_INK_HEIGHT 720`)
- Partial updates supported with configurable full-refresh threshold;
  mirror framebuffer diff against previous frame (stock keeps `_partial`
  buffer in PSRAM)
- Power sequencing: WAKEUP/PWRUP handshake with PMIC, read power-good,
  VCOM control; `einkOn()/einkOff()`, tri-state all EPD pins when off
  (`pinsZstate()`), burn-in clean cycles

### 1.3 Input
- **Keyboard = separate ESP32-WROOM(-DA)** on the keyboard PCB
  (`zerowriter_ink/src/keyboard/zwi_kb_feb2026/zwi_kb_feb2026.ino`):
  - 5×14 matrix scan with light-sleep between polls
  - Sends **raw bytes over UART, 921600 baud 8N1** (`Serial.begin(921600, SERIAL_8N1, -1, 1)` → TX=GPIO1)
  - Byte protocol: key index **0–60** (61-key map), modifiers as down/up
    markers **240/241**=Shift, **242/243**=Ctrl, **244/245**=Alt,
    **246/247**=Meta
  - Keymap: `zerowriter_ink/utils/keymaps/keymap.json`
    (normal + shift + caps layers; arrows, tab, enter, backspace at fixed indices)
- **The main ESP32 receives keys on UART0 RX (GPIO3)** — the stock
  firmware does `Serial.begin(921600)` and reads keys from `Serial`
  (`IZEcompose.ino:4078+`). Keys share the UART0/console line.
- Wake button: **GPIO36** (RTC-capable, `WAKE_BUTTON_PIN`,
  `IZEcompose.ino:66`) — used for power/wake UX.

### 1.4 Storage, power, misc
- SD card: SPI (**HSPI**: SCK=14, MISO=12, MOSI=13, CS=15, 25 MHz —
  `Inkplate5V2Driver.cpp:886`), rail switched by expander pin P1.2
  (SD_PMOS, enabled LOW) — *not* a direct GPIO, which matters because
  SolarOS's `SOLAR_OS_BOARD_PIN_SD_POWER` is a plain GPIO hook.
- Battery: `readBattery()` via ADC divider (verify exact GPIO — GPIO35 is
  typical for Inkplate; confirm against KiCad nets in
  `zerowriter_ink/design/src/Zerowriter Inkplate 5 Gen2/v1.2.0`).
- Panel temperature for waveform selection: TPS65186 temp register.
- Official baseline for validation: `zerowriter_ink/firmware_releases/`
  (`zw_latest.merged.bin`, v1.30).

---

## 2. Architecture mapping

SolarOS services never include concrete driver headers; boards are declared
by a TOML manifest that CMake compiles into board config
(`doc/manual/boards.md`). Port surface:

| ZW Ink hardware | SolarOS layer | What we add |
|---|---|---|
| 1280×720 parallel EPD | board display ops (`src/board/solar_os_board_display.h`) + display service | **New driver**: `epd-parallel` (I2S1 + GPIO + expander + PMIC) + service like `solar_os_ssd1683.c` |
| PCAL6416A expander @0x20 | i2c bus (`i2c_esp_idf`) + fixed device | New thin `pcal6416a` helper driver (datasheet-written) |
| TPS65186 PMIC | same I2C bus | New thin `tps65186` helper driver (rails, power-good, temp, VCOM) |
| Keyboard MCU (UART bytes) | input service (`src/services/solar_os_input.h` key events) | New serial-keyboard decoder feeding the input queue |
| Wake button GPIO36 | `[defines] SOLAR_OS_BOARD_PIN_KEY` / buttons table | Manifest entry |
| SD (HSPI, switched rail) | `storage_expansion` (SDSPI) | Bus + device in manifest; rail quirk (§4.D) |
| Battery ADC | `battery_adc` driver + manifest | Divider pin + scaling defines |
| WROVER-class ESP32 | `esp32_devkitc_v4_wrover` base profile | New manifest extending the classic-ESP32 base |

Flavor: default **`writerdeck`** (already exists: editor, reader, notes,
SD storage, Wi-Fi — matches this device exactly).

---

## 3. Licensing rules for this port

1. **ize-compose** is PolyForm Noncommercial → treat as *documentation only*.
2. **InkplateLibrary** (LGPL-3.0, vendored in ize-compose) → do not translate
   its code 1:1. Write the SolarOS driver from: ESP32 TRM (I2S1, GPIO),
   NXP PCAL6416A datasheet, TI TPS65186 datasheet, panel/waveform *data*
   (numeric waveform tables + pin map are facts). If any vendored waveform
   data is deemed expressive, isolate it in a clearly-marked data file and
   note its provenance in `LICENSE`/headers.
3. **Keyboard firmware** (GPL-3.0) runs on a separate MCU and stays a
   separate binary — no merge concern; its byte protocol is a fact.
4. SolarOS upstream policy: a native board port is acceptable (board
   support explicitly welcomes PRs) but requires **hardware-in-the-loop
   test evidence**, not just a clean build (`solar_os/README.md`).

---

## 4. Phased work breakdown

### Phase 0 — Hardware fact verification (target: 1 session on real hardware)
- [ ] `esptool flash_id` → confirm flash size (4 vs 8/16 MB) and pick
      partition CSV (`partitions_4mb.csv` vs `partitions_8mb_single.csv` /
      `partitions_16mb_single.csv`). Decide: single-app (like stock, no OTA)
      vs OTA slots (SolarOS supports OTA — defer to follow-up).
- [ ] Confirm PSRAM size (heap test) and that the classic-ESP32 4 MB
      mapped-window config suffices.
- [ ] Extract netlist facts from KiCad projects:
      `zerowriter_ink/design/src/Zerowriter Inkplate 5 Gen2/v1.2.0` and
      `.../Zerowriter Keyboard/v1.2.0` — confirm I2C SDA/SCL pins (expander +
      PMIC), battery ADC pin/divider, keyboard-MCU RX wire on the main board
      (UART0 RX GPIO3 vs a free GPIO), wake button GPIO36, LED if any.
- [ ] Verify battery measurement approach (GPIO + divider ratio) by reading
      the schematic; capture reference voltage used by stock `readBattery()`.

**Exit:** a short hardware-notes file committed next to the plan (or PR
description) with confirmed pin table (appendix below updated).

### Phase 1 — Board skeleton: boots SolarOS console on UART0 (no display)
New files in `solar_os/`:
- [ ] `platformio.ini`: add
      `[env:zerowriter_ink]` (`board = esp32dev`-class,
      `board_build.partitions = <from Phase 0>`,
      `custom_solaros_default_flavor = writerdeck`,
      `board_build.cmake_extra_args = -DSOLAR_OS_BOARD=zerowriter_ink
      -DSDKCONFIG_DEFAULTS=sdkconfig.defaults.zerowriter_ink`)
- [ ] `boards/zerowriter_ink.json` — PlatformIO board definition
      (classic esp32, PSRAM flag, flash size from Phase 0; model on
      `boards/esp32...` entries / espressif `esp32dev`)
- [ ] `boards/manifests/zerowriter_ink.toml` — manifest extending the
      classic-ESP32 base: `mcu = "esp32"`, drivers
      `["uart_esp_idf","i2c_esp_idf","spi_esp_idf","gpio_esp_idf",
      "adc_esp_idf","pwm_esp_idf","storage_expansion"]`, capabilities
      (`psram`,`wifi`,`ble`,`sd`,`i2c`,`spi`,`key`,`battery_adc`,…),
      `[defines]` for UART pins, `[buses]` i2c0 + spi (SD HSPI 12/13/14/15),
      `[pins]` policy table, wake-key button entry
- [ ] `sdkconfig.defaults.zerowriter_ink` — derive from
      `sdkconfig.defaults.freenove_esp32_wrover_v3` (keep
      `CONFIG_SPIRAM=y`, quad 40 MHz, cache workaround, size optimization);
      adjust flash size/partition name per Phase 0
- [ ] Run `python3 scripts/validate_board_metadata.py` until clean

**Exit:** `pio run -e zerowriter_ink` succeeds; flashing yields a SolarOS
shell on UART0 (921600 during dev); `status` output sane; note: at this
stage the serial console and keyboard share UART0 — acceptable for bring-up,
resolved in Phase 3.

### Phase 2 — Display driver (largest task, ~60% of the work)
New files:
- [ ] `src/drivers/pcal6416a.c/.h` — I2C expander: pin mode/write/read
      (datasheet-written; ~200 lines)
- [ ] `src/drivers/tps65186.c/.h` — PMIC: power up/down rails, power-good
      read, temperature read, VCOM EEPROM get/set (optional)
- [ ] `src/drivers/epd_parallel.c/.h` — panel driver (new display class):
      - I2S1 8-bit TX with DMA line buffer (160 B/line), CL on GPIO0,
        D0–D7 mux per pin table
      - CKV/SPH/LE bit-bang + expander OE/GMOD/SPV control
      - power sequencing: WAKEUP→PWRUP→rail power-good→VCOM; off path
        tri-states all pins (incl. `pinsZstate`-equivalent)
      - waveform engine: port waveform *tables*; temperature-compensated
        mode selection via TPS65186 temp
      - `present_mono_xbm()` + full-frame present; partial update =
        PSRAM diff vs previous frame + row-scan of changed region;
        full-refresh threshold (configurable, `controller_mode`
        `"partial"/"full"` like ssd1683 service exposes)
      - burn-in clean helper for boot/shutdown
- [ ] `boards/drivers/display_epd_parallel.cmake` —
      `set(SOLAR_OS_BOARD_DISPLAY_DRIVER "epd_parallel")`
- [ ] `src/services/solar_os_epd_parallel.c` — service mirroring
      `solar_os_ssd1683.c` structure: builds `solar_os_board_display_t`
      (1280×720, mono surface formats), registers as `display0`,
      `source=board`, `role=primary`
- [ ] `src/board/solar_os_board_display_epd_parallel.c` — glue instantiated
      from manifest `[[devices]] driver = "epd_parallel"` bindings
      `{ i2c = "i2c0", pmic_addr, expander_addr, pins... }`
- [ ] Manifest: add `[[buses]] i2c0` (SDA/SCL from Phase 0) and
      `[[devices]]` for `epd-parallel`; capability `"display","gfx"`;
      `SOLAR_OS_BOARD_DISPLAY_*` defines (1280×720, orientation 0)

Performance notes: 1bpp frame = 115,200 bytes → keep composed frame in
PSRAM, line DMA buffer in internal RAM; budget the row-scan timing from the
stock clock divider (I2S `clockDivider = 5`) and validate with a scope
against stock behavior if typing latency feels off.

**Exit:** SolarOS boots to an on-panel welcome/console; typing via UART
keyboard (Phase 3 can land in either order) refreshes at usable latency;
full refresh happens every N partials; `display status` (or equivalent
shell command) reports epd-parallel correctly.

### Phase 3 — Keyboard input
- [ ] New serial keyboard decoder (e.g.
      `src/services/solar_os_serial_kb.c` or a board input driver under
      `src/drivers/` + manifest device): 921600 8N1, byte protocol
      §1.3 → `solar_os_input_key_event_t` queue (key + modifier state),
      auto-repeat stays in keyboard MCU (it already repeats) — decoder only
      translates down/up markers
- [ ] Keymap: static table matching
      `zerowriter_ink/utils/keymaps/keymap.json` layers (normal/shift/caps);
      arrows → SolarOS UP/DOWN/LEFT/RIGHT keys; expose layout as data file
- [ ] **UART0 sharing decision (needs Phase 0 evidence):**
      - Stock wiring delivers keys on UART0 RX (GPIO3). Options:
        **(a)** SolarOS board glue installs a UART0 RX tap that forwards
        protocol bytes (0–60, 240–247) to the input service and passes other
        bytes to the console — matches stock hardware, zero rework;
        **(b)** if the ZW breakout routes keyboard TX elsewhere, use a
        dedicated UART. Do **(a)** unless Phase 0 says otherwise.
- [ ] Wake button GPIO36 → board button → `SOLAR_OS_KEY_*` (power menu)

**Exit:** typing in the shell works with modifiers, arrows, CapsLock;
modifier LEDs/state consistent; no keystrokes lost while Wi-Fi active.

### Phase 4 — Storage, battery, panel telemetry
- [ ] SD via `storage_expansion` SDSPI device on HSPI (CS 15);
      **rail quirk:** SD_PMOS is expander pin P1.2, not a GPIO. Approach:
      driver turns the rail on at init and before remounts (small hook in
      board storage glue), or keep rail always-on for v1 and note the
      ~µA-level cost. Prefer explicit hook; do NOT abuse
      `SOLAR_OS_BOARD_PIN_SD_POWER`.
- [ ] Battery: `battery_adc` manifest entry (pin + divider from Phase 0);
      `battery status` correctness vs stock readout
- [ ] Panel temperature available to display driver; VCOM read-only display
      in status (writing VCOM = polish item, keep defaults from EEPROM)

**Exit:** `disk status` mounts/reads SD (stock docs live under `/ize_compose/`
— SolarOS will use its own namespace); battery % matches stock firmware ±5%.

### Phase 5 — Power management & sleep UX
- [ ] Deep sleep entry: panel off sequence (rails down, pins tri-state),
      SD rail off, then `esp_deep_sleep` with **ext1 wake on GPIO36**
      (RTC-capable) — verify GPIO36 RTC capability for ext1 in Phase 0
- [ ] Wake path: boot → driver init → draw boot image/clean cycle (stock
      draws `/ize_compose/initial.png`; SolarOS equivalent: boot splash from
      its own assets)
- [ ] Idle timeouts consistent with SolarOS power policy; typing any key
      aborts sleep countdown

**Exit:** device sleeps with blank/protected panel, wakes on button with
session restored; measured sleep current within sane range of stock.

### Phase 6 — Flavor tuning & writing UX
- [ ] `writerdeck` flavor as default; verify editor/reader/notes on 1280×720
      (font metrics at 720p — select larger built-in fonts from `solar_os/fonts`)
- [ ] Orientation default (panel native is landscape per stock rotation 0)
- [ ] Status bar: battery, refresh counter; document keyboard shortcuts in
      on-device `help` tree (manual sync requirement — `doc/manual/`)
- [ ] Optional (out-of-tree): `.bbf` font converter in `scripts/` for user
      fonts from `zerowriter_ink/compiled fonts/` (separate tool, no license
      contamination in firmware)

### Phase 7 — Validation, docs, upstream
- [ ] `python3 scripts/validate_board_metadata.py` clean
- [ ] Regression builds: `pio run -e zerowriter_ink` plus
      `odroid_go` (classic-ESP32 shared paths: ILI9341, SD-SPI, DAC, ADC
      D-pad) and `solar_term` per `doc/manual/boards.md` checklist
- [ ] HITL evidence on real hardware: boot log, typing latency video/notes,
      battery + SD + sleep measurements → required by upstream CONTRIBUTING
- [ ] Docs: target table + pin rules in `doc/manual/boards.md`,
      `doc/manual/expansion.reference.md` GPIO/bus tables (validator checks
      these), CHANGELOG entry
- [ ] PR to `solar_os` upstream with maintenance commitment (ZW Ink is not
      the maintainer's primary target — we own ongoing validation)

---

## 5. Risks & open questions

| # | Risk / question | Mitigation |
|---|---|---|
| R1 | Parallel-EPD driver is a new display *class* in SolarOS (no I2S-EPD precedent) | Model service/board glue 1:1 on ssd1683 stack; isolate panel timing inside `epd_parallel.c` |
| R2 | Typing latency (partial-update row scan) may exceed stock if waveform timing differs | Port stock clock divider + waveform tables exactly; scope-check CL/CKV vs stock |
| R3 | UART0 console ↔ keyboard byte collision (stock shares one line) | RX tap filter (Phase 3a); console input restricted to keyboard device; verify no shell escape sequence collides with bytes 240–247 |
| R4 | LGPL waveform tables provenance | Numeric tables treated as data; header notes origin; keep in dedicated `waveform` data file |
| R5 | Flash size / partition choice unknown until Phase 0 | Both 4 MB and 8/16 MB single-app CSVs already exist upstream |
| R6 | SD rail behind I2C expander vs GPIO power hook | Board-glue hook; fallback always-on |
| R7 | 8 MB PSRAM only partially cacheable on classic ESP32 | Same as existing WROVER targets; keep framebuffer mirror in mapped 4 MB |
| Q1 | Battery ADC pin + divider ratio | Phase 0 (schematics) |
| Q2 | Keyboard TX routing on main board (GPIO3 vs dedicated) | Phase 0 (schematics) |
| Q3 | GPIO36 ext1 wake support on this board revision | Phase 0 (datasheet + bench test) |
| Q4 | Upstream acceptance: maintainer discussion before large native driver | Open issue with plan + evidence early |

---

## 6. Confirmed pin map (to be verified in Phase 0)

| Function | ESP32 pin / bus |
|---|---|
| EPD CL (I2S1 BCK) | GPIO0 |
| EPD D0–D7 (I2S1 data) | GPIO4, 5, 18, 19, 23, 25, 26, 27 |
| EPD LE | GPIO2 |
| EPD CKV / SPH | GPIO32 / GPIO33 |
| EPD OE / GMOD / SPV / WAKEUP / PWRUP / VCOM | PCAL6416A @ 0x20: P0.0–P0.5 |
| SD rail (SD_PMOS) | PCAL6416A @ 0x20: P1.2 (active low) |
| SD SPI (HSPI) | SCK 14, MISO 12, MOSI 13, CS 15 (25 MHz) |
| Keyboard MCU → keys | UART 921600 8N1, TX→main RX (GPIO3 per stock fw) |
| Wake button | GPIO36 |
| I2C (expander + PMIC) | *(SDA/SCL: confirm — Arduino Wire defaults 21/22)* |
| Battery ADC | *(confirm — likely GPIO35 divider)* |
| PMIC TPS65186 | I2C 0x48 |

## 7. Immediate next actions
1. Phase 0 bench session: `esptool flash_id`, PSRAM heap probe, KiCad
   netlist extraction (I2C pins, battery, keyboard RX, GPIO36).
2. Create Phase 1 files (manifest skeleton → validator green → console boots).
3. Open upstream SolarOS issue referencing this plan (per CONTRIBUTING
   "discuss large … new boards … before investing").
