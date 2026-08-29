# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

ZMK **user config** repo for [Ergohaven](https://github.com/ergohaven) keyboards. It contains only keymaps, layouts, and Kconfig fragments — no build system, no sources, no tests.

Boards, shields, and the reusable build workflow live upstream in [ergohaven/ergohaven-zmk](https://github.com/ergohaven/ergohaven-zmk), imported via `config/west.yml` (revision `main`, so builds track upstream HEAD). A local checkout is usually at `../ergohaven-zmk` — read `boards/shields/<kb>/` there to see key-position transforms, `Kconfig.defconfig` defaults (keyboard name, split central role), and which shield names are valid.

Five keyboards are configured — **Op36** (36-key split), **Velvet V3**, **Velvet V3 UI** (trackball variant), **K03**, **Imperial44** — plus three standalone **Trackball** keymaps (v3.0, v3.1, Royal). Each keyboard has an English-only and a bilingual English/Russian (`_ruen`) keymap.

## Building

There is no local build, lint, or test. **All firmware is built by GitHub Actions**: `.github/workflows/build.yml` fires on push, PR, and `workflow_dispatch`, and delegates entirely to `ergohaven/ergohaven-zmk/.github/workflows/build-user-config.yml@main`.

The only way to verify a change compiles is to push and read the run:

```sh
git push                     # any branch triggers a build
gh run list --limit 5
gh run watch                 # or: gh run view --log-failed
```

Devicetree errors (wrong binding count, undefined behavior label, unknown keycode) surface only there.

### `build.yaml`

The authoritative target list. Only **Op36** targets and `settings_reset` are active; Velvet V3, Velvet V3 UI, K03, Imperial44, and all trackball entries are present but commented out. **Uncommenting is how you switch which keyboard gets built** — leaving everything enabled makes every push build ~30 firmwares.

Entry fields:

| Field | Notes |
|---|---|
| `board` | Always `ergohaven` (the trackball entries too). |
| `shield` | `{kb}_left`, `{kb}_right`, `{kb}_left_qube`, or the dongle triple `{kb}_qube qube dongle_screen`. Trackballs use bare `trackball`. |
| `keymap` | Optional. Resolves to `-DKEYMAP_FILE=config/{value}.keymap`. Used for `_ruen` variants and to pick between the three trackball keymaps. |
| `snippet` / `cmake-args` | `studio-rpc-usb-uart` + `-DCONFIG_ZMK_STUDIO=y` enable ZMK Studio. Applied to the central half and dongles, **not** to `{kb}_right` or `{kb}_left_qube`. |
| `artifact-name` | Required only when default naming would collide — i.e. every `_ruen` variant and every `qube` dongle target. |

Trackball entries additionally override the advertised name via cmake, e.g. `-DCONFIG_ZMK_KEYBOARD_NAME=\"EH\ TB\ v3.1\"`, and Royal flips both pointer axes with `-DCONFIG_PMW3610_INVERT_X=y -DCONFIG_PMW3610_INVERT_Y=y`.

### How config files get picked up

ZMK matches `.keymap`/`.conf` by **shield name**, stripping the `_left`/`_right` suffix for split halves. This drives the whole file layout:

- `op36_left` and `op36_right` → `op36.keymap` + `op36.conf`.
- `op36_qube qube dongle_screen` → `op36_qube.conf`. The base `op36.conf` does **not** apply, which is why each `{kb}_qube.conf` is a standalone two-liner (`CONFIG_ZMK_SLEEP=n`, `CONFIG_ZMK_POINTING=y`) rather than a delta.
- An explicit `keymap:` in `build.yaml` bypasses this entirely.

So editing `{kb}.conf` has no effect on the dongle build, and vice versa.

## Repository layout

```
config/
├── {kb}.keymap          # English-only
├── {kb}_ruen.keymap     # bilingual En/Ru
├── {kb}.json            # physical layout for ZMK Studio
├── {kb}_ruen.json       # SYMLINK -> {kb}.json  (layout is shared; never fork it)
├── {kb}.conf            # halves
├── {kb}_qube.conf       # dongle only — standalone, not a delta
├── keys_ru.h            # RU_* keycode macros, shared
└── west.yml
```

The `_ruen.json` files are symlinks. If you change a physical layout, edit the real `{kb}.json`.

## Keymap architecture

`.keymap` files are devicetree overlays: `behaviors {}`, `combos {}`, `macros {}`, then `keymap {}` with named layer nodes, plus optional `&trackball_listener {}` at the end. Layer *index* is the node's position in `keymap {}`; `display-name` is only cosmetic (it's what ZMK Studio and the dongle screen show).

Every keymap `#include "keys_ru.h"` — even the English-only ones, where nothing uses it.

### English-only keymaps — 4 layers

`0 base` · `1 nav` · `2 sym` · `3 adj`. Thumbs are `&mo 1` / `&mo 2`; `adj` (BT pairing, F-keys, `&bootloader`, `&studio_unlock`, output switching) is reached from `&mo 3` nested inside both nav and sym.

`velvet_v3` and `velvet_v3_ui` append four all-`&trans` `User0`–`User3` layers as scratch space for ZMK Studio edits. `velvet_v3_ui` also inserts `mouse` / `scroll` / `sniper` before them.

### RUEN keymaps — dual-base, dual-symbol

All five `_ruen` keymaps share one design:

- **Layers 0 (`en`) and 1 (`ru`) are both base layers.** They are switched with `&to 0` / `&to 1` (persistent), never `&mo`, so the keyboard stays on the active language between keystrokes. Same physical layout, different keycodes.
- The layer switch and the **OS input source must stay in sync**, so `layer_en` = `&to 0` + `to_en`, `layer_ru` = `&to 1` + `to_ru`, where `to_en`/`to_ru` emit `LS(LC(N1))` / `LS(LC(N2))` — the user's OS input-source shortcuts. Never bind `&to 0`/`&to 1` directly; always go through these macros.
- Switching is done by the **`cmben` / `cmbru` combos**: identical `key-positions`, scoped to opposite layers (`layers = <1>` and `layers = <0>`), so the same chord toggles both ways. Note that `layers = <0>` does **not** restrict a combo to the base layer — layer 0 is always active, so such a combo is live under every other layer too, including Op36's game layer.
- **Two symbol layers**, `sym_en` and `sym_ru`, reached from their respective base. `sym_ru` exists because HID keycodes produce different characters under a Russian input layout. Each key there uses one of three strategies:
  1. plain `&kp` — the character is layout-independent (`=`, `-`, `+`, `(`, `)`, `*`, `%`);
  2. an `RU_*` alias from `keys_ru.h` — same character, different physical key under the Ru layout (`RU_COMMA`, `RU_COLON`, `RU_SEMI`, `RU_FSLH`, `RU_QMARK`, `RU_DQT`, `RU_DOT`, `RU_BACKSLASH`, `RU_NUMERO`);
  3. the **`en` macro** — a `behavior-macro-one-param` that does `to_en` → press param → `macro_pause_for_release` → release → `to_ru`. Used for characters unreachable in the Ru layout (`#`, `<`, `>`, `` ` ``, `^`, `@`, `~`, `&`, `[`, `]`, `{`, `}`, `|`, `$`). It round-trips the OS input source per keypress, so use it only when 1 and 2 don't work.

**Layer indices differ between keymaps — do not copy `&mo N` across files.**

| | 0 | 1 | 2 | 3 | 4 | 5 | 6+ |
|---|---|---|---|---|---|---|---|
| `op36_ruen` | en | ru | **sym_en** | **sym_ru** | **nav** | adj | games, games_n, games_add |
| all other `_ruen` | en | ru | **nav** | **sym_en** | **sym_ru** | adj | (ui: mouse, scroll, sniper) |

Op36's Ru base right thumb is `&mo 3` where the En base has `&mo 2` — that offset is the whole mechanism for landing on the right symbol layer.

### Home row mods — Op36 only

`op36.keymap` and `op36_ruen.keymap` define `hml` / `hmr` hold-taps: `flavor = "balanced"`, `tapping-term-ms = <280>`, `require-prior-idle-ms = <150>`, `quick-tap-ms = <175>`, `hold-trigger-on-release`. `hml` lists only right-hand key positions in `hold-trigger-key-positions` (and `hmr` only left-hand ones), so a mod fires only for a cross-hand key release. Both files also retune the global `&mt` (`tap-preferred`, 200ms quick-tap, 125ms prior-idle).

Those position lists are **hard-coded 36-key indices** (0–35) and must be updated by hand if the layout changes. The 12-column boards have no home row mods — they put real modifiers on thumb and pinky keys.

The two Op36 keymaps assign **different mod orders**, so they are not interchangeable:

- `op36.keymap` — left `LGUI LALT LSHFT LCTRL`, right `RCTRL RSHFT LALT RGUI`
- `op36_ruen.keymap` — left `LGUI LALT LCTRL LSHFT`, right `RSHFT RCTRL LALT RGUI`

### Combos

- `kha` (positions 6+7) and `hrdsgn` (7+8) reach **Х** and **Ъ**, which have no home on a 36-key Cyrillic grid. In `op36_ruen` they're `&kp RU_CYRILLIC_HA` / `RU_CYRILLIC_HARD_SIGN` scoped to `layers = <1>`; in `op36.keymap` the same positions send the same HID keys as `&kp LBKT` / `&kp RBKT`.
- `io` (positions 19+29, layer 1) reaches **Ё** the same way.
- `to_games` (positions 18+28, `layers = <0 1 6>`) does `&tog 6`, toggling Op36's gaming cluster on and off — hence layer 6 in the scope list, so the same chord escapes it. It sits on the idle right half because toggling game mode is a deliberate session-level action, while a misfire mid-game is not.

- `CONFIG_ZMK_COMBO_MAX_KEYS_PER_COMBO=3` / `MAX_COMBOS_PER_KEY=7` in the `.conf` files cap what you can add.

**Op36's game cluster (layers 6–8) is built for left-half-only play with the mouse in the right hand.** Everything typeable must therefore be reachable from the left half, and the right half of all three layers is `&none` — blanking it is deliberate, since `&trans` would leave the right-hand home row mods live, expose Cyrillic when entering games from the Ru base, and put the symbol layer under the right thumb. Layer 6 (`games`) is the left hand's QWERTY block with no home row mods and `&mo 7` / `&mo 8` / SPACE on the thumbs. Layer 7 (`games_n`) holds the number row, ESC, ENTER, BSPC and P. Layer 8 (`games_add`) mirrors the right hand's columns 5–8 onto columns 3–0 and keeps the left inner column (T/G/B) in column 4 — it drops column 9, which is why P lives on layer 7.

Note that `layer_en` / `layer_ru` are the wrong primitive for switching chat language inside the cluster: their `&to 0` / `&to 1` deactivates layer 6. Raw `&to_en` / `&to_ru` only send the OS shortcut and leave the game layer on.

### `keys_ru.h`

Generated (Unicode-licensed) header of `RU_*` defines built on `ZMK_HID_USAGE(...)`. Each maps a Cyrillic character or Russian-layout symbol to the **HID usage the OS emits when its input source is Russian** — e.g. `RU_CYRILLIC_A` → keyboard `F`, `RU_CYRILLIC_YA` → `Z`, `RU_CYRILLIC_IO` → grave. Consult it before adding any Russian-mode key; if the character isn't defined there, wrap the English key in the `en` macro instead.

### Pointing devices

`velvet_v3_ui*.keymap` and the three `trackball_*.keymap` files end with a `&trackball_listener` node whose child nodes bind input-processor chains to **layer numbers**:

```
&trackball_listener {
    input-processors = <&zip_xy_scaler 9 20>;      /* base sensitivity */
    scroller { layers = <N>; ... };                /* Y-invert + zip_xy_to_scroll_mapper */
    sniper   { layers = <M>; input-processors = <&zip_xy_scaler 1 4>; };
};
```

Those layer numbers are raw indices into `keymap {}` — **reordering layers silently breaks the pointer behavior**, since nothing validates them. `velvet_v3_ui` additionally defines `zip_auto_mouse` (a `zmk,input-processor-temp-layer` with 800ms prior-idle) to pop the mouse layer on movement, `cap_sen` (hold = `&mo`, tap = `&mkp`, `hold-while-undecided`), and a `mouse_layer` sticky key. Pointing needs `CONFIG_ZMK_POINTING=y`, which currently lives only in the `*_qube.conf` files.

## Editing conventions

- **Binding count per layer must exactly match the key count in the matching `.json`** — 36 for Op36 (10/10/10 + 6 thumbs), 58 for Velvet V3 (12/12/12 + 10 thumbs), more for K03/Imperial44 which add encoders and inner keys. A mismatch is a build failure, only visible in the Actions log.
- Binding grids are column-aligned with spaces, with a wider gap marking the split halves. Preserve the alignment; keymaps get round-tripped through the ZMK Studio / Keymap Editor formatter.
- Adding a layer means renumbering every `&mo`/`&to`/`&tog` **and** any `&trackball_listener` `layers = <>` reference. Prefer appending.
- K03 and Imperial44 have encoders: `sensor-bindings` per layer (K03 defines an `enc_vol` `zmk,behavior-sensor-rotate`; Imperial44 uses `&inc_dec_kp` inline).
- `&studio_unlock` is on the `adj` layer of every keymap. Layers remapped at runtime through ZMK Studio persist in settings — flash the `settings_reset` firmware to clear them.
