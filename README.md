# Ergohaven OP36 — ZMK config

Personal [ZMK](https://zmk.dev) firmware config for the
[Ergohaven OP36 (Omega Point 36)](https://ergohaven.xyz/) split keyboard.

This repo is a ZMK *user config*: a west manifest plus Kconfig and keymap
files in `config/`. The board definitions and firmware come from
[ergohaven/ergohaven-zmk](https://github.com/ergohaven/ergohaven-zmk)
(Ergohaven's ZMK fork), pulled in via `config/west.yml` — it is not vendored
here.

This repo focuses on a single keymap: **`op36_ruen`** — a dual-language
EN (QWERTY) / RU (ЙЦУКЕН) layout with lockstep layout switching, plus a mouse
layer with ramping speed and a momentary precision mode.

## Repository layout

| Path | Purpose |
|---|---|
| `config/west.yml` | West manifest: fetches `ergohaven-zmk` (boards + firmware) |
| `config/op36.conf` | Kconfig for OP36 builds: sleep/soft-off, battery reporting, combo limits, pointing |
| `config/op36_qube.conf` | Kconfig overlay for Qube builds (no sleep, pointing on) |
| `config/op36.json` | Physical layout geometry (KLE-style JSON) used by layout renderers |
| `config/op36.keymap` | Stock QWERTY keymap (base/nav/sym/adj) for the plain OP36 builds |
| `config/op36_ruen.keymap` | **The main keymap** — see below |
| `config/op36_ruen.json` | Symlink to `op36.json` (same geometry, different keymap name) |
| `config/keys_ru.h` | Cyrillic letter and RU-layout symbol keycodes (HID usage IDs) |
| `build.yaml` | Build matrix for CI |
| `.github/workflows/build.yml` | Calls Ergohaven's reusable user-config build workflow |

## The `op36_ruen` keymap

### Layers

Layer number = position of the node in the `keymap` tree; `&mo N` / `&lt N` /
`&to N` reference it.

| # | Name | Access | Content |
|---|---|---|---|
| 0 | `en` (Base) | base layer when OS layout is English | QWERTY + home-row mods (`hml`/`hmr`) |
| 1 | `ru` | base layer when OS layout is Russian | ЙЦУКЕН + home-row mods |
| 2 | `sym_en` | left thumb (mo 2) on EN base | EN symbols |
| 3 | `sym_ru` | left thumb (mo 3) on RU base | RU symbols; keys with no RU-layout keycode use the `&en` escape (e.g. `<`, `>`, `^`, `[`, `]`, `'`, `&`, `|`, `@`) |
| 4 | `nav` | space (mo 4) on EN base | digits, arrows, scroll, PrintScreen; holds `adj` (mo 6); **language switch keys on Z / P** |
| 5 | `nav_ru` | space (mo 5) on RU base | same as `nav` for the RU state |
| 6 | `adj` | thumb button (mo 6) from sym/nav layers | F1–F12, BT device select/clear, USB/BLE output toggle, media, `studio_unlock`, **layout recovery anchors** |
| 7 | `mouse` | ENTER hold (`lt 7`) from base | mouse buttons, scroll, ramping movement |
| 8 | `mouse_precision` | left thumb (mo 8) while in `mouse` | constant-speed (half) movement |

Key facts:

- Home-row modifiers: `A S D F` (left) and `H J K L ;` (right) hold
  GUI/ALT/SHFT/CTRL; releasing on any key of the *opposite* half triggers the
  modifier (see `hold-trigger-key-positions` in the keymap).
- Combos (RU base layer only): `O+P` → **Х** (`kha`), `M+,` → **Ъ** (`hrdsgn`).
- The `en` macro (defined in the keymap) types a symbol the RU layout has no
  key for by relative-flipping the OS to EN, tapping the keycode, and flipping
  back — reserved for `sym_ru`-style escapes.

### Language switching design

Core invariant, kept by construction:

> **ZMK `ru` layer ON ⟺ OS keyboard layout is Russian.**

- **Primary:** per-base nav layers each have one *directional cycle* key for
  the opposite language: `P` in `nav` (reached from the EN base) and `Z` in
  `nav_ru` (from the RU base). The `cycle_to_*` macros send `Super+Space`
  (host "next input source") followed by `&to N`, flipping the OS layout and
  forcing the ZMK base layer in lockstep. No host keybinding is needed where
  Super+Space already toggles layouts (GNOME does, by default).
- **Current-language re-assert:** the other key (`Z` in `nav`, `P` in
  `nav_ru`) stays absolute via `to_en` / `to_ru` — `Meta+Ctrl+1` → English,
  `Meta+Ctrl+2` → Russian (e.g. KDE's "Switch keyboard layout to …"
  shortcuts). It is a no-op on hosts without those bindings, and heals desync
  where they exist. Digits, not letters: the RU layout remaps letters, which
  would break keysym-matched host shortcuts.
- **Recovery:** `layer_en` / `layer_ru` on the adj layer's outer thumbs (pos
  30 / 35). Absolute: force the ZMK base layer, and the OS layout where the
  shortcut is bound. Use when parity is lost — after jumping to a machine
  already on the other layout, or after switching layout via the OS tray /
  Super+Space.
- Never change the layout from the OS side while this keyboard is the input
  authority — it silently breaks the invariant.
- The full rationale (including the generic `cycle_lang` macro, currently
  unbound and reserved) lives in the comment block above the `macros` section
  of `op36_ruen.keymap`. Keep that comment in sync with any switching change.

### GNOME environment

GNOME Shell claims `Super+Ctrl+1..9` by default:
`org.gnome.shell.keybindings open-new-window-application-N` opens a new window
of the Nth application in the dock. That collides with `to_en` / `to_ru`
(`Meta+Ctrl+1/2`), which used to open dock apps instead of affecting the
layout. On this machine those two bindings were removed:

```sh
gsettings set org.gnome.shell.keybindings open-new-window-application-1 "[]"
gsettings set org.gnome.shell.keybindings open-new-window-application-2 "[]"
```

With them cleared, the absolute macros are harmless no-ops on GNOME, and
primary switching goes through the cycle keys plus GNOME's own layout toggle:

```
org.gnome.desktop.wm.keybindings switch-input-source = <Super>space
org.gnome.desktop.input-sources sources = [('xkb', 'us'), ('xkb', 'ru')]
```

Useful checks:

```sh
gsettings get org.gnome.shell.keybindings open-new-window-application-1  # @as []
gsettings get org.gnome.desktop.wm.keybindings switch-input-source        # ['<Super>space', 'XF86Keyboard']
```

Consequences per host:

- **GNOME:** the cycle keys work with no host configuration. To repair a
  desync (layout changed from the OS side), switch the OS layout manually
  (tray / Super+Space) and press the matching adj anchor — its host-shortcut
  half is a no-op here.
- **KDE / Sway:** keep `Meta+Ctrl+1/2` bound to "switch to layout EN/RU" for
  the absolute keys, and also bind `Super+Space` to the layout toggle so the
  cycle keys keep the invariant there too.

### Mouse

- Base speed is `ZMK_POINTING_DEFAULT_MOVE_VAL` (1300) — the single speed
  knob. Ramping: acceleration exponent 2, time-to-max 300 ms.
- Precision mode: momentary layer stacked over `mouse` (left thumb, pos 31),
  constant speed = half of base (650). No latched state — release to go back
  to ramping.
- Left home row: `A`=MB2, `S`/`D`=scroll down/up, `F`=MB1, `G`=MB3.
- Right home row: `J K L ;` = move left/down/up/right.

### Key positions

36 keys: 18 per half, numbered row-major with the split gap unnumbered.

- Left half: `0–4`, `10–14`, `20–24`, `30–32`
- Right half: `5–9`, `15–19`, `25–29`, `33–35`
- Thumb row: `30`=TAB, `31`=thumb (mo 4 / mo 8), `32`=SPACE, `33`=ENTER
  (lt 7), `34`=thumb (mo 2/3), `35`=BACKSPACE

## Building

### CI (the way to build)

Every push runs `.github/workflows/build.yml` → Ergohaven's reusable
`build-user-config.yml`, using `build.yaml` as the matrix. Artifacts (UF2) are
merged into a single `firmware` archive on the run.

| Artifact | Hardware |
|---|---|
| `op36_left-ergohaven-zmk` | left half, stock keymap (`op36.keymap`), ZMK Studio enabled |
| `op36_left_ruen-ergohaven-zmk` | **left half, `op36_ruen`** (ZMK Studio enabled) |
| `op36_right-ergohaven-zmk` | right half (companion of whatever the left runs) |
| `op36_qube-ergohaven-zmk` | Qube variant (display + dongle screen), stock keymap |
| `op36_qube_ruen-ergohaven-zmk` | Qube variant, `op36_ruen` |
| `op36_left_qube-ergohaven-zmk` | left half with Qube (no display UI) |

Flash by dragging the `.uf2` into the keyboard's USB mass-storage bootloader,
or over ZMK Studio (Studio is enabled via `CONFIG_ZMK_STUDIO=y` + the
`studio-rpc-usb-uart` snippet in most entries).

### Local build

Mirrors the CI invocation (needs a west toolchain with Zephyr — see
[docs.zmk.dev](https://docs.zmk.dev) / [docs.ergohaven.com](https://docs.ergohaven.com)):

```sh
west init -l config
west update
west zephyr-export

# left half, op36_ruen:
west build -s zmk/app -b ergohaven -S "studio-rpc-usb-uart" \
  -- -DZMK_CONFIG="$PWD/config" \
     -DKEYMAP_FILE="$PWD/config/op36_ruen.keymap" \
     -DSHIELD="op36_left"
```

Swap `-DSHIELD` for `op36_right` (companion), `op36_qube qube dongle_screen`
(Qube), etc., and drop `-DKEYMAP_FILE` for the stock `op36.keymap`. Flash with
`west flash` (nrfjprog) or drag the UF2.

## Editing the keymap

- [Ergohaven's web keymap editor](https://keymap.ergohaven.com) can open this
  repo and commit changes — some history entries are authored by
  `keymap-editor[bot]`. It renders `op36.json` geometry, so keep it in sync
  with any physical layout change.
- Manual editing: `.keymap` files are devicetree source (C preprocessor
  available — `keys_ru.h` is included on purpose).

## References

- [ZMK documentation](https://docs.zmk.dev) — keymaps, behaviors, layers
- [Ergohaven ZMK docs](https://docs.ergohaven.com) — building & flashing Ergohaven boards
- [ergohaven/ergohaven-zmk](https://github.com/ergohaven/ergohaven-zmk) — the upstream firmware fork this repo builds against
