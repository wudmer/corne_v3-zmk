# Corne + Prospector dongle (dongle-as-central)

This config builds the Corne as a **3-piece split**: the Prospector dongle
(Seeed XIAO nRF52840) is the **central** and holds the keymap; both Corne halves
are **peripherals**.

All three devices are built from this one repo on ZMK `main`, using the
Prospector module's `feat/new-status-screens` branch (Zephyr 4.1 / ZMK main).

## What GitHub Actions builds

Pushing this repo produces these `.uf2` artifacts:

| Artifact | Board | Flash to |
| --- | --- | --- |
| `corne_dongle prospector_adapter` | `xiao_ble` | Prospector dongle |
| `corne_left`  | `nice_nano_v2` | Left half |
| `corne_right` | `nice_nano_v2` | Right half |
| `settings_reset` | `nice_nano_v2` | Both halves (reset) |
| `settings_reset` | `xiao_ble` | Dongle (reset) |

## Flashing procedure (order matters)

Because the halves were previously bonded to each other (left↔right), you must
clear old bonds first or they won't pair to the dongle.

1. **Reset all three.** Put each device in bootloader mode (double-tap the reset
   button → a USB drive appears: `NICENANO` for the halves, `XIAO-SENSE` for the
   dongle) and drag the matching `settings_reset` `.uf2` onto it.
   - Halves → `settings_reset` (nice_nano_v2)
   - Dongle → `settings_reset` (xiao_ble)
2. **Flash real firmware.** Reset into bootloader again and drag:
   - Dongle → `corne_dongle prospector_adapter` `.uf2`
   - Left half → `corne_left` `.uf2`
   - Right half → `corne_right` `.uf2`
3. **Pair in order.** Power on the dongle, then power on the **left** half first,
   then the **right**. This order sets how the Prospector arranges the per-half
   battery widgets.
4. Plug the dongle into your computer via USB. The keyboard should now type, and
   the Prospector shows layer / battery / connection status.

## Notes / things you may want to change

- **Ambient light sensor:** `config/corne_dongle.conf` has
  `CONFIG_PROSPECTOR_USE_AMBIENT_LIGHT_SENSOR=n` with a fixed brightness so it
  builds whether or not your Prospector has the APDS9960 fitted. If you *did*
  install the sensor, set it to `y` for auto-brightness.
- **RGB / ext-power keys on the dongle:** the `extras` layer's `&rgb_ug` and
  `&ext_power` keys are `&trans` in `config/corne_dongle.keymap`, because the
  dongle has no LEDs and doesn't compile those subsystems. RGB underglow still
  runs on the halves with their start-up settings; it just isn't adjustable from
  the keyboard in this setup. Keep `corne_dongle.keymap` in sync with
  `corne.keymap` when you edit layers.
- **ZMK Studio** now runs on the dongle (`corne_dongle` build has the
  `studio-rpc-usb-uart` snippet).
- **To "un-dongle"** and go back to left-central 2-piece: restore the old
  `build.yaml` and `west.yml`.
