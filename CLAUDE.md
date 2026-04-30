# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a ZMK keyboard firmware configuration repository for [Ergohaven](https://github.com/ergohaven) keyboards. There is no local build toolchain — firmware is compiled entirely via GitHub Actions by pushing to the repository, which delegates to the reusable workflow at `ergohaven/ergohaven-zmk/.github/workflows/build-user-config.yml@main`.

**To trigger a build:** push changes to the repository. Firmware artifacts are produced by the CI workflow.

## Supported Keyboards

| Keyboard | Form Factor | Notable Features |
|---|---|---|
| `velvet_v3` | Split wireless | Standard split |
| `velvet_v3_ui` | Split wireless | Integrated trackball on right half |
| `op36` | Split wireless | 36-key columnar stagger |
| `k03` | Split wireless | Rotary encoders |
| `imperial44` | Split wired/wireless | Two rotary encoders per half |
| `trackball_v3.0/v3.1/royal` | Standalone | Trackball-only peripheral |

## Repository Structure

- `build.yaml` — Defines every firmware build target (board, shield, keymap, cmake-args combinations)
- `config/west.yml` — Zephyr west manifest; pins ZMK source to `ergohaven/ergohaven-zmk@main`
- `config/<keyboard>.keymap` — Primary keymap in ZMK DTS syntax
- `config/<keyboard>_ruen.keymap` — Russian+English bilingual keymap variant
- `config/<keyboard>.conf` — Kconfig settings (sleep, battery, combo limits)
- `config/<keyboard>_qube.conf` — Kconfig overrides for the USB dongle variant
- `config/<keyboard>.json` — Physical key layout for ZMK Studio
- `config/keys_ru.h` — Russian locale key definitions (`RU_CYRILLIC_*` macros)

## Build Target Conventions

Each keyboard has multiple shield variants specified in `build.yaml`:

- `<keyboard>_left` / `<keyboard>_right` — individual halves of a split keyboard
- `<keyboard>_qube qube dongle_screen` — USB dongle with display (replaces wireless central)
- `<keyboard>_left_qube` — left half paired with qube dongle
- `snippet: studio-rpc-usb-uart` + `cmake-args: -DCONFIG_ZMK_STUDIO=y` — enables ZMK Studio real-time keymap editing over USB
- `artifact-name` — when set, overrides the default artifact filename (used for non-default variants)

## Keymap Architecture

### Layer Structure (all keyboards follow this pattern)
1. **Base** — QWERTY alpha keys
2. **Nav** — numbers row + navigation (arrows, home/end/pgup/pgdn)
3. **Sym** — symbols
4. **Adj** — Bluetooth profiles (`BT_SEL 0-4`, `BT_CLR`), output switching (`OUT_BLE`/`OUT_USB`), media keys, F-keys, `bootloader`, `studio_unlock`

### Russian/English (`_ruen`) Keymaps
These replace the single base layer with two layers (`en` and `ru`) and add language-switching logic:
- `to_en` / `to_ru` macros send `Ctrl+Shift+1` / `Ctrl+Shift+2` (OS-level input method switch)
- `layer_en` / `layer_ru` macros combine layer switch with OS language switch
- Combo on key-positions `3 4` switches between en/ru base layers
- `en` is a one-parameter macro that temporarily switches to English, sends a keypress, then restores Russian — used throughout `sym_ru` layer for symbols that differ under the Russian OS layout
- `sym_en` and `sym_ru` are separate symbol layers activated from their respective base layers

### Trackball Keymaps
Trackball devices configure `&trackball_listener` input processors:
- Base layer: standard pointer movement with CPI set via `&trackball { cpi = <N>; }`
- Scroll layer: `zip_xy_to_scroll_mapper` with scroll-snap, Y-axis inverted
- Sniper layer: `zip_xy_scaler 1 4` (reduced speed)
- `velvet_v3_ui` adds `zip_auto_mouse` (auto-activates mouse layer on trackball movement after 800ms idle)
- `trackball_royal` inverts both axes via cmake args (`-DCONFIG_PMW3610_INVERT_X=y -DCONFIG_PMW3610_INVERT_Y=y`)

### Home Row Mods (`op36`)
Uses custom `hml`/`hmr` behaviors (balanced hold-tap, 280ms term, 175ms quick-tap, 150ms prior-idle) with `hold-trigger-key-positions` restricting hold activation to the opposite hand's keys.

## Adding a New Keyboard or Variant

1. Add `.keymap` and `.conf` files to `config/`
2. Add the corresponding `_ruen` keymap variant if Russian support is needed
3. Add `.json` layout for ZMK Studio
4. Add build entries to `build.yaml` following the existing pattern

## Key Configuration Options (`.conf` files)

- `CONFIG_ZMK_SLEEP=y` + `CONFIG_ZMK_IDLE_SLEEP_TIMEOUT=600000` — deep sleep after 10 min
- `CONFIG_ZMK_PM_SOFT_OFF=y` — soft power off support
- `CONFIG_ZMK_BATTERY_REPORTING=y` — battery level over BLE
- `CONFIG_ZMK_SPLIT_BLE_CENTRAL_BATTERY_LEVEL_PROXY=y` — central reports peripheral battery
- `CONFIG_ZMK_COMBO_MAX_KEYS_PER_COMBO=3` / `CONFIG_ZMK_COMBO_MAX_COMBOS_PER_KEY=7` — raised combo limits
- `CONFIG_ZMK_SLEEP=n` in `_qube.conf` — qube dongle is USB-powered, no sleep needed
