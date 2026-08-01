# Clover — Corne v3 + Prospector dongle: manual

A **3-piece split**: the Prospector dongle (Seeed XIAO nRF52840) is the BLE
**central** and holds the keymap; both Corne halves are **peripherals**.

> The dongle must be plugged in for the keyboard to work at all. It has no
> battery, so it always needs USB power — even when typing to a host over
> Bluetooth.

- [1. Daily use](#1-daily-use)
- [2. Flashing](#2-flashing)
- [3. Troubleshooting](#3-troubleshooting)
- [4. Hardware notes and limitations](#4-hardware-notes-and-limitations)
- [5. Config reference](#5-config-reference)

---

## 1. Daily use

### Layers

| # | Layer | How to reach it |
| - | ----- | --------------- |
| 0 | QWERTY  | default base |
| 1 | Colemak | Extras + `C` |
| 2 | Lower   | hold right thumb (`&lt 2`) |
| 3 | Upper   | hold left/right thumb (`&lt 3`) |
| 4 | Extras  | hold left inner thumb (`&lt 4`) |

Switch base layout: hold **left inner thumb**, then **`X`** → QWERTY, **`C`** → Colemak.

> **Layer order is load-bearing.** ZMK resolves a press from the highest active
> layer downwards (`app/src/keymap.c:714`), stopping at the first non-transparent
> binding. An alternate base layer must therefore sit *below* the momentary
> layers. Colemak used to be layer 4 (the highest) and shadowed everything: you
> could not leave it, and it had no numbers, symbols or arrows. Do not reorder
> these layers without re-checking that.

### Output switching (USB / Bluetooth)

On the **Extras** layer (hold left inner thumb):

| Key | Binding | Effect |
| --- | ------- | ------ |
| `A` | `&out OUT_USB` | force wired |
| `S` | `&out OUT_BLE` | force wireless |
| `D` | `&out OUT_TOG` | toggle |
| `Y` / `U` / `I` | `&bt BT_SEL 0/1/2` | pick Bluetooth host profile |
| `'` | `&bt BT_CLR` | clear the **current host profile** only |

5 host profiles exist (`BT_MAX_PAIRED − peripherals` = 7 − 2). When both USB and
BLE are live, ZMK uses the stored preference set by `&out`; when only one is
available it uses that (`app/src/endpoints.c:284`).

`&bt BT_CLR` is safe: it only clears the active host profile. Split peripheral
bonds live in a separate store (`ble/peripheral_addresses/N`), so host switching
can never break the halves' pairing.

### What the displays show

**Prospector (dongle):** active layer, per-half battery bars, connection status,
caps-word indicator.

**Half OLEDs:** battery % and connection state — and nothing else. ZMK gates the
layer, WPM and output-status widgets to the central, so a peripheral physically
cannot display them.

Battery bars are ordered by **pairing sequence**, not physical side. Pair left
before right or the sides appear swapped; there is no config option to swap them.

---

## 2. Flashing

Builds run in GitHub Actions on push to `config/**`, or via **Actions → Build ZMK
firmware → Run workflow**. Download the `firmware` artifact (a zip).

| .uf2 | Board | Flash to |
| ---- | ----- | -------- |
| `clover_dongle prospector_adapter-seeeduino_xiao_ble` | `seeeduino_xiao_ble` | Prospector dongle |
| `corne_left-nice_nano_v2` | `nice_nano_v2` | Left half |
| `corne_right-nice_nano_v2` | `nice_nano_v2` | Right half |
| `settings_reset-nice_nano_v2` | `nice_nano_v2` | halves — **only** for a bond reset |
| `settings_reset-seeeduino_xiao_ble` | `seeeduino_xiao_ble` | dongle — **only** for a bond reset |

Enter the bootloader by **double-tapping reset**; a drive appears (`XIAO-SENSE`
for the dongle, `NICENANO` for a half). Drag the matching `.uf2` on.

- Both halves report as `NICENANO` — they are indistinguishable, so keep track of
  which one you are holding before dropping `corne_left` vs `corne_right`.
- The copy ends with an `fchmod failed` / "disk not ejected properly" error. This
  is **normal** — the board reboots the moment the transfer completes.
- If a board shows up as `nice_nano` / `XIAO nRF52840 Sense` over USB instead of
  `Clover [crkbd]` / `Clover Dongle`, it is sitting in the bootloader. A single
  tap boots it into the real firmware.

### Routine update

Flash only the app `.uf2` to whichever boards changed. **Do not** flash
`settings_reset` — app flashing preserves BLE bonds, so nothing needs re-pairing.

### Full reset (only when bonds are broken)

Reflashing does **not** clear bonds; the settings partition survives. When bonds
are genuinely mismatched:

1. Flash `settings_reset` to **all three** devices (partial resets do not work —
   the surviving side keeps stale keys).
2. Flash the real firmware to all three.
3. Power up in order: **dongle (USB) → left → right**, waiting for each to
   connect. This order also fixes the battery-bar sides.

---

## 3. Troubleshooting

### Halves show "connected" but no keys type

**Stale BLE bonds.** The dongle log shows:

```
bt_smp: pairing failed (peer reason 0x3)
zmk: Security failed: ... err 4
zmk: split_central_notify_func: [UNSUBSCRIBED]
```

The half connects and even subscribes to the key-position characteristic, then
security fails and the subscription is torn down — so zero key events arrive.
Fix: the [full reset](#full-reset-only-when-bonds-are-broken) above.

### A battery bar on the dongle is blank

The central reads the peripheral's battery **once** on connect
(`central.c:638`) and then relies on notifications — but ZMK only notifies when
the level **changes** (`battery.c`), and the Prospector widget **drops any event
that arrives before it has initialised** (`battery_bar.c:31`, `:54`). If that one
read lands too early it is discarded, and with a steady cell voltage nothing ever
re-sends it, so the slot stays blank indefinitely.

Fixes, cheapest first: power-cycle that half; or boot **dongle first**, let its
screen come up, then power the halves; charging a cell also forces a change and
therefore a notify.

Note also that a half **stops sampling its battery entirely once idle**
(`battery.c` stops the timer on IDLE/SLEEP), so values freeze after
`ZMK_IDLE_TIMEOUT` (10 min) until you type again.

### A single key does nothing (others in its row and column work)

Not a firmware problem. If other keys share that row *and* that column and work,
both GPIOs and both traces are proven good, so the fault is isolated to that one
key's switch, socket or diode.

Narrow it down without tools:

1. Press the key on another layer (e.g. Lower gives a digit). Dead on every layer
   means the matrix never sees it.
2. Short the **socket** contacts — this bypasses the switch. Still dead ⇒ the
   switch is fine.
3. Short the **PCB pads** (not the socket's internal contacts). If that works but
   step 2 did not, the socket has a cold joint or lifted pad.

**Diode orientation is the classic cause.** This board is `col2row`, so a diode
fitted backwards blocks the scan current and the key is permanently dead while
its row and column keep working. Every diode's cathode stripe must point the same
way — compare against neighbours rather than trusting the silkscreen. Rotate or
replace (1N4148W, SOD-123), and sweep the rest of the board while it is open,
since a mis-placed diode is rarely the only one.

Other candidates, in order: cold joint on a socket leg, dead diode (diode mode
should read ~0.6 V one way and open the other), hairline trace break.

### Dongle lags, ignores layer keys, or reboots

LCD redraws starving key/BLE handling, and/or insufficient USB power. Already
mitigated in-config with `CONFIG_ZMK_DISPLAY_WORK_QUEUE_DEDICATED=y` and
brightness 40. If it recurs: plug the dongle **directly into the computer** with
a known-good data cable — a dock/hub shared with other devices was a real cause
here.

### Reading serial logs

Only one snippet can be passed per build target, and `ZMK_STUDIO` requires
`studio-rpc-usb-uart`, so **Studio and USB logging are mutually exclusive**. To
debug: comment out the three `CONFIG_ZMK_STUDIO*` lines in
`config/clover_dongle.conf` and set the dongle's snippet in `build.yaml` to
`zmk-usb-logging`. Revert afterwards.

On macOS the port must be opened **read-write** to assert DTR, otherwise a Zephyr
CDC console stays silent. Capture across a reboot, since the interesting events
happen in the first few seconds.

---

## 4. Hardware notes and limitations

- **Controllers:** the halves use nRF52840 "pro micro" / SuperMini clones, not
  genuine nice!nano. They sense battery via `zmk,battery-nrf-vddh`, which follows
  the USB rail — so **100% while plugged in is expected and meaningless**.
- **18650 batteries:** ZMK maps `≥4200 mV → 100%`, **`≤3450 mV → 0%`**, else
  `mv*2/15 − 459` (`battery.c`). So 3.6 V → 21%, 3.9 V → 61%. Onboard charging is
  only ~100 mA, which cannot realistically fill a 3000 mAh cell (20–30 h) —
  **charge 18650s in an external charger**. Confirm cells are Li-ion (4.2 V full),
  not LiFePO4 (3.65 V full), which would always read ~0% and is unsafe to charge
  to 4.2 V.
- **RGB cannot be controlled from the keyboard.** A keyless central cannot compile
  `&rgb_ug` / `&ext_power` (no `zmk,underglow` node), so those 12 keys are
  `&trans` in `config/clover_dongle.keymap`. Underglow still runs on the halves
  from their start-up settings.
- **Deep sleep is disabled** (`CONFIG_ZMK_SLEEP` is commented out), so
  `ZMK_IDLE_SLEEP_TIMEOUT` is inert. The halves idle after 10 min (OLED blanks,
  RGB off, battery sampling stops) but never power down.
- **No reset buttons on the halves** — entering the bootloader means opening the
  case. Consider soldering reset leads.
- **Switches are Gateron Low Profile (GLP/KS-33):** plate-mount, MX 14×14 cutout,
  19×19 spacing — so standard **Corne v3 / MX** plates and cases fit. Kailh Choc
  parts (13.8 mm, 18×17) do **not**.

---

## 5. Config reference

| File | Purpose |
| ---- | ------- |
| `config/west.yml` | ZMK pinned to **v0.3**, prospector-zmk-module `main` |
| `.github/workflows/build.yml` | **must** stay pinned to `build-user-config.yml@v0.3` — the `@main` workflow throws `KeyError: 'qualifiers'` on v0.3 boards |
| `build.yaml` | build matrix; halves forced to peripheral via `-DCONFIG_ZMK_SPLIT_ROLE_CENTRAL=n` |
| `config/clover_dongle.keymap` | **the keymap that is actually flashed** (dongle = central) |
| `config/corne.keymap` | kept in sync for reference; RGB keys are real here |
| `config/clover_dongle.conf` | Prospector display, Studio, BLE TX power |
| `config/corne.conf` | halves: OLED, RGB, idle, battery reporting |
| `config/boards/shields/clover_dongle/` | dongle shield — **must** live in its own folder, or `corne.conf` gets merged into the keyless dongle build and breaks it |

Two Kconfig traps worth remembering:

- A Kconfig **`select` cannot be overridden** on the command line. The left shield
  originally used `ZMK_SPLIT_BLE_ROLE_CENTRAL` (a `select`); it is now a plain
  `ZMK_SPLIT_ROLE_CENTRAL default y` so `build.yaml` can force it to `n`.
- `ZMK_DISPLAY_STATUS_SCREEN_BUILT_IN` and `..._CUSTOM` are a `choice` — setting
  both is meaningless.

### Regenerating the keymap diagram

```sh
uvx --from keymap-drawer keymap parse -z config/clover_dongle.keymap \
  | sed 's|^layout: .*|layout: {qmk_keyboard: crkbd/rev1, layout: LAYOUT_split_3x6_3}|' \
  > assets/keymap_parsed.yaml
uvx --from keymap-drawer keymap -c assets/keymap draw assets/keymap_parsed.yaml > assets/keymap.svg
```

### Reverting to a 2-piece split

Restore the pre-dongle `build.yaml` and `west.yml`, drop the
`-DCONFIG_ZMK_SPLIT_ROLE_CENTRAL=n` cmake-args, and `settings_reset` both halves
before re-pairing them to each other.
