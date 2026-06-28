# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a ZMK keyboard firmware configuration repository for [Ergohaven](https://github.com/ergohaven) keyboards. Config files for two keyboards are present: **Op36** and **Velvet V3**. Each has an English-only keymap and a bilingual English/Russian (RUEN) variant.

The build system and board definitions live in the upstream [ergohaven-zmk](https://github.com/ergohaven/ergohaven-zmk) repository, imported via `config/west.yml`. This repo only contains user config files — no local west/cmake build setup is needed.

## Building

Firmware is built exclusively via **GitHub Actions**. Push to any branch to trigger a build. The workflow file is `.github/workflows/build.yml` and delegates to the upstream reusable workflow `ergohaven/ergohaven-zmk/.github/workflows/build-user-config.yml@main`.

All build targets are declared in `build.yaml` at the root. Each entry specifies a `board` (always `ergohaven`), a `shield` (keyboard half + optional accessories), optional `keymap` override (for RUEN variants), `snippet: studio-rpc-usb-uart`, and `cmake-args: -DCONFIG_ZMK_STUDIO=y`.

`build.yaml` currently builds Op36 targets only (left half, right half, Qube dongle — each with English and RUEN variants) plus a `settings_reset` utility firmware. The Velvet V3 config files exist but have no corresponding `build.yaml` entries.

To add a new build target, add an entry to `build.yaml`. The `artifact-name` field is required only when the default naming would conflict (e.g., RUEN variants, Qube dongle).

## Repository Structure

All user config files live in `config/`:

```
config/
├── {keyboard}.keymap          # English-only keymap
├── {keyboard}_ruen.keymap     # Bilingual English/Russian keymap
├── {keyboard}.json            # Physical key layout for ZMK Studio
├── {keyboard}.conf            # ZMK Kconfig options
├── {keyboard}_qube.conf       # Extra options for Qube dongle screen variant
├── keys_ru.h                  # Russian Cyrillic key macros (shared header)
└── west.yml                   # Upstream dependency manifest
```

## Keymap File Structure (ZMK DSL)

Keymap files are devicetree overlays (`.keymap`). A typical file declares in order: `behaviors {}`, `combos {}`, `macros {}`, then `keymap {}` with named `layer` nodes.

### English-only variant (e.g. `op36.keymap`)

4-layer structure:

| # | Name | Purpose |
|---|------|---------|
| 0 | Base | QWERTY alpha, home row mods, thumb layer-taps |
| 1 | Nav | Numbers row, navigation (arrows, home/end/pgup/pgdn), media |
| 2 | Sym | Programming symbols, brackets, operators |
| 3 | Adjust | BT pairing/clearing, F-keys, bootloader, ZMK Studio unlock |

Nav → Adjust is accessed via `mo 3` nested inside the Nav layer.

### RUEN variant — `op36_ruen.keymap`

9-layer structure. Layers 0 and 1 are both "base" layers, switched via `to 0`/`to 1` macros (not `mo`), allowing the keyboard to stay on the active language layer between key presses.

| # | Name | Purpose |
|---|------|---------|
| 0 | En | English QWERTY base; right thumb is `mo 2` (sym) |
| 1 | Ru | Russian Cyrillic base (same physical layout, different keycodes); right thumb is the `sym_act` macro |
| 2 | Sym | Single unified symbol layer (plain English symbol keycodes) |
| 3 | Nav | Numbers row, navigation, modifiers |
| 4 | Adjust | BT pairing/clearing, F-keys, bootloader, ZMK Studio unlock |
| 5 | Game | Gaming layer (WASD, no home row mods) |
| 6 | Game Num | Number row for gaming |
| 7 | Game Right | Right-hand alpha for gaming |
| 8 | ru_add | Holds extra Cyrillic chars (Х, Ё, Ъ); currently a placeholder — not bound to any access key |

There is **one** `sym` layer shared by both bases. From the English base it is reached with a plain `mo 2`. From the Russian base the right thumb runs the `sym_act` macro, which switches the OS to English, holds `mo 2` for the press duration, then switches the OS back to Russian on release — so the unified layer can use ordinary English symbol keycodes regardless of the active base. This replaced an older design (still used by Velvet V3 below) that had separate `sym_en` / `sym_ru` layers.

### RUEN variant — `velvet_v3_ruen.keymap`

Uses the **older dual-symbol design** and a different layer order (wider 12-column grid):

| # | Name | Notes |
|---|------|-------|
| 0 | En | English base |
| 1 | Ru | Russian base |
| 2 | Nav | navigation/numbers |
| 3 | Sym En | symbols from English base |
| 4 | Sym Ru | symbols from Russian base — wraps unavailable keys in the `en` macro |
| 5 | Adjust | BT / F-keys / bootloader / Studio unlock |
| 6–8 | Game / Game Num / Game Right | gaming cluster |

### Common Custom Behaviors

- `hml` / `hmr` — home row mod hold-taps (balanced flavor, 280ms tapping term, 150ms require-prior-idle, 175ms quick-tap); `hml` triggers only on right-side key releases, `hmr` only on left-side
- `spc_lt` — space bar hold-tap: hold = `mo <layer>`, tap = `kp SPACE`
- `ru_space` / `en_space` (Velvet V3 only) — mod-morph: tap = `SPACE`, LGUI+tap = switch language layer

### Language-Switching Macros

- `to_ru` — emits `LS(LC(N2))` (OS input source shortcut for Russian)
- `to_en` — emits `LS(LC(N1))` (OS input source shortcut for English)
- `layer_ru` — `to 1` then `to_ru` (switch layer and notify OS)
- `layer_en` — `to 0` then `to_en`
- `sym_act` (op36_ruen only) — zero-param macro on the Russian base's right thumb: `to_en`, hold `mo 2` (sym) for the press, then `to_ru` on release. Lets the unified `sym` layer use English symbol keycodes from the Russian base.
- `en` (velvet_v3_ruen only) — one-param macro: switches OS to English, holds/taps the passed key, then switches back to Russian; used per-key in the `sym_ru` layer. In `op36_ruen` this macro is still defined but no longer referenced (superseded by `sym_act`).

### Russian Variant Conventions

RUEN keymaps include `keys_ru.h` which provides `RU_*` macros that map Russian characters to the HID keycodes the OS expects when set to a Russian input layout (e.g. `RU_CYRILLIC_A` maps to the `F` HID key).

Combos `cmben` (key-positions 2+3 on layer 1) and `cmbru` (key-positions 2+3 on layer 0) invoke `layer_en` / `layer_ru` for language switching. Additional combos for characters not on the physical layout (Х, Ъ) are placed on adjacent key pairs within the Russian base layer.

The `cmblock` combo (3-key, layers 0+1) sends `LG(LC(Q))` to lock the macOS screen. The `cmbgame` combo (3-key) toggles the Game layer cluster on/off.

## Key Files to Understand

- `config/op36_ruen.keymap` — most feature-complete keymap; canonical example of home row mods, language macros, combos, and game layers
- `config/velvet_v3_ruen.keymap` — Velvet V3 variant; shows `ru_space`/`en_space` mod-morphs and a wider key grid (12-column vs 10-column for Op36)
- `config/keys_ru.h` — all `RU_*` keycode definitions; consult when adding or modifying Russian-mode symbol keys
- `build.yaml` — authoritative list of all shipped firmware variants
