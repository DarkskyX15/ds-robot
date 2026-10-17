# Spec Delta

## Purpose

Defines the hardware mapping contract of the XLeRobot (three-wheel bimanual) driver in this repository: how the two serial buses group arm, head and base motors, motor identity and naming, calibration organization, and the observation/action routing relied on by teleoperation, data collection and pi0.5 remote inference.

## ADDED Requirements

### Requirement: Bus grouping matches the shipped wiring
The driver SHALL group motors so that bus1 (port1) carries the **left arm and head** motors and bus2 (port2) carries the **right arm and base** motors, matching the shipped robot and the seller/upstream reference implementation. Every read and write SHALL be routed to the bus that physically hosts the addressed motor group.

#### Scenario: Observation reads come from the correct bus
- **WHEN** the robot is powered and connected with the shipped wiring
- **THEN** left arm and head positions are read over port1, and right arm positions and wheel velocities are read over port2, with values matching the physical motors

#### Scenario: Head commands never drive wheels
- **WHEN** an action commands head motor positions
- **THEN** only the head servos move; wheel motors on the same physical daisy chain remain unaffected

#### Scenario: Base commands never drive head servos
- **WHEN** an action commands base velocities (x/y/theta)
- **THEN** only the wheels move; the head servos on the same physical daisy chain remain unaffected

### Requirement: Motor identity and naming follow the shipped convention
The driver SHALL use per-bus motor IDs 1–6 for each arm, IDs 7–8 for the head motors on bus1, and IDs 7–9 for the wheels on bus2. Head motors SHALL be named `head_motor_1` / `head_motor_2` (replacing the fork's `head_pan` / `head_tilt`) so they match the seller, upstream and the repository's two-wheel variant.

#### Scenario: Observation feature keys expose head motors under the new names
- **WHEN** the driver's observation features are inspected
- **THEN** they contain `head_motor_1.pos` and `head_motor_2.pos` (not `head_pan.pos` / `head_tilt.pos`), alongside the unchanged arm and base keys

#### Scenario: Seller-format calibration can be reused
- **WHEN** a calibration file produced by the seller/upstream software is present for the robot
- **THEN** the driver accepts it (keys and motor IDs match) and restores it to the correct motors on each bus

### Requirement: Port configuration exposes the two buses
The robot configuration SHALL expose exactly two bus port fields (`port1`, `port2`), defaulting to `/dev/xlerobot_port1` and `/dev/xlerobot_port2`, so the two physical buses can be pointed at the machine's actual device nodes without editing driver code.

#### Scenario: Custom device paths
- **WHEN** the configured port values are set to the device nodes present on the machine
- **THEN** connecting the robot opens both buses successfully

### Requirement: Calibration is organized per physical bus
The calibration workflow SHALL split motors by the bus that physically hosts them (left arm + head on port1; right arm + base on port2), run homing/range recording per bus, and write each group's calibration to its bus. Restoring calibration from file SHALL distribute entries to the correct bus.

#### Scenario: Fresh calibration covers all motors
- **WHEN** a fresh calibration is performed
- **THEN** calibration entries are produced for all 17 motors (6+2 on port1, 6+3 on port2) and saved, and a subsequent connect restores them to the matching motors

### Requirement: pi0.5 remote-inference support is preserved
The driver SHALL retain the fork's pi0.5 support surface: the camera observation helper returning `head`, `left_wrist` and `right_wrist` frames with latest-frame semantics, direct per-bus motor handles (`left_arm_motors`, `right_arm_motors`, `head_motors`, `base_motors`), and `use_degrees` behavior — all re-keyed to the shipped bus layout.

#### Scenario: Camera helper returns three frames
- **WHEN** the camera observation helper is called
- **THEN** it returns one frame per configured camera key (`head`, `left_wrist`, `right_wrist`) without waiting for a freshly captured frame

#### Scenario: Dual-arm examples operate with the new bus names
- **WHEN** the dual-arm pi0.5 inference example and the dual-arm VR recording example are run against the robot
- **THEN** they read both arms through the driver's per-bus handles, move the arms to the start pose, and stream actions without attribute errors
