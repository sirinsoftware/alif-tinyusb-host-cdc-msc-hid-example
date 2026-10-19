# TinyUSB USB Host CDC/MSC/HID Example for Alif Semiconductor Ensemble E7

A USB **host** example for the [Alif Semiconductor](https://www.alifsemi.com)
Ensemble **E7 DevKit** (`AE722F80F55D5LS`), built on
[TinyUSB](https://github.com/hathach/tinyusb).

The Ensemble xHCI controller acts as the USB host: plug a USB serial adapter,
a flash drive, a keyboard or a mouse into the board and the example enumerates
it and reports what it sees on the serial console. All three class drivers —
CDC-ACM, mass storage and HID — are active at the same time, and a USB hub is
supported, so several devices can be attached at once.

| | |
|---|---|
| **Target device** | Alif Ensemble `AE722F80F55D5LS`, Cortex-M55 HP and HE cores |
| **Target board** | Alif Ensemble E7 DevKit Gen 2 (`DevKit-E7`) |
| **USB role** | Host, via the Ensemble xHCI controller (`hcd_ensemble.c`) |
| **USB classes** | CDC-ACM (plus FTDI / CP210x / CH34x serial), MSC, HID, Hub |
| **Published by** | Sirin Software, with Alif Semiconductor's permission |
| **Status** | Development snapshot, provided as-is — see below |

---

## Publication and collaboration statement

This example was developed as part of the collaboration between **Sirin
Software** and **Alif Semiconductor**, and is published by Sirin Software with
Alif Semiconductor's permission.

This repository is **not** an official Alif Semiconductor examples repository.

## Snapshot / As-Is

> This repository provides **development snapshots** for the Alif Semiconductor
> Ensemble platform. They are provided **as-is** for demonstration and development
> purposes and should be considered snapshots rather than finalized demos.
>
> Customer support is not provided for these snapshots. Future finalized versions
> may be supported separately.

---

## Target Setup Requirements

### Hardware

- **Alif Ensemble E7 DevKit Gen2**
- **Micro-USB cable** (included with the DevKit)
- A **host computer** (Linux, macOS or Windows) with terminal emulation software
- One or more **USB devices to attach**: a USB-to-serial adapter, a USB flash
  drive, a USB keyboard or mouse, optionally through a USB hub
- Optional: **JTAG debugger**, such as SEGGER J-Link or Arm ULINKpro

### USB Connections

The DevKit provides two USB interfaces used by the example:

#### PRG USB (Micro-USB)

Connect a USB cable from the host computer to the **PRG USB** connector.

This interface provides a two-channel USB-to-UART adapter:
- used by the SETOOLS applications to program images into MRAM.
- used for the example's console output and keyboard input. The example
  configures it as 115200 baud, 8-N-1.

#### HOST USB (USB-A receptacle)

This interface is connected directly to the Ensemble SoC. In this example it
operates in **host** role, so it is where the USB device under test is
attached rather than being connected to the host computer.

The USB host interface is powered only from the DC Jack. Connect a 5 V DC power
supply to the J9 connector on the DevKit. The interface provides the required
VBUS power to the attached USB device.

### GPIO used by the example

| Function | Board definition | GPIO |
|---|---|---|
| Status LED | `BOARD_LEDRGB1_R_*` | GPIO port 6, pin 2 (RGB LED 1, red) |

---

## Getting started

Clone with submodules

```bash
git clone --recurse-submodules <repository-url>
```

The example is built with the CMSIS solution flow described below. There is no
CMake, FreeRTOS or Zephyr build in this repository.

---

## CMSIS solution build (VS Code / CMSIS-Toolbox)

### Prerequisites

- VS Code with the recommended extensions:
  `arm.environment-manager`, `arm.cmsis-csolution`,
  `ms-vscode.cpptools-extension-pack`, `marus25.cortex-debug`.
- The tools installed by the Arm Environment Manager: CMake ≥ 3.28.4,
  Ninja ≥ 1.12.0,`arm-none-eabi-gcc` ≥ 13.2.1-0, CMSIS-Toolbox ≥ 2.6.1.
- CMSIS packs (installed by the task below): `ARM::CMSIS@6.2.0`,
  `ARM::CMSIS-FreeRTOS@11.2.0`, `AlifSemiconductor::Ensemble@2.0.2`.
- For the Security Toolkit programming tasks: the Alif Security Toolkit
  (SETOOLS) installed locally, and a VS Code setting `alif.setools.root`
  pointing at its installation directory. This setting is not stored in the
  repository; add it to your user or workspace settings.

Setting up the Alif VS Code environment itself is documented by Alif in the
[VS Code Getting Started Template](https://github.com/alifsemi/alif_vscode-template);
follow that first.

### Build

1. Open the repository folder in VS Code (**File → Open Folder**).
2. When the **Arm Environment Activation** prompt appears, choose
   **Allow For Current Workspace**.
3. **F1 → Tasks: Run Task → First time pack installation** (only needed once
   per machine; it runs `cpackget` to install the three CMSIS packs listed above).
4. **F1 → CMSIS: Manage Solution Settings** and select a target type:
   `E7-HP` (Cortex-M55 High Performance core) or `E7-HE` (High Efficiency core).
5. Build with the CMSIS extension's build (hammer) button, or
   **F1 → Tasks: Run Task → cmsis-csolution.build: Build**.

Build types are `release` (balanced optimisation, debug info off) and `debug`
(no optimisation, debug info on). GCC is the selected compiler.

### Output location

```
out/<project>/<target>/<build-type>/<project>.elf
out/<project>/<target>/<build-type>/<project>.bin
```

for example `out/host_cdc_msc_hid/E7-HP/release/host_cdc_msc_hid.elf`.

### Program the board

Two paths are pre-configured in [`.vscode/tasks.json`](.vscode/tasks.json):

**Alif Security Toolkit (writes to MRAM)**

**F1 → Tasks: Run Task → Program with Security Toolkit**.
Variants for selecting a COM port and for installing debug stubs are provided
as separate tasks. The MRAM load addresses come from the per-core
configuration files in [`.alif/`](.alif): `0x80200000` for M55_HP and
`0x80000000` for M55_HE.

**pyOCD over CMSIS-DAP**

**F1 → Tasks: Run Task → CMSIS Load** (or `CMSIS Load+Run`). Related tasks:
`CMSIS Run`, `CMSIS Erase`, `CMSIS TargetInfo`.

### Debug

[`.vscode/launch.json`](.vscode/launch.json) provides a J-Link configuration
("Alif Ensemble Debug (Cortex-Debug)", using
[`.alif/JLinkDevices.xml`](.alif/JLinkDevices.xml) and RTT console) and pyOCD
gdb launch/attach configurations for the M55_HP and M55_HE cores.

---

## The example in detail

TinyUSB runs in host role on the Ensemble xHCI controller. `main()` initialises
the board, brings up the host stack with `tusb_init()`, and then runs
`tuh_task()` together with the LED blinker and the per-class application tasks
in a super-loop. There is no RTOS.

The three class applications are kept in separate files and all run
concurrently:

- **CDC-ACM** ([`cdc_app.c`](host_cdc_msc_hid/src/cdc_app.c)) — data received
  from an attached USB serial device is printed on the console, and characters
  typed on the console are forwarded to the device. On top of true CDC-ACM, the
  FTDI, CP210x and CH34x vendor serial drivers are enabled, so common USB-serial
  adapters work too. Line control (DTR/RTS) and line coding (115200 8-N-1) are
  applied on enumeration.
- **MSC** ([`msc_app.c`](host_cdc_msc_hid/src/msc_app.c)) — when a mass storage
  device is mounted, a SCSI INQUIRY is issued and the vendor, product and
  revision strings are printed along with the capacity, block count and block
  size. The SCSI response buffer is placed in the USB DMA memory section, which
  is what the Ensemble controller requires.
- **HID** ([`hid_app.c`](host_cdc_msc_hid/src/hid_app.c)) — the interface
  protocol of each mounted HID interface is reported. Keystrokes from an
  attached keyboard are echoed to the console; mouse reports are printed as
  `(x y wheel)` deltas. Non-boot-protocol devices are handled through the
  generic report parser.

Hub support is enabled, and the device table is sized for up to four devices,
so a hub with several peripherals behind it can be attached.

This example is a practical check of the Alif host controller driver: device
attachment and detachment, enumeration through a hub, control transfers, and
concurrent bulk and interrupt traffic from different device classes.

**Run.**

1. Program the board and reset it.
2. Connect **PRG USB** to the host computer and open the serial port at
   115200 8-N-1.
3. Attach a USB device to **HOST USB**.
4. Watch the console; type into it to drive an attached serial device.

**Expected result.**

- The console prints `TinyUSB Host CDC MSC HID Example` on start-up.
- The status LED blinks, indicating the loop is running.
- Attaching a device prints `A device with address N is mounted`, followed by
  class-specific output: the inquiry and capacity lines for a flash drive, the
  HID interface protocol for a keyboard or mouse, the baud rate and framing for
  a serial adapter.
- Keystrokes from an attached keyboard appear on the console; moving an
  attached mouse prints a stream of `(x y wheel)` lines.
- Removing a device prints `A device with address N is unmounted`.
- TinyUSB's own logging is off by default (`CFG_TUSB_DEBUG 0` in
  [`tusb_config.h`](host_cdc_msc_hid/src/tusb_config.h)); raise it to see stack
  level traces on the same serial port.

---

## Licensing

This repository contains software from multiple sources and under multiple
applicable licences. **No single licence covers the whole repository.**

- **TinyUSB source and TinyUSB-derived source** (the example sources under
  `host_cdc_msc_hid/src/`, and the `lib/tinyusb` submodule except where noted):
  MIT Licence — see [`LICENSE-MIT.txt`](LICENSE-MIT.txt) and the notices in the
  individual files. The copyright notices name Ha Thach (tinyusb.org).
- **Alif Semiconductor material** (the Ensemble xHCI host controller sources
  under `lib/tinyusb/src/portable/xhci/`, the Alif BSP inside `lib/tinyusb`,
  the `lib/alif_common-app-utils` submodule, the CMSIS pack configuration under
  `host_cdc_msc_hid/RTE/`, and the project scaffolding derived from the Alif VS
  Code template): the Alif Semiconductor Software License Agreement — see
  [`LICENSE-ALIF.txt`](LICENSE-ALIF.txt) and the notices in the individual
  files.
- **Azure RTOS USBX material.** Four files compiled into this example
  (`ux_system_initialize.c` and the three `ux_utility_memory_*` files in
  `lib/tinyusb/src/portable/xhci/`) carry a Microsoft Corporation copyrightS
  notice and refer to the Microsoft Software License Terms for Microsoft Azure
  RTOS. That licence text is not included in this repository. See
  [`THIRD_PARTY_NOTICES.txt`](THIRD_PARTY_NOTICES.txt) before redistributing or
  building on this code.
- **Other third-party components** (Arm CMSIS under Apache-2.0): see
  [`THIRD_PARTY_NOTICES.txt`](THIRD_PARTY_NOTICES.txt) and the components' own
  licence files.

Modifications developed for the Alif Semiconductor platform are marked as such
in the affected files; they do not replace the upstream copyright or licence
notices in those files, which continue to apply to the original material.

---

## Support and issues

This repository is a **published snapshot**. It is maintained on a best-effort
basis by Sirin Software and is not covered by an Alif Semiconductor
customer-support commitment.

- Questions or problems with the content of **this repository** can be raised
  through this repository's issue tracker on GitHub. Opening an issue here does
  not create any support obligation for Alif Semiconductor.
- Questions about **Alif Semiconductor devices, SDKs, packs or tools** belong in
  Alif's own support channels; see [alifsemi.com](https://www.alifsemi.com).
- Questions about **TinyUSB itself** belong upstream at
  [hathach/tinyusb](https://github.com/hathach/tinyusb).

---

## References

- [Alif Ensemble E7 DevKit documentation](https://alifsemi.com/support/kits/ensemble-e7devkit/)
- [Alif VS Code Getting Started Template](https://github.com/alifsemi/alif_vscode-template)
- [TinyUSB documentation](https://docs.tinyusb.org)
- [CMSIS-Toolbox](https://github.com/Open-CMSIS-Pack/cmsis-toolbox)
