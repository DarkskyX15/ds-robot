# Design

## Context

See `proposal.md` — Why. Facts that shape the approach:

- lerobot 0.4.3 `KeyboardTeleop` (`teleoperators/keyboard/teleop_keyboard.py`): pynput is skipped on Linux when `DISPLAY` is unset; `connect()` then silently does nothing; `is_connected` is `PYNPUT_AVAILABLE and listener.is_alive()`; `get_action()` raises when not connected.
- Current example flow: keyboard construction/`connect()` at lines ~365-368 (does not raise headless), unguarded `keyboard.get_action()` at line ~416 (first statement of the episode loop) — the observed crash point.
- Server endpoint: `OPENPI_SERVER_IP` constant (line ~69) + hardcoded `port=8000` at the client constructor (line ~398); the deployed server listens at `10.10.64.57:20721`.
- The example's style is module-level constants (`TASK_DESCRIPTION`, `FPS`, `NUM_EPISODES`, `EPISODE_TIME_SEC`, `OPENPI_SERVER_IP`).

## Goals / Non-Goals

**Goals:**

- The client runs end-to-end over SSH (no `DISPLAY`), with a clear notice that `^` is unavailable and `Ctrl+C` is the stop.
- Server host and port are both editable via constants; no literal port in the client call.
- Desktop behavior (`^` emergency stop, clean shutdown) unchanged.

**Non-Goals:**

- No CLI/argparse or env-var configuration (keep the file's constant style).
- No change to the IP default (stays a placeholder), no driver/ZMQ/`8_vr` example changes.
- No automatic fallback to the `^` key via other mechanisms.

## Decisions

1. **Add `OPENPI_SERVER_PORT = 20721` next to `OPENPI_SERVER_IP`** and use both constants in `WebsocketClientPolicy(...)`. Inline comment notes openpi's own default is 8000 while this deployment uses 20721. Alternatives: argparse/env (rejected — inconsistent with the file's constant style); keeping 8000 (rejected — known deployment trap).
2. **Detect keyboard availability once, right after initialization**: wrap construct+`connect()` in `try/except Exception`, then treat the keyboard as usable only if `keyboard.is_connected`. If not usable: set `keyboard = None` and print one warning line: `"⚠️ Keyboard unavailable (no DISPLAY?). '^' emergency stop disabled; use Ctrl+C to stop."` Alternatives: checking `DISPLAY` ourselves (duplicates lerobot logic); per-iteration `try/except` around `get_action()` (hides transient errors, log spam).
3. **Guard the loop**: `if keyboard is not None:` around the pressed-keys read and `^` check. All other loop logic untouched.
4. **Ctrl+C**: add `except KeyboardInterrupt:` before `except Exception` in the inference `try` (prints `"⛔ Interrupted (Ctrl+C); returning to home pose and disconnecting..."` and lets `finally` do the cleanup). Harden the `finally` block: wrap the homing calls in an inner `try/finally` so `robot.disconnect()` (torque off) always runs even if a second `Ctrl+C` interrupts the homing motion.
5. **Docs**: runbook §2.3 item 3/4 rewritten for the two constants plus a headless run note in §2.6; handover client row (`:398` port mention) and setup step (`:69`) updated to the two constants.

## Risks / Trade-offs

- [Keyboard dies mid-run after successful start] → the real `DeviceNotConnectedError` still surfaces (not hidden); acceptable.
- [Second `Ctrl+C` during the finally-homing] → homing may end early, but `disconnect()` is guaranteed by the inner `try/finally`; documented in code comment.
- [Port default `20721` differs from openpi's generic `8000`] → inline comment + runbook note; one-line edit for other servers.
- [Headless mode has no `^` stop] → one-time warning + `Ctrl+C`; runbook recommends keeping a physical power cutoff at hand during early tests.

## Migration Plan

- Example-only change; runs directly from the repo (`software/examples/...`), so deployment is just `git pull` on the robot machine — no re-copy into lerobot.

## Open Questions

- None.
