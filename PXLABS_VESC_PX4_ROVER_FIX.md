# PXLabs VESC Firmware - PX4 Rover UAVCAN Fix

## Version Information

| Item | Value |
|------|-------|
| **PXLabs Release** | `pxlabs-v6.06-px4-rover` |
| **Base VESC Version** | 6.06 (development) |
| **Official VESC Tag** | 6.00 |
| **Commit Hash** | `6c8f693a183d0e0f89adac0993ca334f6aec21e6` |
| **Author** | Vinoth Pandiyan <vinothpandiyan@hotmail.com> |
| **Date** | 2026-02-07 |

---

## Git References

### Branches
| Branch | Description |
|--------|-------------|
| `px4_uavcan_reversible` | Current working branch (active) |
| `pxlabs/v6.06-px4-rover-fix` | Stable release branch |
| `master` | Original upstream VESC |
| `release_6_06` | Official VESC 6.06 |

### Tags
| Tag | Description |
|-----|-------------|
| `pxlabs-v6.06-px4-rover` | PXLabs stable release tag |

---

## Fix Details

### Problem
PX4 Rover (Differential) with UAVCAN sends:
- **0-8191** for throttle (4096 = neutral)
- **0** when disarmed

Original VESC code interpreted `0` as full reverse (`-1.0`), causing motor spin when disarmed.

### Solution
Modified `libcanard/canard_driver.c`:
- Values **0-99** → treated as "disarmed/stop"
- Values **100-8191** → mapped to throttle (-1.0 to +1.0)
- Value **4096** → neutral (0.0)

### Files Changed
```
libcanard/canard_driver.c | 43 insertions(+), 2 deletions(-)
```

---

## PX4 Configuration

| Setting | Value |
|---------|-------|
| **Geometry** | Rover (Differential) |
| **Output** | UAVCAN |
| **ESC Range** | Min: 1, Max: 8191 |
| **Rev Range** | Enabled |
| **Failsafe** | -1 |

---

## Build Setup & Commands

### Git Repository Path
```
/home/pxlabs/motor_source/bldc
```

### Build Procedure
```bash
# Navigate to repository
cd /home/pxlabs/motor_source/bldc

# Checkout stable release
git checkout pxlabs-v6.06-px4-rover

# Clean and build for VESC 6 MK5
make fw_60_mk5_clean && make fw_60_mk5

# Firmware output location
/home/pxlabs/motor_source/bldc/build/60_mk5/60_mk5.bin
```

### Flash via VESC Tool
1. Open VESC Tool
2. Connect to VESC
3. Go to **Firmware** → **Custom File**
4. Select `build/60_mk5/60_mk5.bin`
5. Click Upload

---

## Repository

| Remote | URL |
|--------|-----|
| Origin | https://github.com/vedderb/bldc.git |
