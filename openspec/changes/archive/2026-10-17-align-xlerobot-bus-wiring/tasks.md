# Tasks

## 1. Driver & Config: bus regroup and head rename

- [x] 1.1 In `software/src/robots/xlerobot/config_xlerobot.py`, replace `port_left_base`/`port_right_head` with `port1`/`port2` (defaults `/dev/xlerobot_port1`, `/dev/xlerobot_port2`). Verify: `grep -rn "port_left_base\|port_right_head" software/` returns no hits.
- [x] 1.2 In `software/src/robots/xlerobot/xlerobot.py`, rebuild bus construction: `bus1` = left arm (IDs 1–6) + `head_motor_1`/`head_motor_2` (IDs 7–8); `bus2` = right arm (IDs 1–6) + wheels (IDs 7–9); update calibration split dicts and the `left_arm_motors`/`right_arm_motors`/`head_motors`/`base_motors` group lists. Verify: in the robot env, `XLerobot(XLerobotConfig(id="xlerobot", use_degrees=True))` constructs and `list(robot.bus1.motors)` / `list(robot.bus2.motors)` match the reference tables.
- [x] 1.3 Regroup all remaining bus-level methods (`is_connected`, `connect` calibration restore, `is_calibrated`, `calibrate` incl. per-bus homing/range flow with wheels as full-turn on bus2, `configure` operating modes/coefficients, `setup_motors`, `get_observation`, `send_action`, `stop_base`, `disconnect`) to the new grouping, mirroring `.ref/XLeRobot-custom/software/src/robots/xlerobot/xlerobot.py`. Verify: repo-wide `grep -rn "bus_left_base\|bus_right_head" software/` returns no hits, and `python -m py_compile` passes for the driver.
- [x] 1.4 Rename head observation keys `head_pan.pos`/`head_tilt.pos` → `head_motor_1.pos`/`head_motor_2.pos` in `xlerobot.py` `_state_ft` and in `xlerobot_client.py` `_state_ft` (plus stale comments). Verify: repo-wide `grep -rn "head_pan\|head_tilt" software/` returns no hits; `observation_features` contains the two new keys in unchanged order (arms → head → base).
- [x] 1.5 Update `dev-docs/pi05_async_inference_handover.md`: mark wiring risk #1 resolved and update any bus/port naming referenced there. Verify: grep shows the new naming (`bus1`/`bus2`, `port1`/`port2`) and no old names in `dev-docs/`.

## 2. Examples

- [x] 2.1 Update `software/examples/9_pi05_inference_dualarm.py` bus handles to `robot.bus1` / `robot.bus2` (14 references; left arm + head on bus1, right arm on bus2). Verify: `grep -n "bus_left_base\|bus_right_head"` returns no hits and `python -m py_compile` passes.
- [x] 2.2 Update `software/examples/8_vr_teleop_with_dataset_recording_dualarm.py` the same way (14 references) and fix the init print of bus motor tables. Verify: no old-name hits and `python -m py_compile` passes.

## 3. On-robot hardware verification (motors powered, wheels off ground)

- [x] 3.1 Deploy: pull `feat/pi05-async-inference` on the robot machine, re-copy the driver files into the lerobot install, confirm `port1`/`port2` values point at `/dev/xlerobot_port1` / `/dev/xlerobot_port2`. Verify: robot-env import test passes and a logger output during `connect()` shows both buses opening.
- [x] 3.2 Pre-movement bus-role check with torque not yet enabled: read `Present_Position` of `left_arm_shoulder_pan` + `head_motor_1` via bus1 and `right_arm_shoulder_pan` + one wheel via bus2; confirm all expected IDs respond and cross-bus reads fail. Verify: read values are sane (not errors) and no motor moves during the check.
- [x] 3.3 Calibration: either restore a seller-format calibration file (keys now match) or run a fresh calibration (bus1: left arm + head; bus2: right arm + wheels with full-range wheels). Verify: calibration file contains all 17 motors, `connect()` restores without warnings, `is_calibrated` is true.
- [x] 3.4 Routing safety check per spec: send a small head position command and confirm only head servos move and wheels stay still; send a small base velocity and confirm only wheels move and head stays still; send a small arm target and confirm only that arm moves. Verify: observed motion matches expectations with no cross-group movement.

## 4. Integration acceptance

- [x] 4.1 Run `9_pi05_inference_dualarm.py` end-to-end against the openpi server (`10.10.64.57:20721`): both arms interpolate to `FULL_START_POS`, observation/action chunks stream at the expected cadence, `^` emergency stop works. Verify: at least one full episode chunk cycle completes and stop works.
- [x] 4.2 Smoke-test `8_vr_teleop_with_dataset_recording_dualarm.py` far enough to confirm it reads both arms and connects cameras (VR headset optional), then exit cleanly. Verify: no attribute/connection errors during init and a short manual read.

## Workflow follow-up

- Archive the change (via the OpenSpec archive workflow) once on-robot verification is complete and the branch is reviewed.
