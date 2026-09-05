# PXLabs VESC Firmware — PX4 Rover (DroneCAN/UAVCAN)

Custom VESC 6.06 firmware for the PXLabs 4-wheel differential rover, driven by a
PX4 flight controller over DroneCAN. Four VESC 6 MK5 controllers, one per wheel.

Last updated: 2026-09-05

---

## Version Information

| Item | Value |
|------|-------|
| **Current stable release** | `v6.06.0-pxlabs-rover-r1` |
| **Release commit** | `6fc2cf17337ce6d1e494c47a5969b93acac0661e` |
| **Release date** | 2026-02-08 |
| **Base VESC Version** | 6.06 (development) |
| **Hardware target** | `60_mk5` (VESC 6 MK5) |
| **Author** | Vinoth Pandiyan <vinothpandiyan@hotmail.com> |

---

## Git References

### Branches
| Branch | Description |
|--------|-------------|
| `pxlabs-6.06-rover-uavcan_main` | **Main development branch (active)** — work here |
| `pxlabs-release-6.06-rover-r1` | Stable release branch r1 — content-identical to the r1 tag |
| `pxlabs-6.06-rover-brake-rc` | RC brake feature — implemented, **not yet bench-tested or merged** |
| `master` | Original upstream VESC |
| `release_6_06` | Official VESC 6.06 |

### Tags
| Tag | Commit | Description |
|-----|--------|-------------|
| `v6.06.0-pxlabs-rover-r1` | `6fc2cf17` | PXLabs stable release r1 |

`r1` = release 1. It is the only PXLabs tag; every other tag in this repo is upstream vedderb/bldc.

### GitHub Release
`v6.06.0-pxlabs-rover-r1` carries flashable assets (`60_mk5.bin`, `.hex`, `.elf`, `.map`):
https://github.com/ArvinVeiyon/PXLABS_BLDC_VESC6_MK5/releases/tag/v6.06.0-pxlabs-rover-r1

This is the rollback target if a development build misbehaves.

---

## Change History

| Date | Commit | Branch | Change |
|------|--------|--------|--------|
| 2026-02-07 | `6c8f693a` | both | Fix PX4 UAVCAN reversible ESC disarm behavior — **the r1 firmware fix** |
| 2026-02-07 | `cfab6c0e` | both | Add docs and bootloader folder |
| 2026-02-07 | `879ab45e` | both | Remove local-only bldc-bootloader folder from repo |
| 2026-02-08 | `6fc2cf17` | both | Update docs with r1 release branch and tag info — **tagged r1** |
| 2026-08-15 | `f80e5781` | dev | Add `Motp_Config_Bldc/` — 43 VESC Tool motor/app config XMLs (folder later renamed) |
| 2026-08-15 | `dcc35366`+`05deb3e8` | release-r1 | Same XMLs added then reverted (a released branch should not gain files post-tag) |
| 2026-09-05 | `a75a0dbf` | brake-rc | Add RC brake channel via UAVCAN RawCommand spare slot |
| 2026-09-05 | `e04bc633` | brake-rc | Add RC brake bench test methodology (`PXLABS_RC_BRAKE_TESTING.md`) |
| 2026-09-05 | `369e5f9d` | brake-rc | Update config folder with current motor and app configs |

**No firmware source changed between 2026-02-07 and 2026-09-05.** Everything in between was
documentation and VESC Tool configuration data. The brake commit is the first firmware change
since r1.

### `Motor_Config_Bldc/`
VESC Tool motor (`mcconf`) and app (`appconf`) configuration XMLs, one pair per wheel.
Configuration data, not firmware — these do not affect the built binary.

Renamed from `Motp_Config_Bldc/` (misspelling) and pruned on 2026-09-05 to the current
set of eight only: four `mcconf` dated 15 Aug 2026 and four `appconf` dated 16 Aug 2026.
The 36 older/duplicate files removed then remain in git history at `369e5f9d` and earlier.

| Wheel | `controller_id` | `uavcan_esc_index` |
|---|---|---|
| Front right | 10 | 0 |
| Front left | 11 | 1 |
| Rear right | 12 | 2 |
| Rear left | 13 | 3 |

All four: `can_mode` UAVCAN, `can_baud_rate` 1M, `uavcan_raw_mode` CURRENT.

---

## r1 Fix Details — UAVCAN disarm behavior

### Problem
PX4 Rover (Differential) with `CA_R_REV` enabled sends a reversible throttle as
**0–8191 with 4096 = neutral**, and sends **0 when disarmed**. Stock VESC code interpreted
`0` as full reverse (`-1.0`), so a disarmed rover would drive backwards.

### Solution
`libcanard/canard_driver.c`:
- Values **0–99** → treated as "disarmed/stop"
- Values **100–8191** → mapped to throttle (−1.0 … +1.0)
- Value **4096** → neutral (0.0)

### Files Changed
```
libcanard/canard_driver.c | 41 +++++++++++++++++++++++++++++++++++++--
1 file changed, 40 insertions(+), 1 deletion(-)
```

---

## RC Brake Channel — IN PROGRESS (branch `pxlabs-6.06-rover-brake-rc`)

> **Status: implemented and builds clean, but NOT bench-tested and NOT merged.**
> Do not flash to a vehicle in service. Rollback is the r1 release binary.

### Problem
`uavcan_raw_mode` forces a choice: `UAVCAN_RAW_MODE_CURRENT` gives reverse on the lower half
of the throttle stick, `UAVCAN_RAW_MODE_CURRENT_NO_REV_BRAKE` gives brake there. You cannot
have both. Braking needs its own axis.

### Solution
PX4 already broadcasts an `esc.RawCommand` array to all four VESCs. The brake demand rides in
a spare slot (index 4) of that existing array — no new DSDL, no new message, no PX4 code.

`libcanard/canard_driver.c`, in `handle_esc_raw_command()`:
```c
#define UAVCAN_BRAKE_SLOT       4      /* 0-based index into cmd.cmd.data[] */
#define UAVCAN_BRAKE_THRESHOLD  0.05f  /* below this, brake is released */
```
Reads `cmd.cmd.data[4]`, scales `0..8191` to `0.0..1.0`, and when above the threshold calls
`mc_interface_set_brake_current_rel()` instead of the normal raw-mode handling.

**Brake overrides throttle via an if/else around the existing raw-mode switch, not an early
`return`.** `timeout_reset()` sits after the switch; skipping it would let the command-timeout
watchdog fire mid-brake.

### Safety property
Slot value `0` means brake released. PX4 also sends `0` on that slot when disarmed (there is no
`UAVCAN_EC_DIS<n>` parameter, so the mixer's disarmed value is hardcoded 0) and on RC loss
(NAN resolves to the same disarmed value). Brake-off is therefore the failsafe by construction.

### Files Changed
```
libcanard/canard_driver.c | 64 ++++++++++++++++++++++++++++++---------------
1 file changed, 46 insertions(+), 18 deletions(-)
```

### Outstanding before merge
1. Confirm which physical RC channel is the brake stick (QGC Radio page).
2. **Fix that channel's trim.** PX4 normalizes as `interpolateNXY(value, {min, trim, max}, {-1,0,+1})`.
   Channel 3 currently has `RC3_TRIM == RC3_MIN == 1001`, making the first segment zero-width —
   the curve jumps from −1.0 at the bottom stop to 0.0 just above it, i.e. instant ~50% brake.
   PX4 auto-repairs this only for the throttle channel. Set `RC3_TRIM` to `(min+max)/2` ≈ 1483.
3. Set the PX4 parameters below.
4. Bench-verify with wheels off the ground, including that `esc_status.esc_errorcount` stays 0
   (the specific check for a missed `timeout_reset()`).

---

## PX4 Configuration

### Vehicle
| Setting | Value |
|---------|-------|
| Geometry | Rover (Differential) |
| Output | UAVCAN / DroneCAN |
| `CA_R_REV` | 3 (motors 1 and 2 reversible) |
| `UAVCAN_EC_FUNC1..4` | 101, 102, 101, 102 (Motor1/2 duplicated per side) |
| `UAVCAN_EC_MIN1..4` | 10 |
| `UAVCAN_EC_MAX1..4` | 8191 |
| `UAVCAN_EC_FAIL1..4` | −1 |

> A later bench session revised `UAVCAN_EC_MIN1..4` to **110** and `MAX1..4` to **8082**, giving
> an exact 4096 neutral. See `px4_vesc_dronecan_implementation.md` in the
> `ArvinVeiyon/Companion_Computer_Pxlabs` repo. Confirm against the live vehicle before relying
> on either set.

### Additional parameters for the RC brake channel
| Param | Value | Effect |
|-------|-------|--------|
| `RC_MAP_AUX1` | brake channel | routes the stick into `manual_control_setpoint.aux1` |
| `RC_MAP_PITCH` | 0 | frees the channel; `rover_differential` never reads `.pitch` |
| `UAVCAN_EC_FUNC5` | 407 (`RC_AUX1`) | passes aux1 through to ESC slot 5 |
| `UAVCAN_EC_MIN5` | 0 | stick at rest → 0 → brake released |
| `UAVCAN_EC_MAX5` | 8191 | stick at full → 8191 → full brake |
| `UAVCAN_EC_FAIL5` | 0 | failsafe = brake released |

Setting `UAVCAN_EC_FUNC5` non-zero is what grows the RawCommand array to 5 elements; PX4 sizes
it to the highest slot with a non-zero `UAVCAN_EC_FUNCn`. The message is broadcast, so all four
VESCs see slot 4 — shared braking, which is correct for a rover.

### VESC Tool
Set `uavcan_raw_mode` = `UAVCAN_RAW_MODE_CURRENT` so the throttle stick keeps reverse and
braking comes solely from the new axis.

---

## Build Setup & Commands

### Repository Path
```
/home/pxlabs/PXLABS_BLDC_VESC6_MK5/bldc
```

### Toolchain
GCC ARM none-eabi 9.3.1 — `/opt/gcc-arm-none-eabi-9-2020-q2-update/bin/`

### Build Procedure
```bash
cd /home/pxlabs/PXLABS_BLDC_VESC6_MK5/bldc

# Stable release:
git checkout pxlabs-release-6.06-rover-r1
# or development:
git checkout pxlabs-6.06-rover-uavcan_main

make fw_60_mk5_clean && make fw_60_mk5

# Firmware output:
# build/60_mk5/60_mk5.bin
```

> `make fw_60_mk5_clean` deletes everything in `build/60_mk5/`. If those artifacts are the only
> copy of a build you care about, save them first — the r1 binaries are always recoverable from
> the GitHub release above.

### Flash via VESC Tool
1. Open VESC Tool
2. Connect to VESC
3. **Firmware** → **Custom File**
4. Select `build/60_mk5/60_mk5.bin`
5. Click Upload

Repeat for all four controllers.

---

## Repository

| Remote | URL |
|--------|-----|
| origin | git@github.com:ArvinVeiyon/PXLABS_BLDC_VESC6_MK5.git |
| upstream | https://github.com/vedderb/bldc.git |
