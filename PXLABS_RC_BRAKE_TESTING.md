# RC Brake — Bench Test Methodology

**Status: UNTESTED. Not merged.** This document is the acceptance procedure for the RC brake
feature on branch `pxlabs-6.06-rover-brake-rc`. Nothing on this branch has been run on hardware.

| | |
|---|---|
| Branch | `pxlabs-6.06-rover-brake-rc` |
| Commit under test | `a75a0dbf` — *Add RC brake channel via UAVCAN RawCommand spare slot* |
| Hardware target | 60_mk5 (VESC 6.0 MK5), ×4 (one per wheel) |
| Firmware artifact | `60_mk5.bin` (524,280 bytes) |
| Base | `pxlabs-6.06-rover-uavcan_main` @ `3b259f89` |
| Rollback | release `v6.06.0-pxlabs-rover-r1` |

Verify before you start that VESC Tool reports firmware hash **`a75a0dbf`** — the pre-release build
from 2026-09-05 06:22 embedded `05deb3e8` (its parent) and is not this feature.

---

## What is being tested

Brake demand travels in **slot index 4** of the `esc.RawCommand` array PX4 already broadcasts to all
four VESCs. No new DSDL, no new message, no PX4 source change — PX4 side is parameters only.

`libcanard/canard_driver.c`, +46/−18, one file:

- Reads `cmd.cmd.data[4]`, guarded on `cmd.cmd.len > 4`, clamped to `[0, 8191]` → `brake_rel` in `[0,1]`.
- If `brake_rel > 0.05`, calls `mc_interface_set_brake_current_rel()`; **else** runs the existing
  `uavcan_raw_mode` switch unchanged.
- The `if/else` is deliberate. An early `return` here would skip `timeout_reset()` further down and
  trip the command-timeout watchdog mid-brake. **Step 10 is the check for this.**

**Failsafe property:** slot value `0` means brake released. PX4 also sends `0` when disarmed
(`mixer_module.hpp` hardcodes the disarmed value; no `UAVCAN_EC_DIS<n>` param exists) and on RC loss
(`FunctionManualRC` emits `NAN` → resolves to the disarmed value). Brake-off is the failsafe by
construction, and the stick's resting position is the bottom stop = 0.

---

## Prerequisites

### 1. BLOCKER — fix the brake channel trim first

Nothing below will behave correctly until this is done.

PX4 normalises each RC channel as
`interpolateNXY(value, {min, trim, max}, {-1, 0, +1})` (`rc_update.cpp:446`).

Channel 3 currently has **`RC3_MIN=1001`, `RC3_TRIM=1001`, `RC3_MAX=1965`** — trim equals min. The
first segment is zero-width, so the curve jumps from −1.0 at the bottom stop to 0.0 one microsecond
above it: **lifting the stick off the stop instantly commands ~50 % brake.**

PX4 auto-repairs this only for the channel mapped to *throttle* (`rc_update.cpp:170-186`), which is
why ch2 behaves and ch3 does not.

**Fix:** re-run RC calibration, then set `RC3_TRIM ≈ 1483` — i.e. `(RC3_MIN + RC3_MAX) / 2`.

### 2. Confirm the channel number

Ch3 is *assumed* to be the TX16S non-centering left stick, from `RC_MAP_PITCH=3` and the `TRIM==MIN`
signature. **Confirm on the QGC Radio page** (or `/fmu/out/input_rc`) before setting anything. If it
is a different channel, substitute it everywhere below.

### 3. PX4 parameters — parameters only, no rebuild, no FMU reflash

| Param | Value | Why |
|---|---|---|
| `RC_MAP_AUX1` | brake channel (likely `3`) | routes the stick to `manual_control_setpoint.aux1` |
| `RC_MAP_PITCH` | `0` | frees the channel; `rover_differential` never reads `.pitch` |
| `UAVCAN_EC_FUNC5` | `407` (RC_AUX1) | passes aux1 to ESC slot 5 → array index 4 |
| `UAVCAN_EC_MIN5` | `0` | stick bottom → no brake |
| `UAVCAN_EC_MAX5` | `8191` | stick top → full brake |
| `UAVCAN_EC_FAIL5` | `0` | already 0 — failsafe is brake released |
| `RC3_REV` | `1` | push **up** = brake |

Setting `FUNC5` non-zero is what grows `msg.cmd` to five elements —
`src/drivers/uavcan/actuators/esc.cpp:110-121` sizes the array to the highest slot with a non-zero
`UAVCAN_EC_FUNCn`. It is a `broadcast()`, so all four VESCs receive slot 4. Shared braking is correct
for a rover.

### 4. VESC configuration

On **all four** VESCs, via VESC Tool over USB:

- `uavcan_raw_mode` = **`UAVCAN_RAW_MODE_CURRENT`** — so the throttle stick keeps reverse and braking
  comes solely from the new axis.
- Leave `can_mode` = `UAVCAN` and `can_baud_rate` = 1M as they are.

### 5. Flashing

Flash `60_mk5.bin` to all four VESCs, then power-cycle and confirm VESC Tool shows commit `a75a0dbf`.

> VESC Tool's CAN-forward will **not** reach these ESCs — in `CAN_MODE_UAVCAN` the firmware discards
> all VESC-protocol frames (`comm/comm_can.c:1346`). Flash each VESC over **USB**, or use DroneCAN
> `file.BeginFirmwareUpdate` from the companion.

---

## Bench procedure

**Wheels off the ground for every step.** Run in order and **stop at the first failure** — later
steps assume earlier ones passed.

| # | Step | Action | Pass criteria |
|---|---|---|---|
| 1 | Calibration sanity | QGC Radio page, sweep brake stick bottom→top | Bar moves smoothly, **no jump off the bottom stop**. Confirms the `RC3_TRIM` fix. |
| 2 | Plumbing, disarmed | Inspect the RawCommand array | `msg.cmd` has **5** elements; slot 4 reads **0** at rest |
| 3 | No regression | Arm. Brake stick at bottom, throttle forward | Wheels spin normally, exactly as before this change |
| 4 | Brake applies | Armed, throttle still forward, push brake stick up | Wheels stop; `esc_current` shows brake current. Release → wheels resume |
| 5 | Proportional | Half stick vs full stick | Half gives noticeably less brake current than full |
| 6 | Brake beats throttle | Command both simultaneously | Brake wins. Validates the if/else ordering |
| 7 | Reverse | Repeat step 4 with throttle in reverse | Brake applies identically |
| 8 | Disarmed safety | Disarm while holding brake stick up | 0 rpm, 0.00 A, no brake current commanded |
| 9 | RC loss | Power off the transmitter while armed | Slot 4 → 0, brake releases, **no runaway** |
| 10 | Watchdog | Check across all of the above | `esc_status.esc_errorcount` stays **0** |

Step 10 is the specific check for a missed `timeout_reset()`. A non-zero error count means the
command-timeout watchdog fired — treat as a failure even if steps 1–9 looked fine.

---

## Recording results

| Step | Result | Notes |
|---|---|---|
| 1 | ☐ pass ☐ fail | |
| 2 | ☐ pass ☐ fail | |
| 3 | ☐ pass ☐ fail | |
| 4 | ☐ pass ☐ fail | |
| 5 | ☐ pass ☐ fail | |
| 6 | ☐ pass ☐ fail | |
| 7 | ☐ pass ☐ fail | |
| 8 | ☐ pass ☐ fail | |
| 9 | ☐ pass ☐ fail | |
| 10 | ☐ pass ☐ fail | |

Tested by: ______________  Date: ______________  Vehicle: ______________

---

## If it misbehaves

Flash the r1 release binary from
<https://github.com/ArvinVeiyon/PXLABS_BLDC_VESC6_MK5/releases/tag/v6.06.0-pxlabs-rover-r1>
to all four VESCs. That is the known-good rollback target.

The PX4 parameter changes are independent — reverting `UAVCAN_EC_FUNC5` to `0` shrinks the array back
to four elements and restores the previous behaviour without touching firmware.

---

## Merging

Only after **all ten steps pass**:

```bash
git checkout pxlabs-6.06-rover-uavcan_main
git merge pxlabs-6.06-rover-brake-rc      # fast-forward
```

Do not merge into, or tag from, `pxlabs-release-6.06-rover-r1` — release branches and tags are frozen.
