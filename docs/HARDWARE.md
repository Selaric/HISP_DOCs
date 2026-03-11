# MentorPi A1 — Hardware Documentation
**Platform:** MentorPi A1 (Ackermann Steering)  
**ROS2 Distribution:** Humble  
**Researcher:** [Your Name]  
**PI:** Xiao Zhang  
**Last Updated:** March 2026

---

## 1. Platform Overview

The MentorPi A1 is a 4-wheeled Ackermann steering robot running ROS2 Humble inside a Docker container on a Raspberry Pi. It is equipped with LiDAR, an RGB-D camera, IMU, and motor encoders for odometry.

**Key constraint:** This is an Ackermann chassis (like a car), not a differential drive or Mecanum robot. It cannot perform point turns — it has a minimum turning radius determined by the steering servo angle.

---

## 2. Critical Setup — MACHINE_TYPE

This is the most important thing to know about this robot.

The environment variable `MACHINE_TYPE` **must** be set to `MentorPi_Acker` inside the Docker container before launching any nodes. Without it, the steering controller never initializes and the front wheels will not respond to any commands.

**Symptom if not set:** Robot drives forward but front wheels do not turn. PWM servo topic has no subscribers.

**Fix:**
```bash
export MACHINE_TYPE=MentorPi_Acker
```

**Make it permanent inside the container:**
```bash
echo 'export MACHINE_TYPE=MentorPi_Acker' >> ~/.bashrc
```

**Verify:**
```bash
echo $MACHINE_TYPE
# Should print: MentorPi_Acker
```

---

## 3. Docker Setup

The entire ROS2 environment runs inside a Docker container named `MentorPi`.

**Enter the container:**
```bash
docker exec -it -u ubuntu -w /home/ubuntu adb8 /bin/bash
```

**Source ROS2:**
```bash
source ros2_ws/install/setup.bash
```

**Important:** Always set MACHINE_TYPE after entering the container if it is not already in `.bashrc`.

---

## 4. Launching the Hardware Controller

The `ros_robot_controller` node is **not auto-launched** at startup. It must be started manually in a dedicated terminal. This node handles all hardware communication — motors, servos, IMU, battery.

**Launch command:**
```bash
ros2 launch ros_robot_controller ros_robot_controller.launch.py
```

**Keep this terminal open.** The node runs in the foreground. Open a second terminal for all other commands.

**Verify it is running:**
```bash
ros2 node list | grep ros_robot_controller
# Should return: /ros_robot_controller
```

**Warning:** Do not launch this node twice. Two instances cause conflicts and neither will control the hardware correctly. Always check `ros2 node list` before launching.

---

## 5. Steering Servo

The front wheel steering is controlled by a PWM servo.

| Parameter | Value |
|---|---|
| Servo ID | **3** |
| Full Left | 1000 |
| Center (Straight) | 1500 |
| Full Right | 2000 |
| Duration | Seconds to reach position (0.02 = instant, 0.5 = smooth) |

**How servo ID 3 was found:** By reading the source file:
```
ros2_ws/src/peripherals/peripherals/joystick_control.py
```
The code explicitly sets `servo_state.id = [3]` for Ackermann steering.

**Manual steering command:**
```bash
# Full left
ros2 topic pub --once /ros_robot_controller/pwm_servo/set_state \
  ros_robot_controller_msgs/msg/SetPWMServoState \
  "{state: [{id: [3], position: [1000], offset: [0]}], duration: 0.5}"

# Center / Straight
ros2 topic pub --once /ros_robot_controller/pwm_servo/set_state \
  ros_robot_controller_msgs/msg/SetPWMServoState \
  "{state: [{id: [3], position: [1500], offset: [0]}], duration: 0.5}"

# Full right
ros2 topic pub --once /ros_robot_controller/pwm_servo/set_state \
  ros_robot_controller_msgs/msg/SetPWMServoState \
  "{state: [{id: [3], position: [2000], offset: [0]}], duration: 0.5}"
```

**Message structure:**
```
ros_robot_controller_msgs/msg/SetPWMServoState
  ros_robot_controller_msgs/PWMServoState[] state
    uint16[] id
    uint16[] position
    int16[] offset
  float64 duration
```

---

## 6. Motor / Velocity Control

High-level movement is controlled via the standard ROS2 `cmd_vel` topic using `geometry_msgs/Twist`.

| Field | Meaning |
|---|---|
| `linear.x` | Forward/backward speed (m/s). Positive = forward |
| `angular.z` | Turn rate (rad/s). Positive = left, Negative = right |

**Move forward:**
```bash
ros2 topic pub /cmd_vel geometry_msgs/msg/Twist \
  "{linear: {x: 0.3, y: 0.0, z: 0.0}, angular: {z: 0.0}}"
```

**Turn in a circle:**
```bash
ros2 topic pub /cmd_vel geometry_msgs/msg/Twist \
  "{linear: {x: 0.3, y: 0.0, z: 0.0}, angular: {z: 1.5}}"
```

**Full stop:**
```bash
ros2 topic pub /cmd_vel geometry_msgs/msg/Twist \
  "{linear: {x: 0.0, y: 0.0, z: 0.0}, angular: {z: 0.0}}"
```

**Important:** Always use `ros2 topic pub` without `--once` for stop commands to ensure the message is received.

---

## 7. Key ROS2 Topics

| Topic | Type | Description |
|---|---|---|
| `/cmd_vel` | geometry_msgs/Twist | High-level velocity command |
| `/controller/cmd_vel` | geometry_msgs/Twist | Controller-level velocity |
| `/ros_robot_controller/pwm_servo/set_state` | SetPWMServoState | Direct servo control |
| `/ros_robot_controller/set_motor` | MotorsState | Direct motor control |
| `/ros_robot_controller/battery` | std_msgs/UInt16 | Battery voltage (millivolts) |
| `/scan_raw` | sensor_msgs/LaserScan | LiDAR raw scan data |
| `/ascamera/camera_publisher/rgb0/image` | sensor_msgs/Image | RGB camera feed |
| `/imu` | sensor_msgs/Imu | IMU data |
| `/odom` | nav_msgs/Odometry | Wheel odometry |
| `/joint_states` | sensor_msgs/JointState | Joint states |

---

## 8. Ackermann Kinematics — Important Constraint

This robot uses **Ackermann steering geometry** — the same as a standard car. This means:

- The front wheels turn to steer, rear wheels drive
- **Cannot perform point turns** (zero radius rotation)
- Has a **minimum turning radius** determined by max servo angle
- At max steering (position 1000 or 2000), the tightest circle possible is approximately 1–2 meters diameter (to be measured precisely)

This is a fundamental hardware constraint that affects all navigation and path planning decisions.

---

## 9. Known Issues & Lessons Learned

| Issue | Root Cause | Fix |
|---|---|---|
| Front wheels not turning | MACHINE_TYPE not set | `export MACHINE_TYPE=MentorPi_Acker` |
| PWM topic has no subscribers | ros_robot_controller not launched | `ros2 launch ros_robot_controller ros_robot_controller.launch.py` |
| Commands not reaching hardware | Two ros_robot_controller instances running | Kill one, check with `ros2 node list` |
| Paste errors in terminal (`~` characters) | Bracket paste mode enabled | Run `printf '\e[?2004l'` to disable |
| Robot doesn't stop with `--once` | Single message dropped | Use continuous publish without `--once` for stop |

---

## 10. Useful Debugging Commands

```bash
# See all running nodes
ros2 node list

# See all active topics
ros2 topic list

# Check what a node subscribes/publishes
ros2 node info /ros_robot_controller

# Check message structure
ros2 interface show ros_robot_controller_msgs/msg/SetPWMServoState

# Check if topic is publishing
ros2 topic hz /ros_robot_controller/battery

# Visualize node graph
rqt_graph

# Make robot beep (confirms controller is alive)
ros2 topic pub --once /ros_robot_controller/set_buzzer \
  ros_robot_controller_msgs/msg/BuzzerState \
  "{freq: 1000, on_time: 0.5, off_time: 0.1, repeat: 1}"
```
