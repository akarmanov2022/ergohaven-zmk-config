# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a ZMK keyboard firmware configuration repository for [Ergohaven](https://github.com/ergohaven) keyboards. It contains keymaps, behaviors, and build configurations for 5 keyboards: **Velvet V3**, **K:03**, **Op36**, **Imperial44**, and **Trackball** (v3.0/v3.1). Each keyboard supports English and Russian (RUEN) keymap variants.

The build system and board definitions live in the upstream [ergohaven-zmk](https://github.com/ergohaven/ergohaven-zmk) repository, imported via `config/west.yml`. This repo only contains user config files — no local west/cmake build setup is needed.

## Building

Firmware is built exclusively via **GitHub Actions**. Push to any branch to trigger a build. The workflow file is `.github/workflows/build.yml` and delegates to the upstream reusable workflow `ergohaven/ergohaven-zmk/.github/workflows/build-user-config.yml@main`.

All build targets are declared in `build.yaml` at the root. Each entry specifies a `board` (always `ergohaven`), a `shield` (keyboard half + optional accessories), optional `keymap` override (for RUEN variants), `snippet: studio-rpc-usb-uart`, and `cmake-args: -DCONFIG_ZMK_STUDIO=y`.

To add a new build target, add an entry to `build.yaml`. The `artifact-name` field is required only when the default naming would conflict (e.g., RUEN variants, trackball with custom keyboard name).

## Repository Structure

All user config files live in `config/`:

```
config/
├── {keyboard}.keymap          # English keymap
├── {keyboard}_ruen.keymap     # Russian/bilingual keymap
├── {keyboard}.json            # Physical key layout for ZMK Studio
├── {keyboard}_ruen.json       # Symlink to base JSON (shared layout)
├── {keyboard}.conf            # ZMK Kconfig options
├── {keyboard}_qube.conf       # Extra options for Qube dongle screen variant
├── keys_ru.h                  # Russian Cyrillic key macros (shared header)
└── west.yml                   # Upstream dependency manifest
```

## Keymap File Structure (ZMK DSL)

Keymap files are devicetree overlays (`.keymap`). A typical file declares in order: `behaviors {}`, `combos {}`, `macros {}`, then `keymap {}` with named `layer` nodes.

### Standard Layer Layout

All keyboards use a consistent 4-layer base structure:

| # | Name | Purpose |
|---|------|---------|
| 0 | Base | QWERTY alpha, common punctuation, thumb layer-taps |
| 1 | Nav | Numbers row, navigation (arrows, home/end/pgup/pgdn), media |
| 2 | Symbols | Programming symbols, brackets, operators |
| 3 | Adjust | Bluetooth pairing/clearing, bootloader, ZMK Studio unlock |

Extended variants add more layers: Velvet V3 UI adds Mouse (4), Scroll (5), Sniper (6), and User0-3 (7–10) placeholder layers. Trackball uses Scroll (1), Sniper (2), Adjust (3).

### Common Custom Behaviors

- `hm` / `hml` / `hmr` — home row mod hold-taps (balanced flavor, ~200ms tapping term)
- `spc_lt` — space bar hold-tap: hold = `mo <layer>`, tap = `kp SPACE`
- Language-switching macros: `to_ru`, `to_en`, `layer_ru`, `layer_en` — emit OS input source switch keycode then activate the corresponding base layer
- Trackball-specific: `cap_sen`, `mouse_layer`, `zip_auto_mouse`, input processors for CPI/scaling/sniper mode

### Russian Variant Conventions

RUEN keymaps include `keys_ru.h` and define combos for Cyrillic characters not directly on the layout (e.g., `kha` for Х, `hrdsgn` for Ъ). Language-switch combos (`cmben`/`cmbru`) are placed on key-positions 3+4, active only on the relevant base layer (0 or 1).

The `en` macro pattern — switch to English OS layout, emit a key, switch back to Russian — is used to allow English symbol keys to work while in Russian OS input mode.

## Key Files to Understand

- `config/velvet_v3_ruen.keymap` — most actively developed keymap; contains the canonical example of home row mods, language macros, and combos
- `config/velvet_v3_ui.keymap` — shows trackball/mouse layer behaviors and input processors
- `config/trackball_v3.1.keymap` — dedicated trackball device with tap-dance Bluetooth management
- `build.yaml` — authoritative list of all shipped firmware variants
