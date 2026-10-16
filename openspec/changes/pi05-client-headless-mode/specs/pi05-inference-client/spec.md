# Spec Delta

## Purpose

Defines the behavior contract of the dual-arm pi0.5 inference client example (`9_pi05_inference_dualarm.py`): how the openpi server endpoint is configured, how the episode loop behaves with and without keyboard input, and how emergency stop works in both keyboard and headless operation.

## ADDED Requirements

### Requirement: Server endpoint is configured by editable constants
The client SHALL construct the policy client from the module constants `OPENPI_SERVER_IP` and `OPENPI_SERVER_PORT`. The policy client call SHALL NOT contain a literal port value, so pointing the client at a different server requires editing only these constants.

#### Scenario: Port constant takes effect
- **WHEN** `OPENPI_SERVER_PORT` is set to the port the server actually listens on and the script is started
- **THEN** the client connects to that port (connection succeeds without waiting loops), and no connection is attempted against any other port

### Requirement: Headless operation degrades gracefully
When keyboard input is unavailable (e.g., Linux session without `DISPLAY` so pynput is disabled), the client SHALL continue the inference loop without key-based stop checks and SHALL print a single clear warning stating that keyboard input is unavailable and that emergency stop is `Ctrl+C`. When keyboard input is available, behavior SHALL remain unchanged.

#### Scenario: Start over SSH without DISPLAY
- **WHEN** the client is started in a session without `DISPLAY`
- **THEN** it prints the keyboard-unavailable warning once and proceeds into the inference loop (observations sent, actions executed) instead of raising `DeviceNotConnectedError`

#### Scenario: Start on a desktop session
- **WHEN** the client is started with a usable keyboard (e.g., desktop session)
- **THEN** no keyboard-unavailable warning is printed and the `^` key check is active every loop iteration

### Requirement: Emergency stop works in both operating modes
The client SHALL always terminate the inference loop on its emergency stop and SHALL then run its cleanup (home both arms, disconnect the robot). In keyboard mode the `^` key SHALL stop inference as before; in headless mode `Ctrl+C` SHALL stop inference with a clear message and exit without an unhandled-exception traceback.

#### Scenario: ^ key in keyboard mode
- **WHEN** `^` is pressed during an episode
- **THEN** the inference loop stops, the arms are homed, and the robot is disconnected

#### Scenario: Ctrl+C in headless mode
- **WHEN** `Ctrl+C` is pressed during an episode
- **THEN** a clear interruption message is printed, the arms are homed, the robot is disconnected, and no traceback for `KeyboardInterrupt` is shown
