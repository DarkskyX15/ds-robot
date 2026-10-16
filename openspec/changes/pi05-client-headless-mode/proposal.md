# Proposal

## Why

The dual-arm pi0.5 inference client (`software/examples/9_pi05_inference_dualarm.py`) aborts in the very first loop iteration when run from an SSH session: with no `DISPLAY`, lerobot's `KeyboardTeleop` is intentionally disabled (pynput skipped), `connect()` fails silently, and `keyboard.get_action()` raises `DeviceNotConnectedError` — so zero inferences execute. Robot bring-up and testing happen over SSH, so the client must run headless. Separately, the openpi server port is hardcoded (`port=8000`) while the deployed server listens on `20721`; the endpoint must be editable like `OPENPI_SERVER_IP`, as already proven to be a deployment trap.

## What Changes

- Add module constant `OPENPI_SERVER_PORT` (default `20721`) next to `OPENPI_SERVER_IP`; the policy client SHALL be constructed from both constants (no literal port in the call).
- Headless degradation: after `keyboard.connect()`, detect availability (`keyboard.is_connected`); if unavailable, print a clear warning once and skip the per-iteration `^` key check instead of crashing.
- Emergency stop in headless mode: `Ctrl+C` SHALL terminate the inference loop with a clear message, and the existing cleanup SHALL still home the arms and disconnect.
- With a usable keyboard (desktop session), the `^` emergency stop behavior is unchanged.
- Update `dev-docs/pi05_deployment_runbook.md` (port constant, headless run instructions) and the port reference in `dev-docs/pi05_async_inference_handover.md`.

## Capabilities

### New Capabilities

- `pi05-inference-client`: behavior contract of the dual-arm pi0.5 inference client example — server endpoint configuration, episode loop and emergency-stop handling in both keyboard and headless operation.

### Modified Capabilities

<!-- no existing capability specs yet in openspec/specs/ -->

## Impact

- Code: `software/examples/9_pi05_inference_dualarm.py` only (no driver, ZMQ, or server changes).
- Docs: `dev-docs/pi05_deployment_runbook.md`, `dev-docs/pi05_async_inference_handover.md`.
- Operations: enables headless (SSH) inference runs; desktop runs unchanged.
