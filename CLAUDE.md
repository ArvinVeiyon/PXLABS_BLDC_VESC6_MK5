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
