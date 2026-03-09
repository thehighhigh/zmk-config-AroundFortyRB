# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a ZMK firmware configuration repository for the **AroundForty RB** — a wireless split keyboard with a trackball (PMW3610 sensor) on the right half. The keyboard uses Seeeduino XIAO BLE (nRF52840) as the controller for both halves.

## Build

Firmware is built via **GitHub Actions** — push to `main` triggers the workflow automatically. There is no local build toolchain configured in this repo; builds are entirely CI-driven.

The workflow uses ZMK's reusable workflow at `zmkfirmware/zmk/.github/workflows/build-user-config.yml@v0.3.0`.

Build targets defined in [build.yaml](build.yaml):
- `AroundForty-RB_R` + `rgbled_adapter` + `studio-rpc-usb-uart` snippet (right half, central)
- `AroundForty-RB_L` + `rgbled_adapter` (left half, peripheral)
- `settings_reset` (for pairing resets)

To trigger a build: commit and push, or use the "Run workflow" button in GitHub Actions.

## Repository Structure

```
config/
  AroundForty-RB.keymap        # Keymap (all layers defined here)
  AroundForty-RB.json          # Physical layout for ZMK Studio
  west.yml                     # ZMK + module dependencies (west manifest)
  boards/shields/AroundForty-RB/
    AroundForty-RB.dtsi        # Shared hardware: physical layout, matrix transform, kscan
    AroundForty-RB_R.overlay   # Right half: col GPIOs + PMW3610 SPI trackball config
    AroundForty-RB_L.overlay   # Left half: col GPIOs only
    AroundForty-RB_R.conf      # Right (central): BLE, PMW3610, ZMK Studio, RGB LED configs
    AroundForty-RB_L.conf      # Left (peripheral): BLE, RGB LED configs
    Kconfig.defconfig          # Sets keyboard name, enables ZMK_SPLIT + CENTRAL role
    Kconfig.shield             # Shield select definitions
    AroundForty-RB.zmk.yml     # Shield metadata
```

## Architecture

### Split Keyboard Design
- **Right half = Central**: hosts the PMW3610 trackball, connects to host OS via BLE, runs ZMK Studio
- **Left half = Peripheral**: connects to right via BLE, no trackball
- Matrix: 4 rows × 11 columns (col-offset=6 on right half to share the same transform)

### External Modules (west.yml)
- `zmk` @ `v0.3.0` — ZMK firmware base
- `zmk-pmw3610-driver` (badjeff) @ `zmk-0.3` — PMW3610 trackball driver
- `zmk-rgbled-widget` (caksoylar) @ `v0.3-branch` — RGB LED status widget

### Keymap Layers (AroundForty-RB.keymap)
| Index | Name | Description |
|-------|------|-------------|
| 0 | Mac-Base | Default QWERTY layer for macOS |
| 1 | Mac-Fnc | Function/symbol layer |
| 2 | Mac-Common | Navigation, mouse clicks, window control |
| 3 | Num_Scroll | Number pad + F-keys |
| 4 | Settings | BT profiles, reset, bootloader, ZMK Studio unlock |
| 5 | AML | Auto mouse layer (mouse buttons) |
| 6 | layer_10 | Scroll + mouse buttons |
| 7 | V_Scroll | Vertical scroll only |

Layer 7 (V_Scroll) is the scroll layer in the trackball input listener (`_R.overlay`). Layer 6 (layer_10) uses the trackball for cursor movement while providing mouse buttons and scroll keys on the keymap.

### Trackball Input Processing (_R.overlay)
The trackball listener applies:
1. X-axis invert transform
2. 1:1 XY scaling
3. On layer 7 only: converts XY motion to scroll with 1:45 scaler

To enable scroll inversion, uncomment `scroll-y-invert` / `scroll-x-invert` in `zip_xy_to_scroll_mapper_tune` and switch the mapper reference.

To enable AML (Auto Mouse Layer), uncomment `<&zip_temp_layer 5 300>` in `trackball_listener`.

### Key Macros
- `ime_tog`: Windows IME toggle (Alt + Grave)
- `mac_ime`: macOS IME toggle (Ctrl + Space)
- `to_layer_0`: Parameterized macro — sends a keycode then switches to layer 0
- `lt_to_layer_0`: Hold-tap using `to_layer_0` as tap action

## Important Configuration Notes

- Right half conf enables `CONFIG_ZMK_STUDIO=y` with `CONFIG_ZMK_STUDIO_LOCKING=n`
- NFC pins are repurposed as GPIO on both halves (`CONFIG_NFCT_PINS_AS_GPIOS=y`)
- BLE 2M PHY is disabled on both halves for stability (`CONFIG_BT_CTLR_PHY_2M=n`)
- Idle sleep timeout: 30 minutes (`CONFIG_ZMK_IDLE_SLEEP_TIMEOUT=1800000`)
- PMW3610 CPI: 400 (configured in `_R.overlay`)
