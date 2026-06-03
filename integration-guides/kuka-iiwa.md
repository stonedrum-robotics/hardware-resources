# KUKA LBR iiwa 7 / iiwa 14 Integration Guide

This guide outlines a practical integration path for Linkerbot dexterous hands on KUKA LBR iiwa 7 and iiwa 14 arms. It assumes a research setup using a ROS 2 bridge or supervisor process and does not replace KUKA safety validation, site rules, or Linkerbot vendor documentation.

## Section 1 - Prerequisites

### Hardware Required

- KUKA LBR iiwa 7 or iiwa 14 with a validated cabinet, smartPAD, and safety configuration.
- Linkerbot L20 Lite, L20, or L30 hand with vendor controller, firmware, and interface mode recorded.
- Adapter plate from the iiwa tool flange to Linkerbot wrist geometry. KUKA-side flange pattern and Linkerbot-side pattern must be confirmed from mechanical drawings: `[ARM_VENDOR_SPEC_REQUIRED]` and `[VENDOR_SPEC_REQUIRED]`.
- External hand power supply with voltage, current, fuse, and connector polarity set to `[VENDOR_SPEC_REQUIRED]`.
- CAN, CAN FD, or RS-485 interface hardware compatible with the hand model and the ROS 2 workstation.

### Tools Needed

- Torque wrench, metric hex tooling, calipers, and thread inspection tools.
- KUKA smartPAD access for tool data, payload, load-data validation, and reduced-speed mode.
- ROS 2 workstation with the dexterous-hand SDK and the lab-approved iiwa ROS bridge.
- CAN analyzer or USB-to-CAN adapter, multimeter, and continuity tester.

### Cable List

- Shielded twisted-pair CAN/CAN FD or RS-485 cable.
- Dedicated DC power cable sized for `[VENDOR_SPEC_REQUIRED]` peak load.
- Ethernet cable between the KUKA controller, bridge machine, and ROS 2 workstation as required by the local stack.
- Strain-relief clips and protective sleeving rated for repeated wrist rotation.

## Section 2 - Mechanical Mounting

### Flange Adapter Specification

The adapter must match the exact iiwa 7 or iiwa 14 tool flange pattern from the KUKA manual and the Linkerbot wrist pattern from the vendor drawing. Keep the adapter compact to reduce wrist moment, and add a clear frame mark for repeatable tool orientation. Do not assume interchangeability between iiwa variants without checking the arm drawing.

### Torque Specs

- Apply KUKA-side fastener torque from the active iiwa service manual.
- Apply Linkerbot-side torque from `[VENDOR_SPEC_REQUIRED]`.
- Use locking hardware only if approved by both vendors.
- Reinspect fasteners after initial low-speed and payload-identification runs.

### Cable Strain Relief Notes

The iiwa wrist can rotate through large ranges during research trajectories. Secure the cable bundle to the adapter, leave a controlled service loop, and run a wrist-only motion envelope before enabling finger commands. Add physical stops or software joint limits if the cable path cannot survive full wrist rotation.

### Payload and Center-of-Gravity Check

Enter hand, adapter, cable, and expected object mass in KUKA load data before motion. Validate iiwa 7 and iiwa 14 separately because allowable payload and wrist margin differ. Re-run load-data checks when the hand changes from mock adapter mass to final hardware.

## Section 3 - Electrical Connection

### CAN / CAN FD Wiring Diagram

```text
ROS 2 workstation / CAN bridge            Linkerbot hand controller
┌─────────────────────────────┐           ┌────────────────────────┐
│ CAN_H / CANFD_H ────────────┼───────────┤ CAN_H / CANFD_H        │
│ CAN_L / CANFD_L ────────────┼───────────┤ CAN_L / CANFD_L        │
│ Signal GND ─────────────────┼───────────┤ Signal GND             │
│ Shield drain ── cabinet ────┼────┐      │ Shield drain           │
└─────────────────────────────┘    │      └────────────────────────┘
                                   │
                          termination, bitrate, and
                          controller mode [VENDOR_SPEC_REQUIRED]
```

For RS-485 hand modes, replace CAN high/low with the `A/B` pair and verify biasing, Modbus parameters, and controller address from the Linkerbot manual.

### Power Supply Requirements

Use a dedicated external supply for the hand during first integration. Required voltage, current limit, fuse rating, and connector pinout are `[VENDOR_SPEC_REQUIRED]`. Keep the supply accessible to the operator and label it separately from the KUKA cabinet power.

### EMI Shielding Notes

- Keep hand motor power cables separate from KUKA cabinet and Ethernet lines.
- Bond cable shields according to the site grounding plan and avoid multiple uncontrolled shield-ground paths.
- Monitor CAN/CAN FD error counters during wrist rotation and finger motion.
- Add ferrites or reroute the bus if errors correlate with motor current spikes.

## Section 4 - Software Setup

### ROS 2 Package Dependencies

Build the hand package in the ROS 2 workspace that communicates with the iiwa bridge:

```bash
python -m pip install -e ".[dev]"
cd ros2_ws
rosdep install --from-paths src --ignore-src -r -y
colcon build --symlink-install
source install/setup.bash
```

Hand-side dependencies include `rclpy`, `sensor_msgs`, `launch`, and `launch_ros`. The iiwa-side bridge, controller interface, and synchronization mechanism are lab-specific and should be launched separately until both systems are stable.

### URDF Modification

Attach the hand base to the iiwa tool flange or active end-effector link using a fixed joint. Use placeholders until final Linkerbot CAD, inertial values, and cable-exit geometry are available.

```xml
<joint name="iiwa_tool_to_linkerbot_hand" type="fixed">
  <parent link="iiwa_link_ee"/>
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

Keep the iiwa arm planning group separate from the Linkerbot `hand` group during first tests. Add an `iiwa_with_hand` group only after collision geometry, attached-object behavior, and cable keep-out zones are represented. Validate named hand states such as `relaxed_open`, `hook`, and `cylindrical_power` before planning combined grasps.

## Section 5 - Verification Checklist

- Confirm KUKA reduced-speed operation and emergency stop behavior before mounting the hand.
- Verify adapter geometry, fastener torque, and tool-frame orientation marks.
- Enter and validate load data for hand, adapter, cables, and expected object.
- Sweep wrist motion slowly and confirm cable strain relief survives the envelope.
- Verify hand power voltage, current limit, and fuse before energizing the controller.
- Confirm communication termination, bitrate, protocol mode, and node address.
- Launch the hand node in mock mode and inspect `/joint_states`.
- Command one low-amplitude finger joint while the iiwa is stationary.
- Test hand `stop()` and KUKA stop behavior independently.
- Run combined iiwa-hand motion only after a cleared-workspace review and reduced-speed dry run.
