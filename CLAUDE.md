# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the VESC (Vedder Electronic Speed Controller) firmware - an open source motor controller firmware for BLDC/FOC motor control. It runs on STM32F4 microcontrollers with ChibiOS RTOS.

## Build Commands

### Initial Setup
```bash
make arm_sdk_install    # Install ARM GCC toolchain (one-time)
```

### Building Firmware
```bash
make                    # Show help and list all supported boards
make <board_name>       # Build firmware for specific board (e.g., make 100_250)
make all_fw             # Build firmware for all boards
```

Build output goes to `build/<board_name>/`. The firmware binary is `build/<board_name>/<board_name>.bin`.

### Flashing
```bash
make <board_name>_flash      # Build and flash via OpenOCD/SWD
make <board_name>_flash_only # Flash without rebuilding
```

### Custom Hardware
```bash
make fw_custom HW_SRC=path/to/hw.c HW_HEADER=path/to/hw.h
```

### Unit Tests
```bash
make all_ut        # Build all unit tests
make all_ut_run    # Build and run all unit tests
```

### IDE Setup
```bash
pip install aqtinstall
make qt_install
# Open Project/Qt Creator/vesc.pro in tools/Qt/Tools/QtCreator/bin/qtcreator
```

## Architecture

### Core Motor Control Stack
- **`motor/mcpwm_foc.c`** - Field-Oriented Control (FOC) implementation, main control algorithm
- **`motor/mcpwm.c`** - BLDC commutation control (legacy mode)
- **`motor/mc_interface.c`** - High-level motor control API, abstracts FOC/BLDC modes
- **`motor/foc_math.c`** - FOC mathematical operations (Park/Clarke transforms)

### Hardware Abstraction
- **`hwconf/`** - Board-specific hardware configurations (one `hw_<board>.h` per board)
- **`hwconf/hw.h`** - Master hardware include that pulls in the selected board config
- Hardware is selected at compile time via `HW_HEADER` and `HW_SOURCE` build macros

### Encoder Support (`encoder/`)
Multiple encoder types: ABI, AS504x, AS5x47U, BiSS-C, MT6816, SinCos, TLE5012, TS5700N8501

### Communication (`comm/`)
- **`commands.c`** - VESC Tool protocol command handler
- **`comm_can.c`** - CAN bus communication
- **`comm_usb.c`** - USB CDC communication
- **`packet.c`** - Packet framing/parsing

### Application Layer (`applications/`)
Input modes: ADC, PPM, Nunchuk, UART, PAS (pedal assist), custom apps

### LispBM Scripting (`lispBM/`)
Embedded Lisp interpreter for runtime scripting. Extensions in `lispif_vesc_extensions.c`.

### DroneCAN/UAVCAN (`libcanard/`)
DroneCAN protocol support via libcanard. Driver in `canard_driver.c`.

### Key Configuration Files
- **`conf_general.h`** - Global config, firmware version, feature flags
- **`datatypes.h`** - All data structures and enums
- **`motor/mcconf_default.h`** - Default motor configuration
- **`applications/appconf_default.h`** - Default application configuration

### RTOS and HAL
- ChibiOS 3.0.5 in `ChibiOS_3.0.5/`
- Timer allocation: TIM1/TIM8 (MCPWM), TIM2 (FOC), TIM3 (encoder/servo), TIM4 (LEDs/encoder), TIM5 (system timer)

## Hardware Configuration Pattern

Each board has files in `hwconf/<vendor>/<board>/`:
- `hw_<board>.h` - Pin definitions, limits, features
- `hw_<board>_core.h` - Shared core definitions (if multiple variants)
- `hw_<board>.c` - Board-specific initialization

Key hardware defines: `HW_NAME`, pin mappings (`HW_HALL_*`, `HW_ENC_*`), current/voltage limits, shunt resistor values.

## Important Conventions

- Motor configuration struct: `mc_configuration` (see `datatypes.h`)
- App configuration struct: `app_configuration`
- All floating point uses single precision (`float`, not `double`)
- Build uses `-fsingle-precision-constant -Wdouble-promotion` to catch accidental double usage

---

# PXLABS fork

This repository is a **fork of `vedderb/bldc`** carrying the firmware for the PXLABS rover's four
wheel motor controllers. Everything above this line is upstream VESC documentation and applies
unchanged; everything below is specific to this fork.

## Hardware target

**60_mk5** (VESC 6.0 MK5), four units, one per wheel.

```bash
export PATH=/opt/gcc-arm-none-eabi-9-2020-q2-update/bin:$PATH
make fw_60_mk5            # -> build/60_mk5/60_mk5.bin
```

Toolchain is **ARM GCC 9.3.1** (`9-2020-q2-update`). Do not substitute a newer one — ChibiOS 3.0.5
has not been validated against it, and a binary built with a different compiler is not the artifact
any test procedure was written against. That release shipped x86_64 Linux hosts only, so the
firmware cannot be built on an arm64 machine such as the companion Pi.

> `make fw_60_mk5_clean` **wipes `build/60_mk5/` entirely.** It has destroyed the only local copy of
> a released binary before. Copy anything you care about out of that directory first.

The firmware embeds the git hash at build time. **Build after committing, not before** — a binary
compiled from a dirty tree or from staged-but-uncommitted work bakes in the *parent* commit's hash
and will misreport its version in VESC Tool.

## Branch model

| Branch | Role |
|---|---|
| `pxlabs-6.06-rover-uavcan_main` | main dev branch — all work lands here |
| `pxlabs-6.06-rover-brake-rc` | RC brake feature, awaiting bench test |
| `pxlabs-release-6.06-rover-r1` | **frozen** stable cut |
| `master`, `release_6_00`, `release_6_06` | untouched upstream |

**Release branches and tags are frozen.** Never commit to them, never move, delete or recreate a
published tag, and never branch feature work off them. A new stable state gets a new release branch
and a new tag. Tags and GitHub releases are for **stable** builds only — a feature awaiting testing
ships as a branch, with no tag and no release.

## Code conventions

Upstream code being replaced is **commented out and tagged `[UPSTREAM]`**, never deleted. The
replacement sits directly below, tagged `[PXLABS]`. This keeps the delta against upstream legible
when merging.

Prefer changes that avoid an `APPCONF_SIGNATURE` bump (`confgenerator.h`). Adding a field to
`app_configuration` changes the signature, which means VESC Tool refuses to talk to the firmware
until a matching `parameters_appconf.xml` exists on the tool side. Compile-time `#define`s avoid
this entirely — see `UAVCAN_BRAKE_SLOT` in `libcanard/canard_driver.c`.

## CAN / DroneCAN

The four VESCs run **`can_mode = CAN_MODE_UAVCAN` at 1 Mbit**, commanded by PX4 over DroneCAN.

**VESC Tool's CAN-forward cannot reach them.** In UAVCAN mode `comm/comm_can.c` discards every
VESC-protocol frame, and `comm_can_ping()` returns false. Scan CAN finds nothing. Configure and
flash each unit over **USB**, or use DroneCAN `file.BeginFirmwareUpdate`, which is fully implemented
in `libcanard/canard_driver.c` — serve the raw `.bin`, the firmware writes its own size+CRC header.

DroneCAN node ID is `controller_id`; dynamic node allocation is disabled.

| Wheel | `controller_id` | `uavcan_esc_index` |
|---|---|---|
| Front right | 10 | 0 |
| Front left | 11 | 1 |
| Rear right | 12 | 2 |
| Rear left | 13 | 3 |

## Repository layout additions

- **`PXLABS_VESC_PX4_ROVER_FIX.md`** — the PXLABS release note and change history. `CHANGELOG.md`
  and `README.md` are pure upstream; do not add PXLABS entries to them.
- **`PXLABS_RC_BRAKE_TESTING.md`** — bench acceptance procedure for the RC brake feature.
- **`Motor_Config_Bldc/`** — VESC Tool `mcconf`/`appconf` XML exports, one pair per wheel. Config
  data only; does not affect the built binary.
