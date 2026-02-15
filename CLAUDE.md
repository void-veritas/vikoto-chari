# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

ZMK firmware for a Charybdis Nano (5-column) split keyboard:
- **Right half**: Vikoto board (nRF52840) with optional Procyon I2C trackpad via VIK connector
- **Left half**: nice!nano v2
- **Optional dongle**: Prospector adapter (XIAO BLE) with display

## Build System

Builds use ZMK's GitHub Actions reusable workflow (`.github/workflows/build.yml`), which reads `build.yaml` and `config/west.yml`. There is no local build command — firmware is built via CI or a local ZMK development environment with `west`.

To build a single target locally (requires a ZMK west workspace):
```
west build -s zmk/app -p -b vikoto -- -DSHIELD=chari_right
west build -s zmk/app -p -b nice_nano -- -DSHIELD=chari_left
west build -s zmk/app -p -b xiao_ble -- -DSHIELD="chari_dongle prospector_adapter"
```

The `build.yaml` matrix defines all targets. Dependencies are declared in `config/west.yml`.

## Architecture

### Two Operating Modes

**Non-dongle mode** — right half is BLE central:
- `chari_right` (vikoto): central, reads trackpad directly via `direct_trackpad_listener`
- `chari_left` (nice_nano): peripheral
- `trackpad_split` node is **disabled** on both halves

**Dongle mode** — Prospector is BLE central, both halves are peripherals:
- `chari_dongle` (xiao_ble + prospector_adapter): central, receives forwarded trackpad data via `trackpad_listener` → `trackpad_split`
- `chari_dongle_right` (vikoto): peripheral, forwards trackpad events over BLE using `zmk,input-split` (`trackpad_split.device = <&trackpad>`)
- `chari_dongle_left` (nice_nano): peripheral

### Key File Relationships

- `boards/shields/chari/chari.dtsi` — shared base: matrix transforms (6-col and 5-col), physical layouts, `trackpad_split` definition, `kscan0` stub
- `boards/shields/chari/chari_{left,right}.overlay` — non-dongle sibling overlays fill in kscan GPIOs and trackpad config
- `boards/shields/chari/chari_dongle{,_left,_right}.overlay` — dongle mode siblings
- `boards/shields/chari/Kconfig.defconfig` — sets which sibling is central (`ZMK_SPLIT_ROLE_CENTRAL`)
- `boards/sadekbaroudi/vikoto/` — Vikoto board definition (pinctrl, VIK connector GPIO nexus, flash layout)
- `config/chari.keymap` — user keymap (selects `five_column_transform`, 8 layers with homerow mods)
- `config/chari.conf` — global Kconfig (BLE TX power, experimental connection)

### Trackpad (Procyon / MaxTouch)

- Driver: `microchip,maxtouch` compatible, Kconfig symbol `CONFIG_INPUT_MICROCHIP_MAXTOUCH`
- I2C address `0x4a` on `&vik_i2c` (i2c1), CHG interrupt on `&vik_conn 3` (GPIO P0.04)
- VIK I2C requires `bias-pull-up` on i2c1 pinctrl groups (set in right-half overlays)
- Input processors configure snipe mode (layer 3, 1/3 scale) and scroll mode (layer 2, XY→scroll with Y-invert)

### Root Kconfig

The root `Kconfig` file defines `int` types for `ZMK_TRACKPAD_LOGICAL_X/Y` and `ZMK_TRACKPAD_PHYSICAL_X/Y`. This works around Zephyr 4.1's `KCONFIG_WARNINGS_AS_ERRORS` — the upstream `peacock_vik_module` defines these symbols without types.

## Dependencies (west.yml)

| Repository | Branch | Purpose |
|---|---|---|
| zmkfirmware/zmk | main | ZMK core (also imports fingerpunch deps → VIK, cirque, pmw3610) |
| george-norton/maxtouch-zephyr-module | main | MaxTouch trackpad driver |
| carrefinho/prospector-zmk-module | core/zephyr-4-1 | Prospector dongle shield |

## Module Registration

`zephyr/module.yml` registers this repo as a Zephyr module with `board_root`, `dts_root`, and `kconfig` paths, allowing Zephyr/ZMK to discover the custom board and DTS bindings under `boards/` and `dts/`.
