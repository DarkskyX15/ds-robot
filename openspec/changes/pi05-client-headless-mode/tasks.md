# Tasks

## 1. Example: endpoint constants and headless behavior

- [x] 1.1 In `software/examples/9_pi05_inference_dualarm.py`, add `OPENPI_SERVER_PORT = 20721` next to `OPENPI_SERVER_IP` (comment: openpi default is 8000, this deployment uses 20721) and use both constants in the `WebsocketClientPolicy(...)` call. Verify: `grep -n "port=8000"` has no hits; the constructor references `OPENPI_SERVER_IP` and `OPENPI_SERVER_PORT`; `python -m py_compile` passes.
- [x] 1.2 Make the keyboard optional: wrap keyboard construct/`connect()` in `try/except Exception`; treat it as usable only if `keyboard.is_connected`, otherwise set `keyboard = None` and print the one-time warning (`⚠️ Keyboard unavailable (no DISPLAY?). '^' emergency stop disabled; use Ctrl+C to stop.`). Guard the loop's pressed-keys read with `if keyboard is not None:`. Verify: `python -m py_compile` passes; grep shows the guard and the warning string (runtime behavior verified in task 3.1).
- [x] 1.3 Handle `Ctrl+C`: add `except KeyboardInterrupt:` (before `except Exception`) in the inference `try`, printing a clear interruption message; harden `finally` with an inner `try/finally` so `robot.disconnect()` always runs even if homing is interrupted. Verify: `python -m py_compile` passes; code review confirms the inner `try/finally` structure (runtime behavior verified in task 3.1).

## 2. Documentation

- [x] 2.1 Update `dev-docs/pi05_deployment_runbook.md`: §2.3 items for the example become "set `OPENPI_SERVER_IP` and `OPENPI_SERVER_PORT`" (no more "line 398 hardcoded" instruction); add a short headless-run note (SSH → no `^`, use `Ctrl+C`; keep power cutoff handy) near §2.6. Verify: grep shows `OPENPI_SERVER_PORT` and the headless note; no instruction referencing a hardcoded port edit remains.
- [x] 2.2 Update `dev-docs/pi05_async_inference_handover.md`: the client row (currently "`:398` `WebsocketClientPolicy(host, port=8000)`") and the setup step ("修改 `9_pi05_inference_dualarm.py:69` 的 `OPENPI_SERVER_IP`") now reference both constants (`OPENPI_SERVER_IP`/`OPENPI_SERVER_PORT`). Verify: grep shows both constants referenced in both places.

## 3. On-robot verification (after `git pull` on the robot machine)

- [ ] 3.1 SSH (no DISPLAY) headless run: start the example; confirm the keyboard-unavailable warning prints once, the loop proceeds into inference (observations/actions flow), and pressing `Ctrl+C` prints the interruption message, homes the arms, disconnects the robot, and shows no `KeyboardInterrupt` traceback. Verify: observed console output matches; robot ends free (torque off).
- [ ] 3.2 Desktop session run: start the example with a usable keyboard; confirm no warning is printed, `^` still stops the episode, and the arms home and the robot disconnects as before. Verify: observed console output and motion match previous behavior.

## Workflow follow-up

- Archive the change (via the OpenSpec archive workflow) once on-robot verification is complete and the branch is reviewed.
