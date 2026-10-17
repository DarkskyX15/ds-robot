# Proposal

## Why

The fork driver (`software/src/robots/xlerobot`) was taken from `xuweiwu/XLeRobot@pi05_dual_arm`, whose bus layout is **opposite** to the shipped XLeRobot wiring (seller JoyandAI / upstream Vector-Wangel):

- Fork assumption: `bus_left_base` = left arm + **base**; `bus_right_head` = right arm + **head**.
- Shipped robot: `bus1` (`/dev/xlerobot_port1`) = left arm + **head**; `bus2` (`/dev/xlerobot_port2`) = right arm + **base**.

Running the fork driver unchanged against the real robot would send base velocity commands to head servo motors and position commands to wheels. The robot cannot be operated safely, and pi0.5 on-hardware verification is blocked. The seller repository (`.ref/XLeRobot-custom`, bus1/bus2 motor tables and `port1/port2` fields) is the authoritative reference for the shipped layout.

## What Changes

- Re-key the driver's two motor buses to the shipped layout, aligning naming with seller/upstream code: `bus1` = left arm + head, `bus2` = right arm + base. Config fields `port_left_base`/`port_right_head` become `port1`/`port2` (defaults `/dev/xlerobot_port1`, `/dev/xlerobot_port2`).
- **BREAKING** (internal contract): head motor keys renamed `head_pan`/`head_tilt` -> `head_motor_1`/`head_motor_2` (matches seller, upstream, and this repo's 2-wheels variant; enables reuse of seller calibration data). Affected: observation/action feature keys, calibration keys, ZMQ client feature keys.
- Regroup all bus-level operations to the new grouping: calibration split/restore/write, `configure`, `get_observation`, `send_action`, `stop_base`, `setup_motors`, disconnect.
- Update the two dual-arm examples that read buses directly: `software/examples/8_vr_teleop_with_dataset_recording_dualarm.py`, `software/examples/9_pi05_inference_dualarm.py`.
- Preserve all fork pi0.5 features unchanged: `get_camera_observation()`, patched OpenCV/RealSense `async_read` camera drivers, `quadratic_spline_via_ipol` interpolator, `use_degrees=True` behavior.

## Capabilities

### New Capabilities

- `xlerobot-driver`: hardware mapping contract of the XLeRobot driver — bus grouping and port roles, motor identity/naming, calibration organization, observation/action routing, and the preserved pi0.5 remote-inference support surface (camera observation helper, bus-direct reads).

### Modified Capabilities

<!-- openspec/specs/ is empty; no existing capability paths -->

## Impact

- Code: `software/src/robots/xlerobot/xlerobot.py` (~60 bus references, head keys), `software/src/robots/xlerobot/config_xlerobot.py` (port fields), `software/src/robots/xlerobot/xlerobot_client.py` (feature keys, comments), `software/examples/8_vr_teleop_with_dataset_recording_dualarm.py`, `software/examples/9_pi05_inference_dualarm.py`.
- Not affected: camera driver patches, `xlerobot_2wheels`, `simulation/`, `web_control/`, openpi server (`xuweiwu/openpi`).
- Hardware/ops: on-robot verification requires both buses powered with motors; seller-format calibration may be reused after the head-key rename, or a fresh calibration run is required.
- Documentation: resolves risk item #1 of `dev-docs/pi05_async_inference_handover.md` (wiring difference).
