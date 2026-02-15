Here is a clean, ready-to-commit Markdown file you can add to your repo (e.g. `MIGRATION_TO_ZMK_v0.3.md`).

---

# ErgoTravelLight – Migration to ZMK v0.3

This document summarizes the full migration process from an old ZMK setup (v0.1 era) to modern ZMK v0.3, including structural changes, build fixes, and Bluetooth troubleshooting.

---

## 1. Pin ZMK Version

Do **not** build against `main`.

### `config/west.yml`

```yaml
manifest:
  remotes:
    - name: zmkfirmware
      url-base: https://github.com/zmkfirmware
  projects:
    - name: zmk
      remote: zmkfirmware
      revision: v0.3
      import: app/west.yml
  self:
    path: config
```

### `.github/workflows/build.yml`

Pin the reusable workflow:

```yaml
uses: zmkfirmware/zmk/.github/workflows/build-user-config.yml@v0.3
```

---

## 2. Convert to Modern Module Layout

Old layout (deprecated):

```
config/boards/...
```

New required layout:

```
boards/shields/<shield_name>/
```

### Final Repository Structure

```
zmk-config/
├── boards/
│   └── shields/
│       ├── ergotravellight_left/
│       │   ├── Kconfig.shield
│       │   ├── Kconfig.defconfig
│       │   ├── ergotravellight_left.overlay
│       │   ├── ergotravellight_left.conf
│       │   └── ergotravellight_left.zmk.yml
│       ├── ergotravellight_right/
│       │   ├── Kconfig.shield
│       │   ├── Kconfig.defconfig
│       │   ├── ergotravellight_right.overlay
│       │   ├── ergotravellight_right.conf
│       │   └── ergotravellight_right.zmk.yml
│       └── ergotravellight.dtsi
│
├── config/
│   ├── ergotravellight.keymap
│   └── west.yml
│
├── zephyr/
│   └── module.yml
│
├── Kconfig
├── CMakeLists.txt
└── build.yaml
```

---

## 3. Required Module Files

### `zephyr/module.yml`

```yaml
name: zmk-config
build:
  cmake: .
  kconfig: Kconfig
  settings:
    board_root: .
```

`board_root` is required for shield discovery.

---

### Root `CMakeLists.txt`

```cmake
cmake_minimum_required(VERSION 3.20.0)
```

---

### Root `Kconfig`

```kconfig
menu "ergotravellight module"
endmenu
```

---

## 4. Split Shield Configuration

Each half must define only its own shield.

### `ergotravellight_left/Kconfig.shield`

```kconfig
config SHIELD_ERGO_TRAVEL_LIGHT_LEFT
    def_bool $(shields_list_contains,ergotravellight_left)
```

### `ergotravellight_right/Kconfig.shield`

```kconfig
config SHIELD_ERGO_TRAVEL_LIGHT_RIGHT
    def_bool $(shields_list_contains,ergotravellight_right)
```

---

### Left `Kconfig.defconfig`

```kconfig
if SHIELD_ERGO_TRAVEL_LIGHT_LEFT

config ZMK_KEYBOARD_NAME
    default "ErgoTravel"

config ZMK_SPLIT
    default y

config ZMK_SPLIT_BLE_ROLE_CENTRAL
    default y

endif
```

### Right `Kconfig.defconfig`

```kconfig
if SHIELD_ERGO_TRAVEL_LIGHT_RIGHT

config ZMK_KEYBOARD_NAME
    default "ErgoTravel"

config ZMK_SPLIT
    default y

endif
```

---

## 5. Removed Deprecated Config

Old:

```conf
CONFIG_ZMK_BKE=y
```

This option no longer exists in modern ZMK and must be removed.

---

## 6. Bluetooth Device Name Fix

Zephyr asserts:

```
BUILD_ASSERT(DEVICE_NAME_LEN < CONFIG_BT_DEVICE_NAME_MAX);
```

If your device name is long, add:

```conf
CONFIG_BT_DEVICE_NAME="ErgoTravel"
CONFIG_BT_DEVICE_NAME_MAX=32
```

---

## 7. Devicetree Includes

If using Bluetooth behaviors in keymap:

```dts
#include <dt-bindings/zmk/bt.h>
```

Without this include, `BT_SEL`, `BT_CLR`, etc. will fail to parse.

---

## 8. Bluetooth Pairing Issues

### Symptom

* Device appears in macOS
* Clicking "Connect" briefly shows “Connecting…”
* Then reverts to “Connect”

### Cause

Stale Bluetooth bonds stored in keyboard flash.

### Fix

Add to keymap:

```dts
&bt BT_CLR_ALL
&bt BT_SEL 0
```

Then:

1. Flash firmware
2. Power central (left) half
3. Press `BT_CLR_ALL`
4. Wait 5 seconds
5. Press `BT_SEL 0`
6. Pair from macOS

This resolved pairing issues completely.

---

## 9. macOS UF2 Copy Error (-36)

Finder may fail copying UF2 with:

```
Error code -36
```

Use Terminal instead:

```bash
cp firmware.uf2 /Volumes/NICENANO/
```

If the drive disappears automatically, flashing succeeded.

---

## 10. Final Working State

* ZMK v0.3 pinned
* Modern module layout
* Shields properly discovered
* Split central/peripheral correct
* Bluetooth bonds reset
* macOS pairing stable

---

## Key Lessons

1. Always pin ZMK release versions.
2. Use module layout (`boards/shields/`).
3. Add `board_root` to `module.yml`.
4. Remove deprecated config options.
5. Bluetooth bond storage survives firmware flashes.
6. `BT_CLR_ALL` is essential when debugging pairing.

---

End of document.
