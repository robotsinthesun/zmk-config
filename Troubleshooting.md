# ZMK Config Notes & Best Practices

This document summarizes important structural requirements, configuration patterns, and common troubleshooting steps when creating and maintaining a custom ZMK configuration repository.

---

# 1. Always Pin ZMK Versions

Do **not** build against `main`.

Pin both:

* The ZMK revision in `west.yml`
* The reusable GitHub workflow version

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

```yaml
uses: zmkfirmware/zmk/.github/workflows/build-user-config.yml@v0.3
```

Pinning prevents breakage from upstream changes.

---

# 2. Use Modern Module Layout

ZMK configs must use the Zephyr module layout.

## Required Structure

```
zmk-config/
├── boards/
│   └── shields/
│       └── <shield_name>/
│           ├── Kconfig.shield
│           ├── Kconfig.defconfig
│           ├── <shield_name>.overlay
│           ├── <shield_name>.conf
│           └── <shield_name>.zmk.yml
│
├── config/
│   ├── <keyboard>.keymap
│   └── west.yml
│
├── zephyr/
│   └── module.yml
│
├── Kconfig
├── CMakeLists.txt
└── build.yaml
```

Do **not** place shields under `config/boards/` — that layout is deprecated.

---

# 3. Required Module Files

## `zephyr/module.yml`

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

## Root `CMakeLists.txt`

```cmake
cmake_minimum_required(VERSION 3.20.0)
```

Even if empty, this file is required when using `cmake: .` in `module.yml`.

---

## Root `Kconfig`

```kconfig
menu "zmk-config module"
endmenu
```

A module-level `Kconfig` is required when declared in `module.yml`.

---

# 4. Split Keyboard Configuration

Each half must define only its own shield.

Example:

## `ergotravellight_left/Kconfig.shield`

```kconfig
config SHIELD_ERGOTRAVELLIGHT_LEFT
    def_bool $(shields_list_contains,ergotravellight_left)
```

## `ergotravellight_right/Kconfig.shield`

```kconfig
config SHIELD_ERGOTRAVELLIGHT_RIGHT
    def_bool $(shields_list_contains,ergotravellight_right)
```

### Central (host-facing half)

```kconfig
config ZMK_SPLIT
    default y

config ZMK_SPLIT_BLE_ROLE_CENTRAL
    default y
```

### Peripheral half

```kconfig
config ZMK_SPLIT
    default y
```

Only one half should define `ZMK_SPLIT_BLE_ROLE_CENTRAL`.

---

# 5. Device Tree Includes

When using Bluetooth behaviors in keymaps, include:

```dts
#include <dt-bindings/zmk/bt.h>
```

Other common includes:

```dts
#include <dt-bindings/zmk/keys.h>
#include <dt-bindings/zmk/behaviors.h>
```

Without the correct includes, devicetree parsing will fail.

---

# 6. Bluetooth Best Practices

## Add Maintenance Keys

Keep these available in a utility layer:

```dts
&bt BT_CLR_ALL
&bt BT_SEL 0
&bt BT_SEL 1
&bt BT_NXT
&bt BT_PRV
```

## Clearing Bond Issues

If pairing loops (“Connecting…” → “Connect”):

1. Press `BT_CLR_ALL`
2. Wait 5 seconds
3. Press `BT_SEL 0`
4. Pair again

ZMK stores bonds in flash across firmware updates.

---

# 7. Bluetooth Device Name Length

Zephyr enforces:

```
DEVICE_NAME_LEN < CONFIG_BT_DEVICE_NAME_MAX
```

If using longer names, explicitly set:

```conf
CONFIG_BT_DEVICE_NAME="YourKeyboard"
CONFIG_BT_DEVICE_NAME_MAX=32
```

Otherwise builds may fail with static assertions.

---

# 8. Deprecated Config Options

Old options may no longer exist.

Example (removed):

```
CONFIG_ZMK_BKE
```

Kconfig warnings are treated as fatal errors in modern ZMK.

Always remove undefined symbols.

---

# 9. Shared DTS Files

Common hardware definitions can live in a shared `.dtsi` file:

```
boards/shields/<common>.dtsi
```

Include from overlays:

```dts
#include "../common.dtsi"
```

File naming is flexible — inclusion is what matters.

---

# 10. macOS UF2 Flashing

If Finder fails copying UF2 files with error `-36`, use Terminal:

```bash
cp firmware.uf2 /Volumes/NICENANO/
```

If the drive disappears automatically, flashing succeeded.

---

# 11. Debugging Order for Bluetooth Issues

1. Confirm correct firmware flashed to each half.
2. Power only the central half and test pairing.
3. Clear bonds with `BT_CLR_ALL`.
4. Ensure only central half pairs to host.
5. Verify split halves connect to each other.

---

# 12. General Recommendations

* Use lowercase shield names consistently.
* Match folder name, shield name, and build.yaml entries exactly.
* Keep Bluetooth utilities in keymap permanently.
* Pin ZMK releases.
* Avoid building against `main`.
