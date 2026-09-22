# Stadia Extra Buttons Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make the Stadia controller's Capture and Assistant buttons arrive on Windows as keyboard keys F14 and F15 while the Pico 2 W keeps presenting an XInput controller with rumble.

**Architecture:** Patch Bluepad32's Android HID parser so the two buttons reach `misc_buttons`; carry them through OGX-Mini's `Gamepad` struct as two new fixed bits; turn the XInput USB device into a two-interface composite (XInput vendor interface + HID boot keyboard) with Microsoft OS descriptors so Windows binds `xusb22.sys` to interface 0.

**Tech Stack:** C++17, pico-sdk 2.1.0, TinyUSB (pinned submodule), Bluepad32 4.1.0 (pinned submodule), CMake + Ninja, `gcc-arm-none-eabi`.

**Spec:** `docs/2026-09-22-stadia-extra-buttons-design.md`

## Global Constraints

- Board: `OGXM_BOARD=PI_PICO2W`, `MAX_GAMEPADS=1`.
- Only XInput output mode changes behavior. All other modes must build and behave as before.
- Capture → `HID_KEY_F14` (0x69). Assistant → `HID_KEY_F15` (0x6A). No modifiers.
- The XInput interface stays interface 0 and its descriptor bytes stay unchanged.
- No unit-test framework exists in this firmware. Each task's test is: the build passes, compile-time `static_assert`s hold, and the on-device check listed in the task passes. On-device checks need the user to flash the uf2 and report back.
- Build command (from repo root):
  ```
  cmake --build build
  ```
  Output: `build/OGX-Mini-v1.0.0a3-PI_PICO2W.uf2`.
- Commit after each task on branch `stadia-extras`.

---

### Task 1: Prove the stock build on hardware (checkpoint 1)

**Files:** none changed.

**Interfaces:**
- Produces: a known-good baseline uf2 and a confirmed build workflow.

- [x] **Step 1: Build unmodified upstream**

Run: `cmake --build build`
Expected: exit 0, `build/OGX-Mini-v1.0.0a3-PI_PICO2W.uf2` exists. (Done: 2026-09-22, 1,158,144 bytes.)

- [x] **Step 2: Send the uf2 to the user and have them flash it**

Hold BOOTSEL while plugging in, copy the file to the RPI-RP2 drive, re-pair the controller if needed, switch to XInput with Menu + D-pad Up.

- [x] **Step 3: User confirms**

Confirmed 2026-09-22 with the final build. Expected: inputs and rumble identical to the release build. If not, stop: the toolchain differs from upstream CI and must be fixed before any code change.

---

### Task 2: Bluepad32 patch so Capture and Assistant reach `misc_buttons`

**Files:**
- Create: `Firmware/external/patches/bluepad32_stadia_buttons.diff`
- Modify: `Firmware/cmake/patch_libs.cmake` (add a third apply block)

**Interfaces:**
- Produces: `MISC_BUTTON_CAPTURE` set for Stadia button 17, and a new `MISC_BUTTON_ASSISTANT` (= `BIT(4)`, value 0x10) set for Stadia button 18, both in `uni_gamepad_t.misc_buttons`.

- [x] **Step 1: Edit the submodule working tree**

In `Firmware/external/bluepad32/src/components/bluepad32/include/controller/uni_gamepad.h`, inside the `enum` that defines `MISC_BUTTON_SYSTEM` etc., add after the `MISC_BUTTON_CAPTURE` line:

```c
    MISC_BUTTON_ASSISTANT = BIT(4),  // Stadia: Google Assistant button (not part of the mappings table)
```

In `Firmware/external/bluepad32/src/components/bluepad32/parser/uni_hid_parser_android.c`, replace:

```c
                case 0x11:
                    // Stadia controller: Capture button
                    break;
                case 0x12:
                    // Stadia controller: Google Assistant button
                    break;
```

with:

```c
                case 0x11:
                    // Stadia controller: Capture button
                    if (value)
                        ctl->gamepad.misc_buttons |= MISC_BUTTON_CAPTURE;
                    break;
                case 0x12:
                    // Stadia controller: Google Assistant button
                    if (value)
                        ctl->gamepad.misc_buttons |= MISC_BUTTON_ASSISTANT;
                    break;
```

- [x] **Step 2: Generate the patch file and revert the submodule**

```bash
cd Firmware/external/bluepad32
git diff -- src/components/bluepad32/include/controller/uni_gamepad.h \
            src/components/bluepad32/parser/uni_hid_parser_android.c \
  > ../patches/bluepad32_stadia_buttons.diff
git checkout -- src/components/bluepad32/include/controller/uni_gamepad.h \
                src/components/bluepad32/parser/uni_hid_parser_android.c
cd ../../..
```

- [x] **Step 3: Apply it from CMake**

Append to `Firmware/cmake/patch_libs.cmake` inside `apply_lib_patches`, after the Bluepad32 block:

```cmake
    set(BLUEPAD32_STADIA_PATCH "${EXTERNAL_DIR}/patches/bluepad32_stadia_buttons.diff")

    message(STATUS "Applying Bluepad32 Stadia buttons patch: ${BLUEPAD32_STADIA_PATCH}")

    execute_process(
        COMMAND git apply --ignore-whitespace ${BLUEPAD32_STADIA_PATCH}
        WORKING_DIRECTORY ${BLUEPAD32_PATH}
        RESULT_VARIABLE BLUEPAD32_STADIA_PATCH_RESULT
        OUTPUT_VARIABLE BLUEPAD32_STADIA_PATCH_OUTPUT
        ERROR_VARIABLE BLUEPAD32_STADIA_PATCH_ERROR
    )

    if (BLUEPAD32_STADIA_PATCH_RESULT EQUAL 0)
        message(STATUS "Bluepad32 Stadia buttons patch applied successfully.")
    elseif (BLUEPAD32_STADIA_PATCH_ERROR MATCHES "patch does not apply")
        message(STATUS "Bluepad32 Stadia buttons patch already applied.")
    else ()
        message(FATAL_ERROR "Failed to apply Bluepad32 Stadia buttons patch: ${BLUEPAD32_STADIA_PATCH_ERROR}")
    endif ()
```

- [x] **Step 4: Verify the patch applies and builds**

Run: `cmake -S Firmware/RP2040 -B build 2>&1 | grep -i stadia` then `cmake --build build`
Expected: "Bluepad32 Stadia buttons patch applied successfully." and exit 0. Run configure a second time: expect "already applied".

- [x] **Step 5: Commit**

```bash
git add Firmware/external/patches/bluepad32_stadia_buttons.diff Firmware/cmake/patch_libs.cmake
git commit -m "Add Bluepad32 patch exposing Stadia Capture and Assistant buttons"
```

---

### Task 3: Carry the buttons through the OGX-Mini gamepad model

**Files:**
- Modify: `Firmware/RP2040/src/Gamepad/Gamepad.h` (button constants, near line 47)
- Modify: `Firmware/RP2040/src/Bluepad32/Bluepad32.cpp` (button mapping, near line 246)

**Interfaces:**
- Produces: `Gamepad::BUTTON_CAPTURE = 0x1000` and `Gamepad::BUTTON_ASSISTANT = 0x2000` in `Gamepad::PadIn::buttons`. Not remappable through `MAP_BUTTON_*`.

- [x] **Step 1: Add the constants**

In `Gamepad.h` after `static constexpr uint16_t BUTTON_MISC  = 0x0800;` add:

```cpp
    // Fixed, non-remappable extras (Stadia Capture / Assistant)
    static constexpr uint16_t BUTTON_CAPTURE   = 0x1000;
    static constexpr uint16_t BUTTON_ASSISTANT = 0x2000;
```

- [x] **Step 2: Bridge them from Bluepad32**

In `Bluepad32.cpp` after the `MISC_BUTTON_SYSTEM` line add:

```cpp
    if (uni_gp->misc_buttons & MISC_BUTTON_CAPTURE)   gp_in.buttons |= Gamepad::BUTTON_CAPTURE;
    if (uni_gp->misc_buttons & MISC_BUTTON_ASSISTANT) gp_in.buttons |= Gamepad::BUTTON_ASSISTANT;
```

- [x] **Step 3: Build**

Run: `cmake --build build`
Expected: exit 0. (No behavior change yet: no output driver reads the new bits.)

- [x] **Step 4: Commit**

```bash
git add Firmware/RP2040/src/Gamepad/Gamepad.h Firmware/RP2040/src/Bluepad32/Bluepad32.cpp
git commit -m "Carry Stadia Capture and Assistant buttons through the gamepad model"
```

---

### Task 4: XInput composite device with HID keyboard and Microsoft OS descriptors (checkpoint 2)

**Files:**
- Modify: `Firmware/RP2040/src/Descriptors/XInput.h`
- Modify: `Firmware/RP2040/src/USBDevice/DeviceDriver/XInput/XInput.h`
- Modify: `Firmware/RP2040/src/USBDevice/DeviceDriver/XInput/XInput.cpp`
- Modify: `Firmware/RP2040/src/USBDevice/DeviceDriver/DeviceDriver.h`
- Modify: `Firmware/RP2040/src/USBDevice/tud_callbacks.cpp`

**Interfaces:**
- Consumes: nothing new.
- Produces: `XInput::DESC_HID_REPORT[]`, `XInput::DESC_MS_OS_STRING[]`, `XInput::DESC_MS_OS_COMPAT_ID[]`, `XInput::MS_OS_VENDOR_CODE`; `XInputDevice::kb_wanted_` / `kb_sent_` members of type `hid_keyboard_report_t` (sent in Task 5); `DeviceDriver::echo_set_report()` virtual.

- [x] **Step 1: Descriptors**

In `Descriptors/XInput.h`, add `#include "class/hid/hid_device.h"` under the existing includes. Then:

Change `DESC_DEVICE`:
```cpp
		0x00,	      // bDeviceClass (composite: per-interface)
		0x00,	      // bDeviceSubClass
		0x00,	      // bDeviceProtocol
		0x40,	      // bMaxPacketSize0 64
		0x09, 0x12, // idVendor 0x1209 (pid.codes)
		0x01, 0x00, // idProduct 0x0001 (pid.codes test PID)
```
Reason for the VID/PID change: `xusb22.inf` matches `USB\VID_045E&PID_028E` at device level, which would beat the composite parent driver and swallow the keyboard interface. With a neutral VID/PID, Windows loads the composite parent and binds the Xbox driver to interface 0 via the compatible ID below. Keep `STRING_MANUFACTURER`/`STRING_PRODUCT` as they are.

Add after `DESC_STRING[]`:
```cpp
	static constexpr uint8_t MS_OS_VENDOR_CODE = 0x20;

	// Microsoft OS 1.0 string descriptor, served at string index 0xEE.
	// "MSFT100" in UTF-16LE followed by the vendor code and a pad byte.
	static const uint16_t DESC_MS_OS_STRING[] =
	{
		0x0312,                       // bLength 18, bDescriptorType STRING
		'M', 'S', 'F', 'T', '1', '0', '0',
		static_cast<uint16_t>(MS_OS_VENDOR_CODE), // bMS_VendorCode, bPad 0
	};
	static_assert(sizeof(DESC_MS_OS_STRING) == 18, "MS OS string descriptor must be 18 bytes");

	// Microsoft OS 1.0 Extended Compat ID descriptor: interface 0 is XUSB10.
	static const uint8_t DESC_MS_OS_COMPAT_ID[] =
	{
		0x28, 0x00, 0x00, 0x00, // dwLength 40
		0x00, 0x01,             // bcdVersion 1.00
		0x04, 0x00,             // wIndex 0x0004 (extended compat ID)
		0x01,                   // bCount 1
		0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, // reserved
		// Function section
		0x00,                   // bFirstInterfaceNumber 0
		0x01,                   // reserved
		'X', 'U', 'S', 'B', '1', '0', 0x00, 0x00, // compatibleID "XUSB10"
		0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, // subCompatibleID
		0x00, 0x00, 0x00, 0x00, 0x00, 0x00,             // reserved
	};
	static_assert(sizeof(DESC_MS_OS_COMPAT_ID) == 40, "MS OS compat ID descriptor must be 40 bytes");

	static const uint8_t DESC_HID_REPORT[] = { TUD_HID_REPORT_DESC_KEYBOARD() };
```

Change `DESC_CONFIGURATION`: header becomes
```cpp
		0x09,        // bLength
		0x02,        // bDescriptorType (Configuration)
		0x49, 0x00,  // wTotalLength 73 (9 + 39 XInput + 25 HID keyboard)
		0x02,        // bNumInterfaces 2
```
Leave every byte of the interface-0 block as it is. After its last endpoint (the `0x08, // bInterval 8` line), append:
```cpp
		// Interface 1: HID boot keyboard (Stadia Capture -> F14, Assistant -> F15)
		TUD_HID_DESCRIPTOR(1, 0, HID_ITF_PROTOCOL_KEYBOARD, sizeof(DESC_HID_REPORT), 0x82, 8, 10),
```
and after the array add:
```cpp
	static_assert(sizeof(DESC_CONFIGURATION) == 73, "XInput composite configuration descriptor length mismatch");
```
Note: `DESC_HID_REPORT` must be defined above `DESC_CONFIGURATION` because the macro uses `sizeof(DESC_HID_REPORT)`.

- [x] **Step 2: Driver header**

In `XInput.h`, add `#include "class/hid/hid.h"` and two private members:
```cpp
    hid_keyboard_report_t kb_wanted_{};
    hid_keyboard_report_t kb_sent_{};
```
and a public override:
```cpp
    bool echo_set_report() const override { return false; }
```

- [x] **Step 3: Driver base class**

In `DeviceDriver.h`, inside `class DeviceDriver` public section add:
```cpp
    // Whether tud_hid_set_report_cb should echo the host's SET_REPORT back as an input report.
    // Upstream behavior is to echo; a keyboard must not (LED reports would look like key presses).
    virtual bool echo_set_report() const { return true; }
```

In `tud_callbacks.cpp`, change `tud_hid_set_report_cb` to:
```cpp
void tud_hid_set_report_cb(uint8_t itf, uint8_t report_id, hid_report_type_t report_type, uint8_t const *buffer, uint16_t bufsize)
{
	DeviceDriver* driver = DeviceManager::get_instance().get_driver();
	driver->set_report_cb(itf, report_id, report_type, buffer, bufsize);
	if (driver->echo_set_report())
	{
		tud_hid_report(report_id, buffer, bufsize);
	}
}
```

- [x] **Step 4: Driver callbacks**

In `XInput.cpp` replace these four functions:

```cpp
uint16_t XInputDevice::get_report_cb(uint8_t itf, uint8_t report_id, hid_report_type_t report_type, uint8_t *buffer, uint16_t reqlen)
{
    // Only the HID keyboard interface reaches this callback (XInput is a vendor interface).
    uint16_t len = std::min<uint16_t>(reqlen, sizeof(kb_sent_));
    std::memcpy(buffer, &kb_sent_, len);
    return len;
}

bool XInputDevice::vendor_control_xfer_cb(uint8_t rhport, uint8_t stage, tusb_control_request_t const *request)
{
    if (stage != CONTROL_STAGE_SETUP)
    {
        return true;
    }
    // Microsoft OS 1.0 feature request: GET Extended Compat ID (wIndex 0x0004)
    if (request->bmRequestType_bit.direction == TUSB_DIR_IN &&
        request->bRequest == XInput::MS_OS_VENDOR_CODE &&
        request->wIndex == 0x0004)
    {
        return tud_control_xfer(rhport, request,
                                const_cast<uint8_t*>(XInput::DESC_MS_OS_COMPAT_ID),
                                sizeof(XInput::DESC_MS_OS_COMPAT_ID));
    }
    return false;
}

const uint16_t * XInputDevice::get_descriptor_string_cb(uint8_t index, uint16_t langid)
{
    if (index == 0xEE)
    {
        return XInput::DESC_MS_OS_STRING;
    }
    if (index >= (sizeof(XInput::DESC_STRING) / sizeof(XInput::DESC_STRING[0])))
    {
        return nullptr;
    }
	const char *value = reinterpret_cast<const char*>(XInput::DESC_STRING[index]);
	return get_string_descriptor(value, index);
}

const uint8_t * XInputDevice::get_hid_descriptor_report_cb(uint8_t itf)
{
    return XInput::DESC_HID_REPORT;
}
```
Add `#include <algorithm>` at the top of `XInput.cpp`.

- [x] **Step 5: Build**

Run: `cmake --build build`
Expected: exit 0, all `static_assert`s hold.

- [x] **Step 6: On-device check (checkpoint 2)** (passed 2026-09-22)

User flashes the uf2, switches to XInput mode if needed. Expected in Device Manager: one "USB Composite Device" with children "Xbox 360 Controller for Windows" (under Xbox 360 Peripherals) and "HID Keyboard Device". Gamepad inputs and rumble still work in `joy.cpl` and Steam.

If Windows shows the controller but no keyboard, or a keyboard but the controller is only a generic HID device: delete `HKLM\SYSTEM\CurrentControlSet\Control\UsbFlags\120900010114` (VID 1209, PID 0001, bcdDevice 0114) so Windows re-queries the OS descriptor, unplug, replug. If still failing, stop and revisit the fallback in the spec.

- [x] **Step 7: Commit**

```bash
git add Firmware/RP2040/src/Descriptors/XInput.h \
        Firmware/RP2040/src/USBDevice/DeviceDriver/XInput/XInput.h \
        Firmware/RP2040/src/USBDevice/DeviceDriver/XInput/XInput.cpp \
        Firmware/RP2040/src/USBDevice/DeviceDriver/DeviceDriver.h \
        Firmware/RP2040/src/USBDevice/tud_callbacks.cpp
git commit -m "Make XInput mode a composite device with a HID keyboard interface"
```

---

### Task 5: Send F14 and F15 from the keyboard interface (checkpoint 3)

**Files:**
- Modify: `Firmware/RP2040/src/USBDevice/DeviceDriver/XInput/XInput.cpp` (`process`)

**Interfaces:**
- Consumes: `Gamepad::BUTTON_CAPTURE`, `Gamepad::BUTTON_ASSISTANT` (Task 3); `kb_wanted_`, `kb_sent_` (Task 4).

- [x] **Step 1: Build the wanted report when the pad updates**

Inside `XInputDevice::process`, inside the `if (gamepad.new_pad_in())` block, right after `Gamepad::PadIn gp_in = gamepad.get_pad_in();`, add:

```cpp
        hid_keyboard_report_t kb{};
        uint8_t key_count = 0;
        if (gp_in.buttons & Gamepad::BUTTON_CAPTURE)   kb.keycode[key_count++] = HID_KEY_F14;
        if (gp_in.buttons & Gamepad::BUTTON_ASSISTANT) kb.keycode[key_count++] = HID_KEY_F15;
        kb_wanted_ = kb;
```

- [x] **Step 2: Send it when it changes and the endpoint is free**

At the end of `XInputDevice::process` (after the rumble block), add:

```cpp
    if (std::memcmp(&kb_wanted_, &kb_sent_, sizeof(kb_sent_)) != 0 && tud_hid_n_ready(0))
    {
        if (tud_hid_n_report(0, 0, &kb_wanted_, sizeof(kb_wanted_)))
        {
            kb_sent_ = kb_wanted_;
        }
    }
```
Add `#include "class/hid/hid_device.h"` at the top of `XInput.cpp`.

- [x] **Step 3: Build**

Run: `cmake --build build`
Expected: exit 0.

- [x] **Step 4: On-device check (checkpoint 3)** (passed 2026-09-22)

User flashes the uf2. In a key tester (for example the Windows `Notepad` won't show F14; use a browser key-event page or `PowerShell` with a key reader), pressing Capture reports F14 and Assistant reports F15, both release cleanly. Holding both reports both. Gamepad inputs and rumble still work. Mode switch Menu + LB + RB still enters web-app mode; Menu + D-pad Up returns to XInput.

- [x] **Step 5: Commit**

```bash
git add Firmware/RP2040/src/USBDevice/DeviceDriver/XInput/XInput.cpp
git commit -m "Send Stadia Capture and Assistant as F14 and F15 over the keyboard interface"
```

---

### Task 6: Project docs and push

**Files:**
- Create: `AGENTS.md` (repo root)
- Modify: `README.md` (short "Fork notes" section at top)

- [x] **Step 1: Write AGENTS.md** covering: what the fork is, build command, files touched, boundaries (don't change other output modes; keep XInput interface 0 unchanged), current status, recent changes, lessons learned (VID/PID and the UsbFlags cache, the SET_REPORT echo).

- [x] **Step 2: README fork note** with the Stadia mapping table and a link to the spec and plan.

- [x] **Step 3: Commit and push**

```bash
git add AGENTS.md README.md docs plans
git commit -m "Document the Stadia extra-buttons fork"
git push -u origin stadia-extras
```
