# PXLabs VESC Firmware — PX4 Rover (DroneCAN/UAVCAN)

Custom VESC 6.06 firmware for the PXLabs 4-wheel differential rover, driven by a
PX4 flight controller over DroneCAN. Four VESC 6 MK5 controllers, one per wheel.

Last updated: 2026-09-09

---

## Version Information

| Item | Value |
|------|-------|
| **Current stable release** | `v6.06.0-pxlabs-rover-r1` |
| **Release commit** | `6fc2cf17337ce6d1e494c47a5969b93acac0661e` |
| **Release date** | 2026-02-08 |
| **Current pre-release** | `v6.06.0-pxlabs-rover-r2-alpha1` (RC brake, bench-tested, **not driven**) |
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
| `pxlabs-6.06-rover-brake-rc` | RC brake feature — bench-tested 2026-09-09, **merged into dev** |
| `master` | Original upstream VESC |
| `release_6_06` | Official VESC 6.06 |

### Tags
| Tag | Commit | Description |
|-----|--------|-------------|
| `v6.06.0-pxlabs-rover-r1` | `6fc2cf17` | PXLabs stable release r1 |
| `v6.06.0-pxlabs-rover-r2-alpha1` | (dev tip) | **Pre-release.** RC brake, bench-tested on stands. Not a stable cut. |

`r1` = release 1. Every other tag in this repo is upstream vedderb/bldc.

`r2-alpha1` is a GitHub **pre-release**, not a stable release. It exists so the bench-tested brake
firmware has a citable, frozen artifact. The stable cut is `v6.06.0-pxlabs-rover-r2`, and it is
gated on the open items listed under the RC brake section below. Per PXLabs policy, release
branches and tags are frozen once created — `r2-alpha1` will never be moved or recreated.

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
| 2026-09-05 | `da94056f` | brake-rc | Rename `Motp_Config_Bldc/` -> `Motor_Config_Bldc/`, prune 44 -> 8 |
| 2026-09-05 | `3ae3373f` | brake-rc | Add `Testing_Bin/`; document the fork in `CLAUDE.md` |
| 2026-09-05 | `d513e790` | brake-rc | Record the folder rename, prune and `Testing_Bin` in this note |
| 2026-09-09 | `fa27a6b3` | brake-rc | **Correct docs: flashing this image over DroneCAN bricks the ESC** |

**The only firmware source change since 2026-02-07 is the brake commit `a75a0dbf`.** Everything
else is documentation and VESC Tool configuration data, which do not affect the built binary.

### `Testing_Bin/`
The built `60_mk5.bin` under bench test, so the companion and bench machines get the exact tested
artifact from a plain clone. Holds one binary at a time; see its `README.md`. It is retained through
the `r2-alpha1` pre-release and **is to be deleted at the stable `r2` cut**, when the tagged release
carries the flashable assets instead.

⛔ **Flash over USB only. Never over DroneCAN** — see `Testing_Bin/README.md` for the arithmetic;
the image overruns the staging area into the bootloader sector and bricks the ESC.

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

## RC Brake Channel — TESTED ON THE FLOOR, LOADED (`v6.06.0-pxlabs-rover-r2-alpha1`)

> **Status: tested on the floor under load 2026-09-09, merged to dev.** Two runs: a first
> (motor-side only) and an instrumented re-run with body-motion telemetry. **The loaded condition
> is now measured, not attested.** Headline: the brake stops the rover in **0.30–0.50 m from
> ~0.8 m/s**, at **0.69 m/s² median / 0.94 m/s² peak**. Rollback is the r1 release binary, over
> USB, per ESC.
>
> ⚠️ **The `v6.06.0-pxlabs-rover-r2-alpha1` tag annotation is wrong on this point.** It describes
> the run as a bench test on stands with the rover never driven. That was based on a report later
> retracted (see below). The tag is frozen and will not be rewritten — **this document supersedes
> its annotation.**

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

### 🔴 A hand test cannot measure this brake

`mc_interface_set_brake_current_rel()` resolves to `CONTROL_MODE_CURRENT_BRAKE`
(`mcpwm_foc.c:832`). That brake makes torque by **opposing rotation**, so its strength scales
with back-EMF and therefore with wheel speed. At hand-turning speed, 100 % brake and 10 % brake
both produce approximately nothing.

**Turning a wheel by hand and feeling no resistance is the expected result, and it measures
nothing.** The brake must be tested against a spinning wheel. Two operator observations from
2026-09-07 — "the brake applies immediately whatever the throttle is doing" and "I can still
turn the motor by hand, full stick feels no different from low stick" — are both **correct
behaviour**, not defects. The first is the intended brake-before-throttle precedence
(`canard_driver.c:746`); the second is the back-EMF scaling above.

### Test results — 2026-09-09

All four ESCs online. Operator drove the wheels under throttle and worked the brake stick.
303 s at 5 Hz: 30,090 `input_rc`, 4,337 `manual_control_setpoint`, 29,805 `esc_status` messages,
captured over DDS on the companion.

#### Test conditions: operator-attested — on the floor, loaded

**The rover was on the floor, under its own weight, as confirmed directly by the operator.**

This corrects two earlier accounts in this document's history, both wrong. The first said "rover on
stands, wheels off the ground"; that phrasing began as a suggestion made to the operator before the
run, was never confirmed and never measured, and was mistakenly recorded as observed. The second
replaced it with an inference that the wheels were free-spinning, built on five indicators. That
inference was also wrong, and its central argument was arithmetically invalid:

- **The headline claim was that 51.7 m of implied path is impossible in a room with ~2 m² of floor.**
  But the same analysis counted **49 direction reversals**, and 51.7 m across 49 reversals is
  **~1.05 m per leg**. Drive a metre, brake, reverse a metre, brake, repeat — that is not merely
  possible in a small room, it is precisely what brake testing in a small room looks like. The run
  never required 51.7 m of clear floor.
- **Cross-wheel rpm lockstep (median 0, p90 23) does not discriminate** and is withdrawn. Wheels
  decorrelate on the floor mainly in *turns*; in straight forward/reverse runs a loaded skid-steer
  also tracks closely.
- **Peak implied 1.74 m/s** is high against the ~0.9 m/s measured from a 0.25 command, but full
  stick was commanded and this vehicle's speed response is known to be non-linear.
- **Coast decay 78–85 rpm/s is ambiguous either way** — a loaded rover has more rolling resistance
  but also far more inertia.

The operator's direct account of what was physically done outranks all of it, and is also the only
account that fits without special pleading.

> **Recording gap, and the reason this went wrong at all.** The session logged only `input_rc`,
> `manual_control_setpoint` and `esc_status`. **`esc_current` was inside `esc_status` the whole time
> and was not used** — a current step at constant rpm is the clean loaded/unloaded discriminator.
> No vehicle-motion source was subscribed at all: no `/odom`, no IMU, no local position, all of
> which were available on DDS. Had any one of them been recorded, none of this ambiguity would have
> existed. **Record `esc_current`, `/odom`, `vehicle_local_position` and `sensor_combined` on every
> future run.**

### Instrumented floor run — 2026-09-09 (the vehicle-side numbers)

A second, instrumented run on the floor under load, recorded with the operator's explicit go-ahead.
115.8 s, 579 rows at 5 Hz, armed, `nav_state` 0 (Manual), no failsafe, kill switch safe throughout.
Six topics, each verified to have a non-zero baseline before scoring, after a 4 s DDS discovery
warm-up: `esc_status` (now including **`esc_current`**), `input_rc`, `manual_control_setpoint`,
`/odom`, `vehicle_local_position_v1`, `sensor_combined`. The recorder fails loudly on a silent topic
rather than letting an empty column read as "no motion".

Companion artifacts: raw `~/brake_run_20260909_floor2.csv`, recorder
`bldc_can/diag/brake_run_record.py`, analysis `bldc_can/diag/brake_run_analyse.py`, write-up
`bldc_can/evidence/brake_floor_test_20260909.md`.

#### Loaded condition — now measured, not inferred

| Evidence | Value |
|---|---|
| `esc_current`, stationary | −1.00 … +1.68 A (n=1748) — sensor noise |
| `esc_current`, moving | −12.06 … +8.18 A (n=301) |
| … driving | median **+5.16 A**, peak +8.18 A |
| … **regen (braking)** | median **−3.56 A**, peak **−12.06 A** |
| `/odom` forward velocity | peak **0.943 m/s**, 118 samples above 0.05 m/s |
| IMU accelerometer, x | −3.33 … +3.57 m/s² |

Unloaded wheels need a fraction of an amp to hold speed. This is a loaded vehicle, unambiguously.
The IMU is the witness that cannot be fooled by wheel slip — a wheel spinning on a stand produces no
body acceleration.

> ⚠️ **`/odom` drifts at standstill** — ~0.38 m of phantom travel in a minute with all four wheels at
> 0 rpm (the camera-gyro dead-reckoning term). Use it for *change during a run*, corroborated by the
> IMU. Never as a ruler.

#### Braking performance

| Metric | Value |
|---|---|
| **Braked deceleration** | **0.69 m/s² median**, 0.94 peak (n=9) |
| Coasting deceleration | 0.20 m/s² median (n=2) — ⚠️ **provisional**, see below |
| **Stopping distance from ~0.8 m/s** | **0.30 – 0.50 m** |

Worked stops, throttle-neutral gated:

| From | To | Time | Decel | Distance |
|---|---|---|---|---|
| 0.85 m/s | 0.03 m/s | 1.0 s | 0.81 m/s² | 0.44 m |
| 0.83 m/s | 0.00 m/s | 1.2 s | 0.69 m/s² | 0.50 m |
| 0.76 m/s | 0.00 m/s | 0.8 s | 0.94 m/s² | 0.30 m |

**`esc_errorcount` = 0 = NONE on all four ESCs, every sample — now under load as well as
free-spinning.** The `timeout_reset()` check has passed in both conditions.

> ⚠️ **The coast baseline is weak and the derived figures are provisional.** Coast is n=2, both
> segments only 0.4 s, and **neither ran to a stop** — the rover was still doing 0.63 m/s when each
> ended. So the brake-vs-coast ratio (~3.4×) and the extrapolation to 0.9 m/s (coast ~2.00 m, braked
> ~0.59 m, saving ~1.4 m) **must be treated as provisional**. The *braked* figures do not inherit
> this weakness — 9 runs, 3 of them to a complete stop. One clean coast-to-stop closes it.

> 🔴 **Do not compare braking against the "~0.30 m coast at ~0.9 m/s" figure.** That number is
> **distance-before-contact** from a standoff test in which the rover *hit* the obstacle — 0.345 m
> standoff, ~0.30 m consumed, 0.020 m remaining at contact. It is not a stopping distance. Read as
> one it implies ~1.35 m/s², roughly twice what the brake achieves, and would wrongly suggest the
> brake makes stopping worse. The only sound comparison is a coast-to-stop measured the same way as
> the braked runs.

### RC channel geometry — solved, not read

MAVLink to the FC was down for the whole session (no heartbeat on `tcp:5760`, before and after an
FC reboot), so no parameter could be read or written. The values below were **solved from the
logged data**, not read off the flight controller, by inverting PX4's own piecewise map
`T = (ch3 + MIN·aux1) / (1 + aux1)`:

| Param | Value | How |
|---|---|---|
| `RC3_MIN` | 1001 µs | consistent throughout |
| `RC3_TRIM` | **1487.5 µs** | solved, n=118, median 1487.0; independently confirmed by the steady hold |
| `RC3_MAX` | ≈1973 µs | solved, n=55; `aux1` saturates at +1.0000 from 1969 µs up |
| `RC3_REV` | not reversed | behavioural — bottom stop −1.0, top +1.0 |

**The trim blocker did not regress.** `RC3_TRIM` is 1487.5, not 1001, so the 50 %-brake-on-leaving-
the-bottom-stop bug is not live.

> 🔴 **A QGC RC calibration rewrites TRIM.** If `RC3_TRIM` is ever driven back to `RC3_MIN` = 1001,
> the first segment of `interpolateNXY` (`Functions.hpp:201`) becomes zero-width: −1.0 at exactly
> 1001 µs and ~0.0 at 1002 µs, a one-microsecond discontinuity that
> `output_limit_calc_single` (`mixer_module.cpp:568`) maps onto slot 5 = 4096 = **50 % brake the
> instant the stick leaves its stop.** Re-read `RC3_TRIM` after any RC calibration, always.
>
> ⛔ Do not let `rc_configuration.md` §2.1 talk anyone out of this fix. Its claim that
> `RCn_TRIM == RCn_MIN` is a harmless QGC artefact PX4 self-corrects is **throttle-only** —
> `rc_update.cpp:172` scopes the re-centring to `FUNCTION_THROTTLE`. It never applied to ch3.

Minor: full stick reads 2000 µs against a ~1973 µs max, so full brake is a **saturated command**
and the top ~27 µs of travel is dead. Harmless — PX4 clamps at ±1.0 — and ch2 already has the
same overshoot.

### Known limitations

- **No holding brake.** Regenerative braking gives ≈0 torque at standstill, so this will not hold
  the rover on a slope. The firmware already has one: `mc_interface_set_handbrake_rel()` →
  `CONTROL_MODE_HANDBRAKE`, same `val × |lo_current_min|` scaling, a one-line swap at
  `canard_driver.c:747`. A hybrid — handbrake below some ERPM, regen above — is likely the right
  end state, since regen is what cuts coast while moving.
- **Authority fades silently.** Brake authority is `brake_rel × |lo_current_min|`, and
  `lo_current_min` is the *runtime-scaled* `l_current_min`. It therefore drops as the motor heats
  or the pack nears full, with no indication. The 2026-09-09 run was one battery state at one
  temperature, and authority is not calibrated in amps. Repo values: `l_current_min` −25 A
  (LF −25.8), `l_in_current_min` −5 A, `l_abs_current_max` 35, `cc_min_current` 0.05 — the −5 A
  battery regen cap probably binds before the 25 A does. Never verified against live ESCs.
- **A second, unrelated brake is always active.** `timeout_brake_current` = 2 A at
  `timeout_msec` = 300 (`timeout.c:225-233`) applies a flat, absolute 2 A on command loss or kill
  switch. It is a different mechanism, not a weak version of this one. Anyone benchmarking coast
  is measuring both.
- **`l_max_erpm_fbrake` (300) and `l_max_erpm_fbrake_cc` (1500) are dead params on this vehicle.**
  Every use is in `mcpwm.c`, the BLDC path; `motor_type` = 2 = FOC. Do not tune them chasing brake
  strength.
- **The collision reflex does not use the brake.** It still only zeroes the setpoint; everything
  measured here is the manual ch3 brake. Wiring the reflex to command the brake is a separate,
  unmade change — now backed by 0.69 m/s² braked vs 0.20 m/s² coasting.
- **Authority measured at one battery state and one temperature.** Not calibrated in amps across
  conditions.

### Open items before the stable `r2` cut

1. **One clean coast-to-stop run.** The cheapest remaining item. Coast is n=2 with neither segment
   running to a stop, so the brake-vs-coast ratio and the ~1.4 m saving stay provisional until a
   coast-to-stop is measured the same way as the braked runs.
2. **Wire the collision reflex to the brake.** It still only zeroes the setpoint; everything measured
   here is the *manual* ch3 brake. **0.69 m/s² braked against 0.20 m/s² coasting is now the
   quantitative argument for making that change.**
3. **Explain the −12.06 A regen peak against a repo `l_in_current_min` of −5 A.** Either the live
   config differs from the repo XMLs, or that cap is per-motor rather than pack-side. Needs USB +
   VESC Tool to read the live `mcconf`.
4. **Read `uavcan_raw_mode` off a flashed ESC.** All four repo appconfs carry `CURRENT` (0) and
   nothing observed contradicts it, but no live readback exists. Needs USB + VESC Tool; the
   companion's CAN path is formally dropped (MCP2515 hat hardware-dead, overlay disabled
   2026-09-09). If lower-stick braking is ever observed, this param has been changed live.
5. **Read `UAVCAN_EC_FAIL5`.** Never read, never set. Blocked on the MAVLink link.
6. **Confirm the flashed firmware hash** on each ESC in VESC Tool. No hash was read back off any
   unit after flashing; all four are *reported* flashed, method unconfirmed for FR/FL/RR.
7. **Test the disarm and RC-loss failsafe.** Brake-off on failsafe is correct *by construction*
   (PX4 has no disarmed parameter, so `_disarmed_value` stays 0, which on the brake slot is below
   the 0.05f threshold; `NAV_RCL_ACT` = 6 disarms on RC loss) but has never been exercised.
8. **Set `RC_MAP_PITCH` = 0.** Still 3, the same channel as `RC_MAP_AUX1`. Harmless —
   `manual_control_setpoint.pitch` has zero references in `src/modules/rover_differential/` — but
   it is not the intended end state.

> ⚠️ **There is no known-good ESC left to diff against.** All four now run branch firmware. The
> single-ESC comparison that existed on 2026-09-07 is gone. Rollback is tag
> `v6.06.0-pxlabs-rover-r1`, flashed over USB, per ESC.

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

As actually set and read back on 2026-09-07, after `MAV_CMD_PREFLIGHT_STORAGE` returned
`MAV_RESULT_ACCEPTED` on a freshly rebooted FC:

| Param | Value | Effect |
|-------|-------|--------|
| `RC_MAP_AUX1` | 3 | routes the stick into `manual_control_setpoint.aux1` |
| `UAVCAN_EC_FUNC5` | 407 (`RC_AUX1`) | passes aux1 through to ESC slot 5 |
| `UAVCAN_EC_MIN5` | **1** | PX4 default, left deliberately — 0.012 % at the bottom stop, safely under the 0.05f engage threshold, so it means "off" correctly |
| `UAVCAN_EC_MAX5` | 8191 | stick at full → 8191 → full brake |
| `UAVCAN_EC_FAIL5` | *not read, not set* | intended 0 (= brake released); **unverified** |
| `RC_MAP_PITCH` | 3 | pre-existing, unchanged; intended end state is 0 |

Setting `UAVCAN_EC_FUNC5` non-zero is what grows the RawCommand array to 5 elements; PX4 sizes
it to the highest slot with a non-zero `UAVCAN_EC_FUNCn`. The message is broadcast, so all four
VESCs see slot 4 — shared braking, which is correct for a rover.

> ⚠️ **Ordering matters. Set `RC_MAP_AUX1` before `UAVCAN_EC_FUNC5`.** With slot 5 assigned while
> AUX1 is still unmapped, `aux1` reads 0, which maps to mid-scale — **50 % brake demand on the bus.**

> ⛔ **Do not copy the motor-slot convention onto the brake slot.** `UAVCAN_EC_MIN1..4` = 110 and
> `MAX1..4` = 8082 exist so the four *bipolar* motor slots get an exact 4096 neutral and dodge the
> VESC's `raw < 100` disarm guard. The brake slot is **unipolar** and needs its minimum to mean
> OFF. The defaults (1 / 8191) are correct here. 110 would also work; 4096-centred values would not.

> ⚠️ **`ros2_ws/tools/set_param.py` cannot write INT32 params, and fails silently.** It always
> sends `MAV_PARAM_TYPE_REAL32`; `mavlink_parameters.cpp:129-131` refuses the type mismatch, logs
> "param types mismatch", and writes nothing. PX4 does `param_set` on the raw 4 bytes, so an INT32
> must be sent as the integer's *bit pattern* in the float field. `RC_MAP_AUX1` and
> `UAVCAN_EC_FUNC5` are both INT32 and need a separate writer — the companion has
> `bldc_can/diag/set_param_int.py`.

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

⛔ **USB only, one controller at a time.** VESC Tool's CAN-forward cannot reach these ESCs in
`CAN_MODE_UAVCAN`, and the DroneCAN firmware-update path **bricks them** (384 KB staging area,
~512 KB image, no bounds check, bootloader sector never erased — `flash_helper.c:181` → `:120`).

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
