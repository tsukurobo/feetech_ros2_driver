# User Guide

> **⚠️ Migration notice:** The `offset` parameter is deprecated and ignored. The driver now always centers at 2048 (midpoint of the 0–4095 tick range). Use `homing_offset` instead — it writes directly to the servo's EEPROM so the centering happens in hardware. To migrate: `homing_offset = old_offset - 2048`. The driver will log a warning if it detects the old `offset` parameter.

## ros2_control urdf tag

The feetech system interface has a few `ros2_control` urdf tags to customize its behavior.

#### Hardware Parameters

* `usb_port` (required). Example: `<param name="usb_port">/dev/ttyUSB0</param>`.
* `baud_rate` (optional): Servo bus baud rate. Defaults to `1000000`.
* `joint_config_file` (optional): Path to a YAML file with per-joint parameters. If omitted, only URDF params are used (backward-compatible). See [YAML Joint Configuration](#yaml-joint-configuration-file) below.
* `auto_reconnect` (optional): Start without the serial device or servo power, and reopen the serial port and restore communication after a USB or servo power interruption. Defaults to `true`.
* `reconnect_interval_ms` (optional): Minimum delay between reconnection attempts in milliseconds. Defaults to `1000`.

#### Per-joint Parameters

Make sure to look at [Memory table](https://docs.google.com/spreadsheets/d/1GVs7W1VS1PqdhA1nW-abeyAHhTUxKUdR/edit?gid=364516031#gid=364516031) for a detailed explanation of the parameters.

* `id` (**required**, must be in URDF): Servo ID on the bus. The driver uses this to address the servo and to match YAML config entries. Example: `<param name="id">1</param>`.
* `p_coefficient` (optional): Proportional coefficient of the PID controller. Example: `<param name="p_coefficient">8</param>`.
* `i_coefficient` (optional): Integral coefficient of the PID controller. Example: `<param name="i_coefficient">0</param>`.
* `d_coefficient` (optional): Derivative coefficient of the PID controller. Example: `<param name="d_coefficient">32</param>`.
* `homing_offset` (optional): Signed offset written to the servo's EEPROM. The servo firmware applies `Present_Position = Actual_Position - Homing_Offset`, so setting `homing_offset = actual_position - 2048` makes the servo report 2048 (center) at your desired physical center. If migrating from the old `offset` parameter: `homing_offset = old_offset - 2048` (since the old offset was effectively the actual position at center).
* `multi_turn` (optional): Set to `true` for STS3215 signed multi-turn position commands and feedback. The driver enables Phase bit 4 and forces both angle limits to zero, then reads the Phase and angle-limit registers back to verify the configuration. The ROS position remains centered at 2048 and is limited to ±7 motor rotations. Turn count is not retained after servo power loss. Defaults to `false`.
* `range_min` (optional): Minimum angle limit (raw ticks, after homing offset is applied). Forced to `0` when `multi_turn` is enabled.
* `range_max` (optional): Maximum angle limit (raw ticks, after homing offset is applied). Forced to `0` when `multi_turn` is enabled.
* `max_torque_limit` (optional): Maximum torque limit.
* `protection_current` (optional): Protection current threshold.
* `overload_torque` (optional): Overload torque threshold.
* `return_delay_time` (optional): Response delay time.
* `speed` (optional): Position command speed written to addresses 46–47 in raw servo units (0–32767). Defaults to 2400. Select a practical value for the connected servo model.
* `acceleration` (optional): Position command acceleration written to address 41 in raw servo units (0–254). Defaults to 50.

### Example

Take a look at [ros2_so_arm100](https://github.com/JafarAbdi/ros2_so_arm100/blob/main/so_arm100_description/control/so_arm100.ros2_control.xacro) for an example of how to use the URDF tags.

---

## YAML Joint Configuration File

As an alternative (or addition) to URDF `<param>` tags, joint parameters can be loaded from a YAML file. This is useful for calibration values that change between robots (like `homing_offset`) without modifying the URDF.

The `id` parameter **must** be defined in the URDF (it is the hardware identity used to address the servo on the bus). YAML entries are matched to URDF joints by servo `id`, not by joint name — so the YAML joint name is just a human-friendly label. When both URDF params and YAML are provided for the same `id`, YAML values take precedence.

### Format

```yaml
joints:
  joint_name:
    id: 1
    homing_offset: 530
    range_min: 866
    range_max: 3231
    p_coefficient: 16
    i_coefficient: 0
    d_coefficient: 32
    return_delay_time: 0
    speed: 200
    acceleration: 254
```

### URDF Integration

Pass the YAML file path as a hardware parameter:

```xml
<param name="joint_config_file">$(find my_robot_bringup)/config/joints.yaml</param>
```

### Examples

* URDF-only setup: [ros2_so_arm100](https://github.com/JafarAbdi/ros2_so_arm100/blob/main/so_arm100_description/control/so_arm100.ros2_control.xacro)
* YAML config setup: [so101-ros-physical-ai](https://github.com/legalaspro/so101-ros-physical-ai) — see [follower](https://github.com/legalaspro/so101-ros-physical-ai/blob/main/so101_bringup/config/hardware/follower_joints.yaml) and [leader](https://github.com/legalaspro/so101-ros-physical-ai/blob/main/so101_bringup/config/hardware/leader_joints.yaml) arm configs.

## Experimental STS3215 internal parameters

`internal_vmax` (84), `internal_amax` (85), and `internal_kacc` (86) accept raw
one-byte values 0–254 through the existing per-joint URDF/YAML mechanism.
They are separate from command `speed` and `acceleration`. Omitted values are not written.
Set `allow_experimental_internal_parameters: true` on the joint to enable them.
Only STS3215 model bytes 9,3 (777) and firmware bytes 3,10 are accepted.
The byte range is an implementation limit, not a validated mechanical operating range.

```yaml
joints:
  servo:
    id: 1
    allow_experimental_internal_parameters: true
    internal_vmax: 100
    internal_amax: 80
    internal_kacc: 10
```

Current values are read and logged before modification, identical values are skipped,
and writes are read back. To restore, put the logged original values back in the
URDF/YAML. Removing a parameter does not restore the servo register.
`velocity_i_gain` (39, 0–254) and `experimental_dts` (81, 1–254) use the same
experimental opt-in and firmware check. DTs is a label from the experiment;
its exact control-period semantics and storage behavior remain unverified.
`return_delay_time` (7, 0–254) and `acceleration` (41, 0–254) remain normal parameters.

The [published table](https://ftservo.github.io/ftservo-wiki/reference/parameter/memory-sts/)
marks 84–86 as read-only factory parameters, while
[the FW3.10 experiment](https://zamantika.com/devemin/status/2094640983199842636)
reports direct writes and faster movement. Meaning, persistence and compatibility
across builds are not established by that report; firmware 3.10 is not a guarantee.

## Feedback in /dynamic_joint_states

Declare `voltage`, `temperature`, `current`, `servo_status`, and `connected` state
interfaces on each joint, alongside position and velocity. The joint state
broadcaster publishes them in `/dynamic_joint_states` using the existing sync read
(registers 56–70). Units are V, degrees Celsius and A (signed, 6.5 mA/count).
`servo_status` is the raw register 65 bitmask: 1 voltage, 2 magnetic encoder,
4 temperature, 8 current, 32 overload. See the
[STS register table](https://ftservo.github.io/ftservo-wiki/reference/parameter/memory-sts/).
`connected` is 1 after successful feedback and 0 when disconnected; additional
feedback is NaN until first feedback or after a communication failure. Position
and velocity retain their existing behavior. A failed group read invalidates all
servos, so `connected=0` does not identify the faulty servo. Short events between
polls and values during a complete power loss cannot be captured.
