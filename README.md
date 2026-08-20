# zmk-config

ZMK firmware user configuration for a **Corne** split keyboard (`corne_left` / `corne_right`)
running on nice_nano_v2-compatible controllers. Firmware is built by GitHub Actions on every push
— see `CLAUDE.md` for build/CI details and repo layout. This file documents what's actually
implemented in `config/corne.keymap` and `config/corne.conf`.

## Layers

| # | Name | Purpose |
|---|------|---------|
| 0 | `default_layer` | Base QWERTY, home row mods, outer-column Shift/Ctrl |
| 1 | `lower_layer` | Numbers, arrow/paging keys, Bluetooth profile select |
| 2 | `raise_layer` | Symbols |
| 3 | `adjust_layer` | Bootloader / factory reset / BT bond clear (hidden) |

Layer 1 and 2 are reached with the momentary thumb keys (`LWR`, `RSE`). Layer 3 has **no dedicated
key** — it's a tri-layer reached only by holding `LWR` + `RSE` together (see Adjust layer, below),
so it can't be triggered by accident during normal typing or while using layer 1 or 2 alone.

### Default layer

```
|  TAB/SHFT |  Q  |  W  |  E  |  R  |  T  |   |  Y  |  U   |  I  |  O  |  P  | BKSP |
| ESC/SHFT | A/GUI | S/ALT | D/CTRL | F/SHFT |  G  |   |  H  | J/SHFT | K/CTRL | L/ALT | ;/GUI |  '   |
| CTRL |  Z  |  X  |  C  |  V  |  B  |   |  N  |  M   |  ,  |  .  |  /  | DEL  |
                   | GUI | LWR | SPC |   | ENT | RSE  | ALT |
```

### Lower layer

```
|   TAB    |  1  |  2  |  3  |  4  |  5  |   |  6   |  7  |  8    |  9    |  0    | DEL  |
| ESC/SHFT | PLAY| LFT | DWN |  UP | RGT |   | LFT  | DWN |  UP   | RGT   | VOL-  | VOL+ |
|   CTRL   | BT0 | BT1 | BT2 | BT3 | BT4 |   | HOME | END | PG_UP | PG_DN | PREV  | NEXT |
                       | GUI |     | SPC |   | ENT  |     |  ALT  |
```

Media/transport keys (`PLAY`, `VOL-`/`VOL+`, `PREV`/`NEXT`) fill slots that were previously
transparent. The thumb-row `&trans` entries under `LWR`/`RSE` are deliberately left alone — they're
what let holding `LWR`+`RSE` together fall through to the Adjust layer (see below); repurposing them
would break that shortcut.

### Raise layer

```
|    `     |  !  |  @  |  #  |  $  |  %  |   |  ^  |  &  |  *  |  (  |  )  | BKSP |
| ESC/SHFT |  ;  |  :  |     |     |     |   |  -  |  =  |  [  |  ]  |  \  |  `   |
|   CTRL   |     |     |     |     |     |   |  _  |  +  |  {  |  }  | "|" |  ~   |
                       | GUI |     | SPC |   | ENT |     | ALT |
```

`;` and `:` get dedicated one-tap keys here — `;` is otherwise only reachable via the home-row-mod
key's tap (see below), and `:` previously had no direct binding at all (only Shift+`;` as a
cross-hand chord).

### Adjust layer

Active only while `LWR` + `RSE` are both held down. Everything not listed below is transparent
(falls through to layer 2, then 1, then 0).

```
| BOOT |     |     |     |     |     |   |     |     |     |     |     | BOOT  |
|      |     |     |     |     |     |   |     |     |     |     |     |       |
| RESET|     |     |     |     |     |   |     |     |     |     |     | BTCLR |
                   |     |     |     |   |     |     |     |
```

- **BOOT** (top-left and top-right): reboot into UF2 bootloader. `&bootloader` is a
  peripheral-local behavior in ZMK, so pressing the left-corner key reboots the left half and the
  right-corner key reboots the right half — either side can be re-flashed independently without
  needing the other half connected.
- **RESET** (bottom-left): full device reset (`&sys_reset`).
- **BTCLR** (bottom-right): clears the Bluetooth bond for the currently-selected profile
  (`&bt BT_CLR`), so the next connection to that profile starts a fresh pairing.

## Home row mods

`A S D F` (left hand) and `J K L ;` (right hand) are hold-taps: tapping still types the letter,
holding sends a modifier. Tap behavior is unchanged from a plain QWERTY layer.

| Key | Hold | Key | Hold |
|-----|------|-----|------|
| A | GUI | J | Shift |
| S | Alt | K | Ctrl |
| D | Ctrl | L | Alt |
| F | Shift | ; | GUI |

These use two custom hold-tap behaviors (`&hml` / `&hmr`, defined in the keymap's `behaviors {}`
block) rather than stock `&mt`, tuned specifically to avoid misfires while typing fast:

- **Positional**: each behavior's `hold-trigger-key-positions` only includes the *opposite* hand's
  keys (plus its thumbs). A same-hand roll (e.g. typing "sad") always resolves as tap+tap; only a
  cross-hand chord (e.g. hold `D`, tap a right-hand key) can resolve as hold+tap. This is the
  standard "home row mods done right" pattern and is the main defense against accidental
  modifiers during normal typing.
- **Timing**: `flavor = "balanced"`, `tapping-term-ms = 280`, `quick-tap-ms = 175`,
  `require-prior-idle-ms = 150` — the idle-time gate adds an extra safety margin on top of the
  positional check for fast typists.

The existing outer-column Shift/Ctrl mod-taps (`TAB`/`ESC` on the pinky column, present on all
four layers) are unrelated and unchanged — they use ZMK's stock `&mt` with default timing.

## Combos

| Keys | Action | Scope |
|------|--------|-------|
| `F` + `J` | `&caps_word` (toggle Caps Word) | Default layer only |

Scoped to layer 0 only (`layers = <0>`) because the same two physical positions are arrow keys
(`UP`/`DOWN`) on the lower layer — ZMK combos are active on *all* layers by default, so this scope
is required, not just defensive.

## Firmware/hardware features (`config/corne.conf`)

- **Sleep**: `CONFIG_ZMK_SLEEP=y`, idle timeout 5 minutes (`CONFIG_ZMK_IDLE_SLEEP_TIMEOUT=300000`).
- **OLED display**: enabled, with output/layer/WPM/battery status widgets.
- **RGB underglow**: wired in hardware (`boards/nice_nano_v2.overlay`, 27-LED WS2812 strip) but
  currently **disabled** at the firmware level (`CONFIG_ZMK_RGB_UNDERGLOW=n`); the startup
  effect/brightness settings in `corne.conf` are inert until that's flipped to `y`.
- **Bluetooth**: boosted TX power (`CONFIG_BT_CTLR_TX_PWR_PLUS_8=y`), 5 profile slots selectable
  from the lower layer (`BT0`-`BT4`), bond clearing reachable from the adjust layer.
