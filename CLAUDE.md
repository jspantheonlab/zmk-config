# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

A [ZMK Firmware](https://zmk.dev) user config repo for a **Corne** split keyboard (`corne_left` /
`corne_right` shields, built into upstream ZMK) running on **nice!nano v2** controllers. It does not
contain the ZMK firmware source itself — that's pulled in as a west dependency (`config/west.yml`
points at `zmkfirmware/zmk@main`). This repo only holds the user-level customization: keymap,
board overlay, and build matrix.

Since ZMK moved nice!nano boards to Zephyr's revisioned board model, the board identifier is
**`nice_nano@2//zmk`** (revision 2.0.0 = nice!nano v2, `//zmk` is a required variant qualifier) —
not the old bare `nice_nano_v2` name. Because `revision: main` in `config/west.yml` tracks ZMK's
rolling `main` branch, upstream naming/API changes like this can break the build without any local
change — if CI ever fails with `Invalid BOARD`, check whether the board identifier in `build.yaml`
has drifted from whatever ZMK's current `main` expects.

## Build

There is no local build here — firmware is compiled by GitHub Actions using ZMK's reusable
workflow (`.github/workflows/build.yml` → `zmkfirmware/zmk/.github/workflows/build-user-config.yml`).
Push to any branch (or open a PR) and check the Actions tab for `.uf2` firmware artifacts.

On every push to `main` specifically (not PRs, not other branches), a second job in the same
workflow downloads the merged `firmware` artifact and publishes it as a GitHub Release tagged
`firmware-<date>-<short-sha>`, so a permanent, easy-to-find `.uf2` download exists per merge
without having to dig through Actions run history. `old-main-4116d59` (a long-lived rollback
branch with only the original pre-UX-enhancements keymap) is release-enabled the same way, tagged
`firmware-old-main-4116d59-<date>-<short-sha>` so it's clearly distinguished from `main`'s builds.
Adding another long-lived release branch means adding its ref to this job's `if:` condition.

`build.yaml` defines the GitHub Actions build matrix — currently one entry per half:
`nice_nano@2//zmk` + `corne_left`, `nice_nano@2//zmk` + `corne_right`. Add board/shield combos here
(or use `include:` for one-off cmake-arg variants) rather than creating new workflow jobs.

To build locally instead (requires a working Zephyr/west toolchain), from a west workspace with
this repo as `config/`:

```sh
west build -d build/left -b nice_nano@2//zmk -- -DSHIELD=corne_left -DZMK_CONFIG=$PWD/config
west build -d build/right -b nice_nano@2//zmk -- -DSHIELD=corne_right -DZMK_CONFIG=$PWD/config
```

There is no lint/test suite in this repo — validation happens by letting the CI build succeed
(or fail with a devicetree/Kconfig error) and by flashing the resulting `.uf2` to hardware.

## Layout and how the pieces fit together

- **`config/west.yml`** — west manifest; pins the ZMK firmware revision (`main`) that everything
  else is compiled against. `self.path: config` tells west this repo's config lives in `config/`.
- **`config/corne.keymap`** — the actual keymap, in devicetree syntax. Four layers:
  `default_layer` (base QWERTY + outer-column Shift/Ctrl via stock `&mt`, plus home row mods on
  A-S-D-F/J-K-L-; via custom `&hml`/`&hmr` hold-tap behaviors), `lower_layer` (numbers, arrows,
  Bluetooth profile selection via `&bt BT_SEL n`, paging keys), `raise_layer` (symbols), and
  `adjust_layer` (bootloader/reset/BT-clear — no dedicated key, reached only via the
  `conditional_layers` tri-layer node when `lower_layer`+`raise_layer` are held together). Also
  defines a `combos {}` node (F+J → Caps Word, default layer only). Layers 1/2 are toggled with
  `&mo 1` / `&mo 2` from the base layer. Each layer's bindings block is preceded by an ASCII-art
  comment showing the physical key layout — **keep that comment in sync when editing bindings**,
  it's the only human-readable map of which binding is which key. See `README.md` for the full
  keymap feature reference (layer tables, home-row-mod tuning rationale, combo/adjust-layer
  details).
- **`config/corne.conf`** — Kconfig overlay: enables sleep (`CONFIG_ZMK_SLEEP`), underglow
  (`CONFIG_ZMK_RGB_UNDERGLOW`) with startup effect/brightness, the OLED display and its status
  widgets (output/layer/WPM/battery), boosted BLE TX power, and idle sleep timeout (5 min). Toggle
  features here rather than in the keymap file.
- **`boards/nice_nano.overlay`** — board-level devicetree overlay, auto-applied by Zephyr to any
  build targeting the `nice_nano` board (any revision/variant — matched by base name only, per
  Zephyr's `boards/<BOARD>.overlay` convention). Wires up an SPI-driven WS2812 LED strip (27 LEDs,
  GRB order) as `zmk,underglow` for the RGB underglow feature referenced in `corne.conf`. Must be
  renamed if the board name ever changes again upstream.
- **`boards/nice_nano_v2.dts`** — dead file, not part of the build: it declares an unrelated
  `joric,nrfmicro`-compatible board and `#include`s two `.dtsi` files (`nrfmicro-pinctrl.dtsi`,
  `arduino_pro_micro_pins.dtsi`) that don't exist anywhere in this repo or its west dependencies, so
  it could never have compiled. It also isn't in the `boards/<vendor>/<board>/board.yml` layout
  Zephyr's board discovery requires, so it's silently ignored by the build. Safe to delete; kept
  only because no one has confirmed it's safe to remove yet.
- **`boards/shields/`** — currently empty (placeholder only); the `corne_left`/`corne_right`
  shields referenced in `build.yaml` come from upstream ZMK, not from this repo.
- **`zephyr/module.yml`** — declares this repo as a Zephyr module with `board_root: .`, which is
  what lets the custom board definitions under `boards/` be discovered by the build.

## Making changes

- Keymap edits: change `config/corne.keymap`, update the matching ASCII comment, done — no rebuild
  step needed locally, CI handles it.
- Feature toggles (underglow, display, sleep behavior): edit `config/corne.conf`.
- Hardware-level changes (pinout, LED count/type, power control): edit files under `boards/`.
- After pushing, get firmware from the GitHub Actions run's build artifacts.
