# Stadia Assistant and Capture buttons over XInput — design

Date: 2026-09-22
Branch: `stadia-extras` (fork of wiredopposite/OGX-Mini)
Target board: Raspberry Pi Pico 2 W (`OGXM_BOARD=PI_PICO2W`, `MAX_GAMEPADS=1`)

## Goal

Use a Pico 2 W running OGX-Mini as a Bluetooth-to-USB bridge for a Google
Stadia controller (Bluetooth firmware) on a Windows 11 PC, and make the two
Stadia-only buttons usable:

| Stadia button | Bluetooth HID usage | USB output |
|---|---|---|
| Capture   | Button page, usage 0x11 (button 17) | Keyboard key F14 |
| Assistant | Button page, usage 0x12 (button 18) | Keyboard key F15 |

Stock OGX-Mini already gives XInput plus rumble over Bluetooth on this board.
That must keep working unchanged.

## Why the buttons are lost today

1. Bluepad32 routes the Stadia controller through its Android HID parser
   (`uni_hid_parser_android.c`). That parser has explicit `case 0x11` and
   `case 0x12` entries for the Stadia Capture and Assistant buttons whose
   bodies are empty, so the presses never reach `misc_buttons`.
2. OGX-Mini's XInput output is a single vendor-class interface whose report
   has no spare button bits.

## Design

### 1. Bluetooth side (Bluepad32 patch)

Add `Firmware/external/patches/bluepad32_stadia_buttons.diff` and apply it in
`Firmware/cmake/patch_libs.cmake` next to the existing `bluepad32_uni.diff`.

The patch edits `uni_hid_parser_android.c`:

- `case 0x11` sets `MISC_BUTTON_CAPTURE` (already defined by Bluepad32).
- `case 0x12` sets a new `MISC_BUTTON_ASSISTANT` bit, added to
  `uni_gamepad.h` as an extra bit in `misc_buttons` without touching the
  `UNI_GAMEPAD_MAPPINGS_*` enum or mapping tables.

### 2. Gamepad model (OGX-Mini)

`Gamepad.h` gains two fixed button bits in the existing 16-bit field:

```
BUTTON_CAPTURE   = 0x1000
BUTTON_ASSISTANT = 0x2000
```

`Bluepad32.cpp` copies `MISC_BUTTON_CAPTURE` and `MISC_BUTTON_ASSISTANT`
into those bits. They deliberately bypass the per-profile `MAP_BUTTON_*`
remapping, so the user-profile format and web app are untouched. All other
output drivers ignore the bits.

### 3. USB side (XInput mode only)

`XInputDevice` becomes a two-interface composite device:

- Interface 0: the existing XInput vendor interface, byte-for-byte unchanged
  (class 0xFF / 0x5D / 0x01, EP 0x81 IN, EP 0x01 OUT).
- Interface 1: a HID boot keyboard using TinyUSB's built-in HID class
  (`CFG_TUD_HID` is already 1). Report descriptor is
  `TUD_HID_REPORT_DESC_KEYBOARD()`. One interrupt IN endpoint (0x82), 8-byte
  report, 10 ms interval.

Device descriptor changes `bDeviceClass/SubClass/Protocol` from 0xFF to 0x00
so Windows loads the composite parent driver and splits the interfaces.

Microsoft OS 1.0 descriptors make Windows bind `xusb22.sys` to interface 0:

- String descriptor index 0xEE returns `MSFT100` followed by a vendor code.
- A control request with `bRequest == vendor code` and `wIndex == 0x0004`
  returns an Extended Compat ID descriptor with one section:
  `FirstInterfaceNumber = 0`, `CompatibleID = "XUSB10"`.
  The request may arrive with recipient Device or Interface, so both the
  `tud_vendor_control_xfer_cb` path and the XInput class driver's
  `control_xfer_cb` handle it.

`XInputDevice::get_descriptor_string_cb` currently indexes `DESC_STRING[]`
without a bounds check. It gains one and handles 0xEE.

In `XInputDevice::process`, after the gamepad report is sent, the driver
builds a standard 8-byte keyboard report (modifier byte, reserved byte, six
keycodes) containing `HID_KEY_F14` when `BUTTON_CAPTURE` is set and
`HID_KEY_F15` when `BUTTON_ASSISTANT` is set. It sends the report through
`tud_hid_n_report(0, ...)` only when it differs from the last one sent.

`tud_hid_get_report_cb` for HID instance 0 returns the current keyboard
report. `tud_hid_descriptor_report_cb` returns the keyboard report descriptor.

### 4. Everything else

Mode-switch button combos, pairing, the web app, rumble, and every non-XInput
output mode are unchanged.

## Build

- Toolchain: `cmake`, `ninja-build`, `gcc-arm-none-eabi`,
  `libnewlib-arm-none-eabi`, `libstdc++-arm-none-eabi-newlib` (apt).
- pico-sdk 2.1.0 cloned to `Firmware/external/pico-sdk` (matches upstream CI).
- Build:
  ```
  cmake -S Firmware/RP2040 -B build -G Ninja -DCMAKE_BUILD_TYPE=Release \
        -DPICO_SDK_PATH=$PWD/Firmware/external/pico-sdk \
        -DOGXM_BOARD=PI_PICO2W -DMAX_GAMEPADS=1
  cmake --build build
  ```
- Output: `build/OGX-Mini-*-PI_PICO2W.uf2`. Flash by holding BOOTSEL while
  plugging in and copying the file to the RPI-RP2 drive.

## Verification checkpoints

1. **Stock build.** Unmodified upstream builds and, when flashed, behaves the
   same as the release: inputs and rumble work in XInput mode.
2. **Composite enumeration.** With the USB change, Windows Device Manager
   shows "Xbox 360 Controller for Windows" and "HID Keyboard Device" under one
   composite device. Gamepad inputs and rumble still work.
3. **End to end.** Capture produces F14 and Assistant produces F15 in a key
   tester. Mode-switch combos still work.

## Fallback

If Windows will not bind the Xbox driver to interface 0 of a composite
device, switch the keyboard interface to DInput mode instead (both interfaces
plain HID, no Microsoft OS descriptors) and rely on Steam Input for XInput.

## Out of scope

Web app mapping of the new keys, rumble settings, other output modes,
upstreaming.
