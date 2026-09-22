# ogx-mini-stadia

## What this is
Fork of [wiredopposite/OGX-Mini](https://github.com/wiredopposite/OGX-Mini) (branch `stadia-extras`) that turns a Raspberry Pi Pico 2 W into a Bluetooth-to-USB bridge for a Google Stadia controller on a Windows 11 PC. Stock OGX-Mini already gives XInput plus rumble; this fork adds the two Stadia-only buttons as keyboard keys (Capture → F14, Assistant → F15) by making XInput mode a composite USB device.

Lives inside the `laski-pc` ops workspace at `~/projects/laski-pc/src/ogx-mini-stadia` but is its own git repo. Upstream layout is kept as-is (`Firmware/`, `WebApp/`, `hardware/`, `Tools/`). Fork-specific files: `docs/` (design spec), `plans/` (implementation plan), `dist/` (built uf2s, gitignored), this file.

## Toolchain
- Host: this Linux box. `cmake`, `ninja-build`, `gcc-arm-none-eabi`, `libnewlib-arm-none-eabi`, `libstdc++-arm-none-eabi-newlib` via apt.
- pico-sdk 2.1.0 at `Firmware/external/pico-sdk` (gitignored, shallow clone). Matches upstream CI.
- Only these submodules are needed: `Firmware/external/{bluepad32,tinyusb,Pico-PIO-USB,libfixmath}` (bluepad32 recursive for btstack). `Firmware/ESP32_Blueretro`, `WebApp`, `Tools/dump-dvd-kit` are not checked out.
- Configure once, then build:
  ```
  cmake -S Firmware/RP2040 -B build -G Ninja -DCMAKE_BUILD_TYPE=Release \
        -DPICO_SDK_PATH=$PWD/Firmware/external/pico-sdk -DOGXM_BOARD=PI_PICO2W -DMAX_GAMEPADS=1
  cmake --build build
  ```
  Output `build/OGX-Mini-v1.0.0a3-PI_PICO2W.uf2`. Full build is about 470 Ninja steps.
- Flash: hold BOOTSEL on the Pico while plugging in, copy the uf2 to the RPI-RP2 drive.
- No unit tests exist. Verification is compile-time `static_assert`s on descriptor sizes plus on-device checks on the Windows PC (`laski-pc`).
- Descriptor sanity check without hardware: `arm-none-eabi-objdump -s build/*.elf` and search for `09 02 49 00 02` (config), `12 01 00 02 00 00 00 40 09 12 01 00` (device), `XUSB10`, `MSFT100`.

## Architecture overview
Data path: Stadia controller (BLE) → Bluepad32 Android HID parser → `Bluepad32.cpp` bridge → `Gamepad::PadIn` → `XInputDevice::process` → USB.

Fork changes, all additive:
- `Firmware/external/patches/bluepad32_stadia_buttons.diff`, applied by `Firmware/cmake/patch_libs.cmake`: Stadia button 17 (usage 0x11) sets `MISC_BUTTON_CAPTURE`, button 18 (0x12) sets new `MISC_BUTTON_ASSISTANT = BIT(4)`.
- `Gamepad.h`: `BUTTON_CAPTURE = 0x1000`, `BUTTON_ASSISTANT = 0x2000`, fixed bits that bypass per-profile remapping. `Bluepad32.cpp` copies them in.
- `Descriptors/XInput.h`: device class 0x00 (composite), VID/PID `1209:0001`, two interfaces (0 = original XInput vendor interface, byte-identical; 1 = HID boot keyboard, EP 0x82), Microsoft OS 1.0 string (index 0xEE, vendor code 0x20) and Extended Compat ID (`XUSB10` on interface 0).
- `XInputDevice`: serves those descriptors, answers the vendor request for the compat ID, builds an 8-byte keyboard report from the two bits and sends it via `tud_hid_n_report(0, ...)` only when it changes. `echo_set_report()` returns false so host LED reports aren't echoed as key presses.
- `DeviceDriver::echo_set_report()` (default true) and `tud_callbacks.cpp` gate the upstream SET_REPORT echo on it.

## Boundaries
- Always keep interface 0 of the XInput config descriptor byte-identical to upstream and first.
- Never change other output modes' behavior; the fork must stay mergeable with upstream.
- Never commit `build/`, `dist/`, or `Firmware/external/pico-sdk`.
- Ask first before pushing to GitHub (`origin` = peanutlasko/OGX-Mini, default branch `stadia-extras`) and before opening anything against upstream.
- Keep `master` as an untouched mirror of `upstream/master`; pull upstream changes there and rebase or merge `stadia-extras` on top. Never merge the fork work into `master`.

## Current status
Working. Verified on laski-pc on 2026-09-22: Windows enumerates a USB Composite Device with "Xbox 360 Controller for Windows" and "HID Keyboard Device", gamepad inputs and rumble work, Capture sends F14 and Assistant sends F15. Nothing pending. Possible follow-ups: web-app mapping of the new keys, upstream PR.

## Recent changes
- 2026-09-22: Composite XInput + HID keyboard, MS OS descriptors, Bluepad32 patch, F14/F15 mapping. Spec in `docs/`, plan in `plans/`. Verified on hardware the same day.
- 2026-09-22: Forked upstream at `ccccf66`, set up local toolchain, reproduced stock Pico 2 W build.

## Lessons learned
- Bluepad32 parses the Stadia controller with the Android parser, not the generic one, and it has empty `case 0x11`/`0x12` bodies for Capture/Assistant. The Stadia parser file only handles rumble.
- `xusb22.inf` matches `USB\VID_045E&PID_028E` at device level. Keeping Microsoft's VID/PID on a composite device would let the Xbox driver claim the whole device and starve the keyboard interface, so the fork uses a pid.codes VID/PID and relies on the `XUSB10` compatible ID instead.
- Windows caches the result of the MS OS descriptor query per VID/PID/bcdDevice under `HKLM\SYSTEM\CurrentControlSet\Control\UsbFlags\120900010114`. If the composite ever enumerates wrong, delete that key and replug before debugging the firmware.
- Upstream `tud_hid_set_report_cb` echoes every host SET_REPORT back as an input report. For a keyboard interface the 1-byte LED report would be read as a modifier byte, so the echo is now gated per driver.
- TinyUSB routes every vendor-type control request (any recipient) to `tud_vendor_control_xfer_cb`, so the MS OS compat ID handler lives there, not in the XInput class driver's `control_xfer_cb`.
- `patch_libs.cmake` detects an already-applied patch by matching "patch does not apply" in `git apply`'s stderr. A patch file generated with `git diff` from inside the submodule applies cleanly with `--ignore-whitespace`.
- Appending to upstream `.gitignore` needs a newline check first; the file has no trailing newline.
- Build warnings "OGXM_BOARD redefined" and "_getentropy is not implemented" are present in the stock build too.
