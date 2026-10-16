# Design

## Context

See `proposal.md` — Why. Current state and constraints that shape the approach:

- The driver under `software/src/robots/xlerobot/` was copied from `xuweiwu/XLeRobot@pi05_dual_arm` and assumes `bus_left_base` = left arm + base, `bus_right_head` = right arm + head (opposite of the shipped wiring).
- The seller repository `.ref/XLeRobot-custom` (= upstream Vector-Wangel structure) defines the shipped layout: `bus1`/`port1` = left arm + head, `bus2`/`port2` = right arm + base; head motors named `head_motor_1`/`head_motor_2`.
- Deployment target is the direct pi0.5 path: a robot-side machine runs lerobot with these driver files copied in; `9_pi05_inference_dualarm.py` reads the two arm buses directly (`robot.bus_*`) for its 50 Hz P-control loop; `8_vr_teleop_with_dataset_recording_dualarm.py` uses the same handles for recording.
- The deployed machine already defines stable udev symlinks `/dev/xlerobot_port1` / `/dev/xlerobot_port2`.
- Calibration is stored per motor name in a per-robot JSON file; motor IDs are per-bus (1–6 each arm, head 7–8, wheels 7–9).

## Goals / Non-Goals

**Goals:**

- Driver bus structure and naming identical to the seller/upstream reference, so the shipped robot works and future diffs against seller/upstream stay small.
- Keep every fork pi0.5 feature intact: `get_camera_observation()`, direct per-bus motor handles for the examples, `use_degrees`, interpolator usage (no changes required in `utils/`, camera files).
- Calibration workflow correct for the shipped layout, incl. reuse of a seller-format calibration file.

**Non-Goals:**

- Physical rewiring; changes to `xlerobot_2wheels`, `simulation/`, `web_control/`, camera driver patches, or the openpi server.
- Changing the policy/dataset contract (12 arm dims, order, degrees, delta mask).
- Adding head/base control to the policy path.

## Decisions

1. **Adopt seller/upstream naming (`bus1`/`bus2`, `port1`/`port2`, `head_motor_1`/`head_motor_2`) instead of descriptive renames.**
   - Rationale: 1:1 correspondence with the reference implementation (easy review/diff/merge with seller and upstream), matches the machine's udev names (`/dev/xlerobot_port1|2`), and matches the repository's own two-wheel variant which already uses `head_motor_*`.
   - Alternatives considered: descriptive names (`bus_left_head`/`bus_right_base`) — rejected: larger divergence from both upstreams and no practical gain; keeping old names with swapped contents — rejected: actively misleading and dangerous.

2. **Update examples mechanically to the new handles** (`robot.bus1` / `robot.bus2`), keeping their logic unchanged (14 references each in examples 8 and 9).

3. **Preserve observation key order; only rename head keys.** Order stays left arm (6) → right arm (6) → head (2) → base (3) to avoid perturbing existing recording/policy pipelines; `head_pan.pos`/`head_tilt.pos` become `head_motor_1.pos`/`head_motor_2.pos` in both `xlerobot.py` and `xlerobot_client.py`.

4. **Calibration mirrors the seller routine**: bus1 calibrates left arm + head (8 position servos, half-turn homing + range recording); bus2 calibrates right arm (range recording) and wheels (full-turn range 0–4095, no range recording), preserving the existing prompt/UX style.

5. **Config exposes `port1`/`port2` with defaults `/dev/xlerobot_port1` / `/dev/xlerobot_port2`.** Defaults match the deployed machine; other machines edit config values only.

6. **No changes to `software/src/cameras/**`, `software/src/utils/quadratic_spline_via_ipol.py`, `software/src/model/**`.** They are bus-grouping agnostic.

## Risks / Trade-offs

- [Missed references to old names in fork-only code (driver, client, examples)] → Add a repo-wide grep gate as a task: no `bus_left_base|bus_right_head|head_pan|head_tilt` may remain outside `.ref/`.
- [Actual motor IDs on the robot may differ from the reference table (e.g., re-ID'd motors)] → Hardware verification task reads one motor per group per bus before any torque is enabled; `setup_motors` flow retained for re-ID.
- [udev symlinks may be swapped on a given machine] → Verification task confirms left arm + head respond on port1 and right arm + base on port2 before calibration.
- [Existing calibration files keyed with fork naming cannot be reused directly] → Acceptable: either restore a seller-format file, remap head keys, or run fresh calibration (documented migration step).
- [Renaming feature keys breaks any external script relying on `head_pan.pos`] → Repo-internal scope only; policy contract unaffected (arm-only), noted in migration.
- [ZMQ client path (`xlerobot_client.py`) is not exercised by the pi0.5 deployment] → Keep it consistent anyway (key rename); its hardware retest stays optional.

## Migration Plan

1. Land the change on `feat/pi05-async-inference` with static checks (imports, grep gate).
2. On the robot machine: pull the repo, re-copy the driver files into the lerobot install (existing setup flow), set `port1`/`port2` if needed.
3. Power motors, run the bus-role verification (head vs wheels safety check), then restore-or-recalibrate.
4. Run example 9 (pi0.5) end-to-end; example 8 remains smoke-tested.
5. Rollback: revert the commit and re-copy the previous driver files into the lerobot install; no persistent state changed besides calibration files.

## Open Questions

- Whether/how to share this fix back to `xuweiwu/XLeRobot` or upstream (a PR decision that can be made after hardware verification; does not affect this change's scope).
