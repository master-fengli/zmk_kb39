# AGENTS.md

ZMK firmware user config for the **Keyball39** — a 39-key split keyboard with a PMW3610 trackball on the right half, OLED displays, and nice_nano_v2 controllers. There is no application code here; everything is Zephyr devicetree, Kconfig, and ZMK keymap config.

## Build

There is **no local build setup** (no west workspace here). Firmware is built by GitHub Actions:

- `.github/workflows/build.yml` calls ZMK's reusable `build-user-config.yml@v0.3` workflow.
- `build.yaml` (repo root) defines the build matrix: `keyball39_left`, `keyball39_right` (both with the `studio-rpc-usb-uart` snippet for ZMK Studio), plus `settings_reset`.
- Dependencies are pinned in `config/west.yml`: ZMK **v0.3** plus the out-of-tree trackball driver `tangbonze/zmk-pmw3610-driver` (branch `main`).

To "test" changes, push and let CI build — or reason carefully about devicetree/Kconfig correctness before committing.

## Layout / where things live

- `config/keyball39.keymap` — the keymap (6 layers). This is the file most edits target.
- `config/keyball39.conf` — global firmware options (BLE tuning, display, ZMK Studio, deep sleep 15 min).
- `config/keyball39.json` — physical layout used by the ZMK keymap editor / keymap-drawer. Kept deliberately (see commit "Restore layout for keymap editor"). If you change the physical layout in the `.dtsi`, mirror it here.
- `config/boards/shields/keyball_nano/` — the shield definition:
  - `keyball39.dtsi` — shared: physical layout, matrix transform (12 cols × 4 rows), kscan row GPIOs, i2c0 + SSD1306 OLED.
  - `keyball39_left.overlay` — left half: only col GPIOs.
  - `keyball39_right.overlay` — right half: col GPIOs, `col-offset = <6>` on the transform, spi1 + PMW3610 trackball node with `scroll-layers`/`snipe-layers`, and the `trackball_listener` input listener.
  - `keyball39_right.conf` — trackball driver options (CPI, snipe CPI, scroll tick, orientation 180°, power management). `keyball39.conf` and `keyball39_left.conf` in this folder are intentionally empty.
  - `Kconfig.defconfig` — the **right** half is the BLE split central (`ZMK_SPLIT_BLE_ROLE_CENTRAL`), not the left. Also sets display/LVGL defaults.
- `keymap-drawer/` — empty; the keymap-drawing workflow was removed (commit "Remove keymap drawing workflow"). Don't re-add drawing automation unless asked.

## Keymap conventions

- Layers by index: 0 `QWRT` (default), 1 `LH`, 2 `RH`, 3 `SYM`, 4 `NUM`, 5 `SNIPE`.
- Layer indices are cross-referenced in `keyball39_right.overlay`: `scroll-layers = <1 3 4>` and `snipe-layers = <5>`. **If you add/reorder layers, update these devicetree properties too.**
- Home-row mods use `&mt` (tap-preferred, 200 ms) and layer-taps use `&lt` (balanced, 240 ms); global tweaks are at the top of the keymap file.
- Binding rows are whitespace-aligned into columns matching the physical layout (5+5 per row, thumb row uses `&none` fillers for the 12-column transform). Preserve this alignment when editing.
- Mouse buttons (`&mkp LCLK/RCLK`) live on layers 1/2 since the trackball is used with the right hand.

## Gotchas

- Row 4 of the matrix transform is irregular: `RC(3,5)` and `RC(3,11)` are the extra thumb keys, and the right side only has `RC(3,10)` and `RC(3,6)` — total 39 keys, not 40.
- The right overlay's SPI uses the same pin for MOSI and MISO (`P0.10`) — this is correct for the PMW3610's 3-wire SPI, don't "fix" it.
- ZMK Studio is enabled (`CONFIG_ZMK_STUDIO=y`, locking off), so users may also edit the keymap live; keymap file remains the source of truth for builds.
- Indentation in the devicetree files is a mix of tabs and spaces — match the surrounding text exactly when editing.
