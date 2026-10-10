# dya-for-charybdis

ZMK firmware config for a **Charybdis** split keyboard on [cormoran's ZMK fork](https://github.com/cormoran),
wired up for **[DYA Studio](https://studio.dya.cormoran.works/)** — trackball, macros, combos, per-OS default
layers, diagnostics and watchdog, all editable in the browser.

A `config/`-only west workspace: devicetree, Kconfig, keymap. Not a ZMK fork.

> ⚠️ Experimental stack. cormoran's fork and modules track floating `main` branches and move fast.
> Freeze SHAs with `west list` after any build you care about reproducing.

---

## Status

All twelve Studio tabs live:

**Keymap** · **Trackball** · **Trackball processors** · **Connection** · **Settings** · **Device Info**
· **Key Switches** · **Macros** · **Combos** · **Default layer** · **OS detection** · **Watchdog**

Right half / central: `FLASH 370,236 B / 792 KB (45.65%)` · `RAM 164,324 B / 256 KB (62.68%)`

---

## Hardware

| | |
|---|---|
| Keyboard | [Charybdis](https://github.com/bdsedo/charybdis) — board design by **weekin** |
| MCUs | 2× nice!nano **v2** (nRF52840) |
| Split | Bluetooth LE |
| Central | **right half** — set by `SHIELD_CHARYBDIS_RIGHT` in [config/boards/shields/charybdis/Kconfig.defconfig](config/boards/shields/charybdis/Kconfig.defconfig) |
| Trackball | PMW3610, **right half** (local to the central) |
| Trackball SPI | `&spi0` — SCK `P0.08`, MOSI/MISO `P0.17`, CS `P0.20`, IRQ `P0.06` |

Sensor on the central means **no split relay for the trackball** —
`CONFIG_ZMK_PMW3610_SPLIT_RPC_RELAY` is deliberately absent.

> MOSI and MISO both on `P0.17` is weekin's original value from his working overlay, and it has
> tracked correctly in daily use. Do not "correct" it.

---

## Layout

```
.
├── build.yaml                            # build matrix: board / shield / snippet / artifact
├── keymap_drawer.config.yaml           # keymap-drawer rendering config
├── img/
├── config/
│   ├── west.yml                          # the manifest
│   ├── charybdis.conf                  # SHARED — both halves
│   ├── charybdis.keymap                # SHARED — both halves
│   ├── charybdis.json                  # keymap metadata
│   └── boards/shields/charybdis/
│       ├── Kconfig.defconfig           # central role for the right half
│       ├── Kconfig.shield
│       ├── charybdis.zmk.yml         # shield metadata
│       ├── charybdis.conf
│       ├── charybdis.dtsi              # SHARED — layout, matrix transform, kscan0, usbd
│       ├── charybdis_left.overlay        # left col-gpios
│       ├── charybdis_left.conf
│       ├── charybdis_right.overlay       # right col-gpios + &spi0 trackball + RIP nodes
│       └── charybdis_right.conf
└── .github/workflows/
```

---

## Build

**CI:** push to `config/**` or `west.yml`, or dispatch manually → three artifacts.

**Local:**
```bash
west init -l config && west update
west build -s zmk/app -b nice_nano@2.0.0/nrf52840/zmk -S studio-rpc-usb-uart \
     -- -DZMK_CONFIG=$PWD/config -DSHIELD=charybdis_right
```

Two parts of that string are load-bearing:

- **`nice_nano@2.0.0/nrf52840/zmk`** — *not* `nice_nano_v2`. The `_v2` suffix was replaced by Zephyr's
  revision mechanism; `2.0.0` is your v2 hardware and is already the board's `default_revision`.
  Wrong form = `Invalid BOARD`, and cmake dies before devicetree is parsed.
- **`-S studio-rpc-usb-uart`** is a **snippet**, not a shield. Chaining it into `shield:` leaves you with
  no USB Studio RPC endpoint, and drops the `-DZMK_BEHAVIORS_KEEP_ALL` that keeps
  `/omit-if-no-ref/` behaviours in the image.

## Flash

**First, every time:** download your working `.uf2`s from Actions. Artifacts auto-delete, and once the
branch heads move you cannot rebuild them.

1. `charybdis-settings-reset` → **right**, wait ~10 s at boot (wipes at boot, no keypress)
2. `charybdis-settings-reset` → **left**, same
3. Each half its own firmware
4. Power cycle, re-pair halves, re-add the keyboard to your computer
5. Studio over **USB on the right half**, then press <kbd>studio_unlock</kbd>

**Never `eraseall`** — it destroys the UF2 bootloader, recoverable only over SWD.

**Prefer USB over BLE for Studio.** `CONFIG_ZMK_BLE_EXPERIMENTAL_CONN=y` turns `BT_CTLR_PHY_2M` off:
deliberate link stability tuning, but it caps the bandwidth Studio config writes want.

**Persistence trap:** everything changed in Studio is saved to flash and reapplied at boot *on top of*
your devicetree. A devicetree fix that appears to do nothing is usually this — clear it with Studio's
**Restore Stock Settings** or reflash `settings_reset`. Studio also caches the subsystem list in the
browser, so hard-refresh after reflashing.

---

## Layers

Indices are **positional** — fixed by declaration order in [config/charybdis.keymap](config/charybdis.keymap).

| # | Layer | Entry | Trackball |
|---|---|---|---|
| 0 | `QWERTY` | base | mouse 1:1 |
| 1 | `F_layers` | `&mo 1` | mouse 1:1 |
| 2 | `BT_layers` | `&lt 2 B` | mouse 1:1 — holds **`&studio_unlock`** |
| 3 | `scroll_gate` | `&mo SCROLL_L` | scroll, XY→wheel 1/60 |
| 4–7 | `extra1`–`extra4` | `status = "reserved"` | mouse — **Studio-owned** |

**Snipe has no layer:** `&rip_ldpi` on the right thumb — hold for ½ speed, release restores.
This mirrors cormoran's dya2, whose `trackball_listener` carries only mouse + scroll.

`scroll_gate` must keep existing — layer bits can only be set by a declared layer, so what you freed is
its *binding content*, not its index. Studio cannot add layers beyond the compiled count, hence
`extra1`–`extra4` as headroom. Keep `&mkp LCLK`/`&mkp RCLK` inside the gate, or those slots fall
through to base and type `M` and `,` while scrolling.

Processor masks in `charybdis_right.overlay` are mutually exclusive: `mouse_rip` `<0xF7>` (all but
bit 3), `scroll_rip` `<0x08>` (bit 3 only). Overlap gives you pointer movement *and* wheel events on
one stream.

---

## Adapting to different hardware

### Trackball on the left half, or left is the central
Everything Studio touches must be on the **central**. Either move the sensor nodes, or you need
`CONFIG_ZMK_PMW3610_SPLIT_RPC_RELAY` — which the driver's own docs state is unvalidated on real
BLE split hardware. Check the half role first: it comes from the shield's `Kconfig.defconfig`,
not from any `.conf` (and `CONFIG_ZMK_SPLIT_BLE_ROLE_CENTRAL` has *no prompt*, so assigning it in a
conf is a hard error).

### Different sensor
Swap `compatible = "cormoran,pmw3610"` and the matching driver project in
[config/west.yml](config/west.yml). Its binding requires `irq-gpios`, `evt-type`, `x-input-code`,
`y-input-code` — omit one and the node never instantiates. Use `INPUT_EV_REL` (`0x02`);
`INPUT_EV_RELATIVE` does not exist. `cpi` is a devicetree property, not a Kconfig symbol.

### Different MCU / board
Recheck the board string entirely — `west boards` prints the authoritative list. Board targets moved to
vendor directories on Zephyr 4.x, and `nice_nano_v2` is simply not a recognised name here. Also confirm
the shield still declares its `col-gpios` for each half; `charybdis.dtsi` deliberately owns only the
shared rows, layout and transform.

### Wired split instead of BLE
Drop the BLE link tuning block from `config/charybdis.conf` (`BT_PERIPHERAL_PREF_*`,
`ZMK_SPLIT_BLE_CENTRAL_BATTERY_LEVEL_*`). Zephyr `BUILD_ASSERT`s
`BT_BUF_EVT_RX_COUNT > BT_BUF_ACL_TX_COUNT` — inherited constraint, easy to trip when porting tuning.

### Other platforms
`config/boards` emits a *deprecated* warning. Benign on this base; migrate to a real module when convenient.
Some Kconfig symbols have no prompt and must be set via devicetree instead — `SPI_NRFX_SPIM` among them,
switched on by keeping `status = "okay"` and `compatible = "nordic,nrf-spim"` on `&spi0`.

### A Studio tab reads "not available" on a green build
Almost always a silent CMake exclusion: cormoran's modules gate their `src/studio/*.c` behind nested
`if(CONFIG_X)` **and** `if(CONFIG_X_STUDIO_RPC)`. Missing the inner symbol means the handler never
compiles, nothing references the absent file, the build is green, and the subsystem never registers.
Grep the log for that module's `*_handler.c`; if absent, the symbol is unset. Note the naming is
inconsistent — `zmk-module-settings-rpc` uses `_STUDIO`, every other module `_STUDIO_RPC`.
`CONFIG_ZMK_STUDIO_RPC_CUSTOM_SUBSYSTEM_PRINT_LIST_ON_START=y` prints what actually registered.

**Don't chase these — both ruled out:** TX/RX buffer sizes (subsystem enumeration streams via
`pb_ostream_t`; there is no count cap), and removing `remote:` from `west.yml` (`defaults:` yields a
byte-identical clone URL).

### Other rules that cost the most time
- **The peripheral does not run a keymap.** The central runs it for both halves. So
  `RUNTIME_MACRO`/`RUNTIME_COMBO` are **central-only** — enabling them on the left is an
  `undefined reference to zmk_keymap_highest_layer_active` link failure. kscan-level features
  (`KSCAN_DIAGNOSTICS`) and watchdog belong on **both**. A macro bound to a left-hand key still works.
- **`CONFIG_ZMK_STUDIO=y` stays off the shared conf.** The left has no `studio-rpc-usb-uart` snippet, so
  `proto/zmk/custom.pb.h` is never generated there, and `ZMK_KSCAN_DIAGNOSTICS_STUDIO_RPC` is `default y`
  — Studio in the shared conf auto-enables it on the peripheral and breaks its compile.
- **No inline comments on `CONFIG_X=<value>` lines.** Kconfig reads everything after `=` as the value and
  Zephyr promotes the warning to fatal.
- **Behaviour overrides (`&sl`, `&lt`) belong in the keymap**, never a shield overlay — `behaviors.dtsi`
  is included by the keymap, which dtc processes *after* the overlay.

### Watchdog
`ZMK_WATCHDOG_FREEZE_DETECT=n` by choice. The module documents a low-probability conflict on
split-central builds where its periodic `k_timer` collides with the nRF controller's ~275 µs radio-event
prep window, tripping `LL_ASSERT_OVERHEAD` and rebooting. Incident logging and hard-fault capture stay on
at zero added timer load. Arm freeze detection last. A blank peripheral panel while the left half sleeps
is expected — relayed requests never answer a disconnected peripheral.

---

## Data collection

DYA Studio's connect dialog discloses collection of your **keyboard name** plus anonymous usage telemetry
(features used, connection method, coarse errors) to Google Analytics 4. Keymaps, layers, macros and
trackball values never leave the device — they're edited over local RPC. The keyboard name *is* read from
firmware (`CONFIG_ZMK_KEYBOARD_NAME="Charybdis"`) and sent. The dialog is a notice, not a consent gate:
GA loads at page load. Blocking `googletagmanager.com` disables it with no loss of function.

---

## Credits

- **[cormoran](https://github.com/cormoran)** — the ZMK fork, the custom Studio RPC protocol, and the
  entire module stack this config stands on.
- **weekin** — Charybdis board design and the original shield/devicetree.
- **[ZMK Firmware](https://zmk.dev)** — upstream.
- **[DYA Studio](https://studio.dya.cormoran.works/)** — the web client.

## License

Upstream ZMK is MIT; cormoran's modules carry their own licences — check each repo.
**TODO: declare a licence for this config.**
