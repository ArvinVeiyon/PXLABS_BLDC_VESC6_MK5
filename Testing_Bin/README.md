# Testing_Bin

Pre-built firmware for **bench testing only**. Not a release.

| | |
|---|---|
| File | `60_mk5.bin` |
| Size | 524,280 bytes |
| SHA256 | `b971e9a78aada70cfeae91cb120f1e6aa4d3ba7e5641cd311137a03daf644352` |
| Built from | `a75a0dbf` — *Add RC brake channel via UAVCAN RawCommand spare slot* |
| Branch | `pxlabs-6.06-rover-brake-rc` |
| Target | 60_mk5 (VESC 6.0 MK5) |
| Toolchain | ARM GCC 9.3.1 (`9-2020-q2-update`), zero warnings |

This binary exists so the companion computer and the bench machine can get the exact tested artifact
with a plain `git clone`, without needing a GitHub release or a hand-copied file.

## Before flashing

Read **[`../PXLABS_RC_BRAKE_TESTING.md`](../PXLABS_RC_BRAKE_TESTING.md)**. There is a blocker in the
RC calibration (`RC3_TRIM == RC3_MIN`) that must be fixed first, or lifting the brake stick off its
bottom stop commands ~50 % brake instantly.

Wheels off the ground for all ten bench steps.

## Verify before you flash

```bash
sha256sum 60_mk5.bin
# b971e9a78aada70cfeae91cb120f1e6aa4d3ba7e5641cd311137a03daf644352
```

After flashing, VESC Tool must report firmware hash **`a75a0dbf`**. If it shows `05deb3e8` you have
the superseded 2026-09-05 06:22 build, which was compiled before the commit existed and misreports
its version — discard it.

## Flashing

VESC Tool's CAN-forward **cannot reach these ESCs** — they run `can_mode = CAN_MODE_UAVCAN`, and in
that mode the firmware discards every VESC-protocol frame (`comm/comm_can.c`). Scan CAN will find
nothing.

Use either:

- **USB**, one VESC at a time — simplest, and what the test procedure assumes.
- **DroneCAN** `file.BeginFirmwareUpdate` from the companion. Serve this `.bin` raw; the firmware
  writes its own size+CRC header. Node IDs are 10 (front right), 11 (front left), 12 (rear right),
  13 (rear left).

## Rollback

Flash the r1 release binary from
<https://github.com/ArvinVeiyon/PXLABS_BLDC_VESC6_MK5/releases/tag/v6.06.0-pxlabs-rover-r1>.

## Housekeeping

This folder holds **one** binary — whatever is currently under test. Replace it when the firmware
changes rather than accumulating dated copies; git history keeps the old ones. Delete the folder once
the feature merges and a proper tagged release exists.
