# Franka FR3 / Panda Integration Guide

This guide covers a conservative integration path for Linkerbot L20 Lite, L20, or L30 hands on Franka Research 3 and Panda arms. It is intended for lab engineering teams preparing first motion, not as a substitute for Franka, Linkerbot, or site safety documentation.

## Section 1 - Prerequisites

### Hardware Required

- Franka Research 3 or Franka Emika Panda with a validated controller, brakes, and emergency stop chain.
- Linkerbot L20 Lite, L20, or L30 hand with vendor controller and firmware version recorded in the lab log.
- Adapter plate from Franka wrist flange to Linkerbot wrist pattern. Franka-side flange should follow ISO 9409-1-50-4-M6; Linkerbot-side geometry is `[VENDOR_SPEC_REQUIRED]`.
- External DC power supply for the hand, fused and rated to `[VENDOR_SPEC_REQUIRED]`.
- CAN, CAN FD, or RS-485 interface hardware matching the selected hand model.

### Tools Needed

- Torque wrench and M6 tooling suitable for the wrist flange.
- Franka Desk access for payload, end-effector, and safety configuration.
- Linux workstation with ROS 2 Humble, Iron, or Jazzy plus `libfranka`, `franka_ros2`, or the lab-approved Franka control stack.
- Multimeter, CAN analyzer or USB-to-CAN adapter, and cable continuity tester.

### Cable List

- Shielded CAN/CAN FD or RS-485 cable for the hand bus.
- DC power cable sized for `[VENDOR_SPEC_REQUIRED]` peak current.
- Ethernet cable for Franka controller connectivity.
- Strain-relief hardware suitable for repeated Cartesian compliance tests.

## Section 2 - Mechanical Mounting

### Flange Adapter Specification

Use a Franka-side ISO 9409-1-50-4-M6 adapter with a centered hand frame. The adapter should keep the hand close to the wrist to reduce torque load and preserve Cartesian compliance behavior. The Linkerbot bolt circle, connector clearance, and dowel references are `[VENDOR_SPEC_REQUIRED]`.

### Torque Specs

- Tighten Franka-side M6 fasteners to the current Franka service-manual value.
- Tighten Linkerbot-side fasteners to `[VENDOR_SPEC_REQUIRED]`.
- Recheck torque after the first hour of low-force testing.
- Do not use a fastener length that bottoms out in the wrist flange or hand housing.

### Cable Strain Relief Notes

Franka arms are often used in impedance-control experiments, so cable stiffness matters. Route the cable bundle with enough slack to avoid biasing force-torque behavior, but restrain it near the wrist so the hand connectors cannot be pulled during recovery motion.

### Payload and Center-of-Gravity Check

Set the exact end-effector mass, inertial estimate, and center of gravity in the Franka control stack before enabling motion. Validate separately for FR3 and Panda because controller limits, payload allowances, and lab safety configurations may differ. Re-run collision-threshold tuning after the hand, adapter, and cables are mounted.

## Section 3 - Electrical Connection

### CAN / CAN FD Wiring Diagram

```text
ROS 2 workstation / bus adapter           Linkerbot hand controller
┌─────────────────────────────┐           ┌────────────────────────┐
│ CAN_H or CANFD_H ───────────┼───────────┤ CAN_H / CANFD_H        │
│ CAN_L or CANFD_L ───────────┼───────────┤ CAN_L / CANFD_L        │
│ Signal GND ─────────────────┼───────────┤ Signal GND             │
│ Shield drain ── chassis ────┼────┐      │ Shield drain           │
└─────────────────────────────┘    │      └────────────────────────┘
                                   │
                         termination and bitrate
                         [VENDOR_SPEC_REQUIRED]
```

For L20 Lite or L20 over RS-485, substitute the differential `A/B` pair and confirm Modbus address, baud rate, parity, and termination from the vendor manual.

### Power Supply Requirements

Use a dedicated hand supply rather than drawing from the Franka controller. Set voltage, current limit, fuse value, and connector polarity to `[VENDOR_SPEC_REQUIRED]`. Keep the hand power enable under operator control during early tests so the arm can be evaluated without energizing fingers.

### EMI Shielding Notes

- Keep hand power wiring away from Franka Ethernet and controller cables.
- Bond shield drains according to the lab grounding scheme, with one reference point preferred for first tests.
- Watch for communication errors during high-acceleration impedance moves.
- Capture bus logs before and after finger motion to separate hand noise from arm-control issues.

## Section 4 - Software Setup

### ROS 2 Package Dependencies

Prepare the SDK and Franka control stack in separate workspaces or a single overlay:

```bash
python -m pip install -e ".[dev]"
cd ros2_ws
rosdep install --from-paths src --ignore-src -r -y
colcon build --symlink-install
source install/setup.bash
```

Required hand-side packages include `rclpy`, `sensor_msgs`, `launch`, and `launch_ros`. Franka-side packages are lab-specific; verify compatibility with the installed ROS 2 distribution before combining launch files.

### URDF Modification

Attach the hand to the Franka flange link with a fixed joint. Keep inertial values as `[VENDOR_SPEC_REQUIRED]` until measured or supplied by Linkerbot.

```xml
<joint name="franka_flange_to_linkerbot_hand" type="fixed">
  <parent link="fr3_hand_tcp"/>
  <child link="linkerbot_hand_base"/>
  <origin xyz="[VENDOR_SPEC_REQUIRED]" rpy="[VENDOR_SPEC_REQUIRED]"/>
</joint>
```

If the lab uses Panda naming, replace `fr3_hand_tcp` with the active Panda flange or TCP link from the local URDF.

### Launch File Snippet

```python
from launch import LaunchDescription
from launch_ros.actions import Node


def generate_launch_description() -> LaunchDescription:
    return LaunchDescription(
        [
            Node(
                package="dexterous_hand_ros2",
                executable="hand_node",
                name="dexterous_hand",
                output="screen",
            )
        ]
    )
```

### MoveIt 2 Planning Group Setup

Define a `hand` planning group for Linkerbot joints and preserve the Franka arm group as the primary collision-checked manipulator. Add a combined `franka_with_hand` group after validating attached collision geometry and disabling no real finger collisions. Use conservative named states for first planning: `relaxed_open`, `platform`, and `precision_pinch`.

## Section 5 - Verification Checklist

- Confirm the Franka controller, emergency stop, and recovery behavior before mounting the hand.
- Verify adapter fit, fastener length, torque, and witness marks.
- Configure end-effector mass, center of gravity, and collision thresholds for the mounted hand.
- Confirm the cable bundle does not affect force-control behavior during slow free-space moves.
- Validate hand power polarity and current limit before connecting the controller.
- Confirm bus termination, bitrate, node ID, and protocol mode.
- Run the hand node in mock mode and confirm `/joint_states` publishes.
- With the arm stationary, command one low-amplitude finger motion.
- Test Franka stop/recovery and hand `stop()` behavior as separate actions.
- Run combined motion in a cleared workspace with reduced speed and a spotter at the emergency stop.
