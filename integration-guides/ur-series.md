# Universal Robots UR5e / UR10e Integration Guide

This guide describes a first-pass integration pattern for mounting a Linkerbot L20 Lite, L20, or L30 hand on Universal Robots e-Series arms. Treat it as an engineering checklist, not a certified installation procedure; CE certification and final safety validation remain under verification.

## Section 1 - Prerequisites

### Hardware Required

- Universal Robots UR5e or UR10e arm with a functioning teach pendant and current site-approved safety configuration.
- Linkerbot L20 Lite, L20, or L30 hand with matching vendor controller, firmware image, and vendor manual.
- Mechanical adapter from UR tool flange to the Linkerbot wrist pattern. Linkerbot-side hole pattern is `[VENDOR_SPEC_REQUIRED]`.
- External hand power supply rated for the selected hand model. Exact voltage, current, fuse, and inrush values are `[VENDOR_SPEC_REQUIRED]`.
- CAN, CAN FD, or RS-485 interface hardware matching the hand model:
  - L20 Lite / L20: CAN or RS-485.
  - L30: CAN FD.

### Tools Needed

- Torque wrench covering the selected M6 fastener range.
- Hex key set, threadlocker approved by the lab, calipers, and a multimeter.
- Linux workstation with ROS 2 Humble, Iron, or Jazzy and access to the SDK workspace.
- UR teach pendant access for payload, TCP, tool I/O, and reduced-speed test modes.

### Cable List

- Shielded twisted pair for CAN/CAN FD or RS-485 communication.
- Dedicated DC power cable sized for `[VENDOR_SPEC_REQUIRED]` peak current.
- USB-to-CAN, PCIe CAN, or robot-controller bridge approved by the lab.
- Strain-relief sleeve, cable ties, grounding strap, and emergency-disconnect label.

## Section 2 - Mechanical Mounting

### Flange Adapter Specification

Use a rigid adapter plate with a UR-side flange compatible with ISO 9409-1-50-4-M6. The Linkerbot-side bolt circle, dowel locations, and wrist orientation are `[VENDOR_SPEC_REQUIRED]` and must be taken from the final hand mechanical drawing.

Recommended adapter properties:

- Aluminum or steel plate with no flex under expected grasp loads.
- Centering feature or dowel reference where available.
- Clocking mark so the hand frame can be reproduced after removal.
- Clearance for communication and power connectors without bending the cables.

### Torque Specs

- Tighten UR-side M6 fasteners to the value specified in the Universal Robots service manual.
- Tighten Linkerbot-side fasteners to `[VENDOR_SPEC_REQUIRED]`.
- Use threadlocker only if approved for both the robot flange and hand housing.
- Mark fasteners after torqueing so loosening is visible during inspection.

### Cable Strain Relief

Route the cable bundle along the forearm side that remains outside the wrist pinch zone through the planned range of motion. Leave a service loop near the tool flange, then clamp the bundle to the adapter or wrist cover so connector bodies never carry cable load.

### Payload and Center-of-Gravity Check

In PolyScope, update payload, TCP, and center of gravity before moving the arm. Include the hand, adapter, cables attached to the tool, and any grasped object. Verify UR5e and UR10e margin separately because the same hand can be acceptable on UR10e while leaving little margin on UR5e during extended-reach motion.

## Section 3 - Electrical Connection

### CAN / CAN FD Wiring Diagram

```text
Linux ROS 2 PC / CAN adapter              Linkerbot hand controller
┌──────────────────────────┐              ┌────────────────────────┐
│ CAN_H  ──────────────────┼──────────────┤ CAN_H / CANFD_H        │
│ CAN_L  ──────────────────┼──────────────┤ CAN_L / CANFD_L        │
│ GND    ──────────────────┼──────────────┤ Signal GND             │
│ Shield ───── chassis ────┼──────┐       │ Shield drain           │
└──────────────────────────┘      │       └────────────────────────┘
                                  │
                         terminate per bus layout
                         [VENDOR_SPEC_REQUIRED]
```

For RS-485 on L20 Lite or L20, use the same topology with `A/B` differential lines instead of CAN high/low. Confirm termination, biasing, baud rate, and protocol mode from the vendor manual before enabling motion.

### Power Supply Requirements

Do not power the hand from the UR tool connector unless the vendor current draw and UR tool-output limits have both been checked. Use an external, fused DC supply with voltage and current set to `[VENDOR_SPEC_REQUIRED]`. Connect power ground and signal ground according to the Linkerbot controller manual to avoid ground loops.

### EMI Shielding Notes

- Use shielded twisted-pair communication cable and bond shield at one end unless the lab grounding plan specifies otherwise.
- Keep motor power and CAN/CAN FD lines separated where possible.
- Add ferrites near the hand controller if bus errors appear during high-speed wrist motion.
- Log CAN error counters during first-motion tests.

## Section 4 - Software Setup

### ROS 2 Package Dependencies

Install the SDK and ROS 2 package in a workspace:

```bash
python -m pip install -e ".[dev]"
cd ros2_ws
rosdep install --from-paths src --ignore-src -r -y
colcon build --symlink-install
source install/setup.bash
```

Expected ROS 2 dependencies include `rclpy`, `sensor_msgs`, `launch`, and `launch_ros`. Vendor hardware transport dependencies are `[VENDOR_SDK_REQUIRED]`.

### URDF Modification

Add the hand under the UR wrist link, usually `tool0`, using a fixed joint. Keep the UR arm URDF unchanged and include a separate hand macro when final Linkerbot meshes and inertial values are available.

```xml
<joint name="tool0_to_linkerbot_hand" type="fixed">
  <parent link="tool0"/>
  <child link="linkerbot_hand_base"/>
  <origin xyz="[VENDOR_SPEC_REQUIRED]" rpy="[VENDOR_SPEC_REQUIRED]"/>
</joint>
```

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

Create a separate `hand` planning group for finger joints and keep the UR manipulator group unchanged. Add a combined `ur_with_hand` group only after collision meshes, joint limits, and self-collision matrices are validated. Start with named states such as `relaxed_open`, `precision_pinch`, and `cylindrical_power`.

## Section 5 - Verification Checklist

- Confirm the robot is in reduced speed mode before the hand is attached.
- Verify all adapter fasteners are torqued and marked.
- Check payload, TCP, and center of gravity in PolyScope for the exact mounted configuration.
- Confirm the cable bundle does not tighten, rub, or enter pinch zones across the planned wrist path.
- Measure DC supply voltage at the controller input before connecting the hand.
- Confirm CAN/CAN FD or RS-485 termination and bus bitrate.
- Run `ros2 launch dexterous_hand_ros2 hand.launch.py mock:=true` before hardware mode.
- Move one finger joint at low speed while the arm is stationary.
- Test hand `stop()` behavior and the UR emergency stop independently.
- Execute combined arm-hand motion only after successful mock, stationary, and reduced-speed tests.
