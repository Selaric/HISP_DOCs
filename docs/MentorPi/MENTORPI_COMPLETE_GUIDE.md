# MentorPi A1 — Complete Setup & Operations Guide
**TAI Lab | University of Michigan-Dearborn**
**Last Updated:** May 1, 2026 | **Author:** Selase

---

## Robot Overview

| Component | Details |
|---|---|
| Robot | Hiwonder MentorPi A1 (Ackermann chassis) |
| Computer | Raspberry Pi 5 |
| OS | Ubuntu 22.04 (inside Docker container ID: adb8) |
| ROS | ROS2 Humble |
| Sensors | LD19 LiDAR, Angstrong HP60C Camera, TI IWR6843AOP mmWave |
| Controller | RCC board (Hiwonder) |
| Gamepad | SHANWAN Android Gamepad |

---

## Quick Start — Every Session

```bash
# 1. SSH into robot
ssh pi@192.168.0.112

# 2. Enter Docker
docker exec -it -u ubuntu -w /home/ubuntu adb8 /bin/bash
source ros2_ws/install/setup.bash

# 3. Bringup auto-starts on boot — verify sensors
ros2 topic hz /scan_raw        # LiDAR — should show 10Hz
ros2 topic hz /iwr6843_pcl     # mmWave — should show 30Hz
ros2 topic hz /map             # SLAM — should show 0.1Hz

# 4. Launch Foxglove bridge
ros2 launch foxglove_bridge foxglove_bridge_launch.xml

# 5. Connect Foxglove on laptop
# Open Foxglove Studio → ws://raspberrypi:8765
```

---

## Port Mapping

Always verify ports on boot — they can shift if devices are replugged.

```bash
ls -la /dev/serial/by-id/
```

| Device | ID | Port |
|---|---|---|
| LiDAR (LD19) | 1a86:7523 CH340 | `/dev/ldlidar` symlink |
| RCC Board | 1a86:55d4 | `/dev/ttyACM0` |
| mmWave CLI | CP2105 if00 | `/dev/serial/by-id/usb-Silicon_Labs_CP2105...-if00-port0` |
| mmWave Data | CP2105 if01 | `/dev/serial/by-id/usb-Silicon_Labs_CP2105...-if01-port0` |
| Camera | 3482:6723 NOVATEK | USB (auto detected) |

---

## Critical Configuration

### MACHINE_TYPE — Must be Ackermann
```bash
cat ~/ros2_ws/.typerc | grep MACHINE
# Must show: export MACHINE_TYPE=MentorPi_Acker
```

If wrong, fix permanently:
```bash
sed -i 's/ROSMentor_Mecanum/MentorPi_Acker/g' ~/ros2_ws/.typerc
sed -i 's/MentorPi_Mecanum/MentorPi_Acker/g' ~/ros2_ws/.typerc
```

### Verify bringup is using correct machine type
```bash
source ~/ros2_ws/.typerc
# Should show MACHINE: MentorPi_Acker in output
```

---

## Bringup Launch File

Auto-starts on boot. Contains all nodes:

**Location:**
```
~/ros2_ws/install/bringup/share/bringup/launch/bringup.launch.py
```

**What it launches:**
- LiDAR (LD19)
- Camera (Angstrong HP60C)
- mmWave driver (IWR6843AOP)
- slam_toolbox (SLAM)
- mmWave TF transform (base_link → iwr6843_frame)
- ROS robot controller
- Joystick control
- Foxglove/Rosbridge

**If bringup fails with `need_compile` error:**
```bash
source ~/ros2_ws/.typerc
ros2 launch bringup bringup.launch.py
```

**IMPORTANT:** Never launch sensors manually if bringup is running — causes duplicate instances fighting over ports.

---

## Sensor Status

### LiDAR — LD19
- **Topic:** `/scan_raw`
- **Type:** LaserScan
- **Rate:** 10Hz
- **Frame:** `lidar_frame`
- **Port:** `/dev/ldlidar` → ttyUSB (varies)
- **Status:** ✅ Working

**Verify:**
```bash
ros2 topic hz /scan_raw
```

### Camera — Angstrong HP60C
- **Topic:** `/ascamera/camera_publisher/rgb0/image`
- **Rate:** 15Hz
- **Status:** ✅ Working

**Verify:**
```bash
ros2 topic hz /ascamera/camera_publisher/rgb0/image
```

### mmWave — TI IWR6843AOP
- **Topic:** `/iwr6843_pcl`
- **Type:** PointCloud2
- **Rate:** 30Hz
- **Frame:** `iwr6843_frame`
- **Status:** ✅ Working (requires powered USB hub)

**Config file:**
```
~/ros2_ws/src/mmwave_ti_ros/ti_mmwave_ros2_pkg/cfg/6843AOP_3d.cfg
```

**Key config settings:**
```
clutterRemoval -1 0    # Shows static objects (good for SLAM)
bpmCfg -1 0 0 1       # CRITICAL — required for SDK 3.06, missing from original config
cfarCfg threshold: 12  # Detection sensitivity (lower = more noise, higher = misses objects)
```

**Verify:**
```bash
ros2 topic hz /iwr6843_pcl
ros2 topic echo /iwr6843_pcl | grep width  # Should show non-zero when objects present
```

---

## SLAM — slam_toolbox

SLAM builds a map of the environment as the robot drives.

**Auto-starts in bringup.** Manual launch if needed:

```bash
ros2 run slam_toolbox async_slam_toolbox_node --ros-args \
  -r scan:=/scan_raw \
  -p use_sim_time:=false \
  -p base_frame:=base_footprint \
  -p odom_frame:=odom \
  -p map_frame:=map \
  -p map_update_interval:=1.0 \
  -p transform_timeout:=0.5
```

**View map in Foxglove:**
- Add 3D panel
- Add topic `/map`
- Set Fixed Frame to `map`
- Drive robot with gamepad — watch map build

**Save map when done:**
```bash
ros2 run nav2_map_server map_saver_cli -f my_map
# Saves: my_map.pgm (image) + my_map.yaml (metadata)
```

**Map colors:**
- White = free space (explored)
- Black = walls/obstacles
- Grey = unknown (unexplored)

---

## Robot Movement

### Gamepad Control
1. Power on gamepad — LEDs flash
2. Wait for pairing — green LED stays on
3. Press **START** if gamepad went to sleep
4. **Left joystick up/down** = forward/backward
5. **Right joystick left/right** = steering (requires working servo)

### Terminal Control — Motors
```bash
ros2 topic pub --once /ros_robot_controller/set_motor \
  ros_robot_controller_msgs/msg/MotorsState \
  "{data: [{id: 1, rps: 0.5}, {id: 2, rps: 0.5}]}"
```

Stop:
```bash
ros2 topic pub --once /ros_robot_controller/set_motor \
  ros_robot_controller_msgs/msg/MotorsState \
  "{data: [{id: 1, rps: 0.0}, {id: 2, rps: 0.0}]}"
```

### Terminal Control — Steering Servo (ID 3, J5)
```bash
# Center
ros2 topic pub --once /ros_robot_controller/pwm_servo/set_state \
  ros_robot_controller_msgs/msg/SetPWMServoState \
  "{state: [{id: [3], position: [1500], offset: [0]}], duration: 0.5}"

# Left
ros2 topic pub --once /ros_robot_controller/pwm_servo/set_state \
  ros_robot_controller_msgs/msg/SetPWMServoState \
  "{state: [{id: [3], position: [1000], offset: [0]}], duration: 0.5}"

# Right
ros2 topic pub --once /ros_robot_controller/pwm_servo/set_state \
  ros_robot_controller_msgs/msg/SetPWMServoState \
  "{state: [{id: [3], position: [2000], offset: [0]}], duration: 0.5}"
```

---

## Data Recording

### Record all sensors (mcap format — opens in Foxglove directly)
```bash
ros2 bag record --storage mcap \
  /scan_raw /map /odom /iwr6843_pcl \
  /ascamera/camera_publisher/rgb0/image \
  -o slam_run_$(date +%Y%m%d_%H%M%S)
```

Press **Ctrl+C** to stop.

### Transfer to laptop
```bash
# Step 1 — Copy from Docker to Pi host
docker cp adb8:/home/ubuntu/FOLDER_NAME /home/pi/FOLDER_NAME

# Step 2 — Copy from Pi to Windows laptop (run on laptop)
scp -r pi@192.168.0.112:/home/pi/FOLDER_NAME C:\Users\YourName\Downloads\
```

### Open in Foxglove
File → Open → select mcap file → hit play to replay session

---

## Known Issues & Fixes

### Issue 1 — MACHINE_TYPE wrong (Mecanum instead of Acker)
**Symptom:** Servo doesn't respond, gamepad steering doesn't work
**Fix:**
```bash
sed -i 's/ROSMentor_Mecanum/MentorPi_Acker/g' ~/ros2_ws/.typerc
sudo pkill -f bringup && ros2 launch bringup bringup.launch.py
```

### Issue 2 — mmWave not publishing (bpmCfg missing)
**Symptom:** `sensorStart` fails with "Full configuration must be provided"
**Fix:** Already applied — `bpmCfg -1 0 0 1` added to config file

### Issue 3 — mmWave port shifts after replug
**Symptom:** Driver crashes with "no such file or directory"
**Fix:** Use by-id path (already in bringup). Ports shift but by-id path stays constant.

### Issue 4 — Duplicate node instances
**Symptom:** Foxglove flickering, topics unstable, "nodes share exact name" warning
**Fix:** Never launch manually if bringup is running. Reboot to clean state.
```bash
sudo reboot
```

### Issue 5 — Foxglove bridge port busy
**Symptom:** "Bind Error" when launching foxglove_bridge
**Fix:**
```bash
# On Pi host
sudo kill $(sudo lsof -t -i:8765)
```

### Issue 6 — LiDAR timeout after bringup
**Symptom:** `get ldlidar data is time out`
**Fix:** LiDAR USB port conflict — reboot cleanly, don't launch LiDAR manually if bringup is running

### Issue 7 — slam_toolbox not receiving scan
**Symptom:** `/map` not publishing
**Fix:** slam_toolbox subscribes to `/scan` not `/scan_raw` by default. Use remap:
```bash
-r scan:=/scan_raw
```
Already configured in bringup.

---

## Hardware Issues

### Steering Servo — RCC Board PWM Chip Dead
**Status:** ❌ Pending hardware replacement

**History:**
- Original servo burned — reversed polarity on J5 damaged J5 and J6
- New servo installed — still no movement
- i2cdetect shows empty grid — PWM chip not detected
- Multimeter confirmed: J5 and J6 have no power output, J3 and J4 have power but PWM chip is dead

**Diagnosis:**
```bash
sudo i2cdetect -y 1
# Should show 0x40 if PWM chip alive
# Empty grid = chip dead = board needs replacement
```

**Next step:** Order new Hiwonder RRC board. When installed, plug servo into J5 with correct polarity and test with ID 3.

---

## TF Tree

```
odom → base_footprint → base_link → lidar_frame
                                  → imu_link
                                  → depth_cam → ascamera_camera_link_0
                                             → ascamera_color_0
                                  → wheel_lb_Link
                                  → wheel_lf_Link
                                  → wheel_rb_Link
                                  → wheel_rf_Link

iwr6843_frame → (connected to base_link via static transform)
```

mmWave TF transform (in bringup):
```
base_link → iwr6843_frame
x=0.14, y=-0.02, z=0.06
```

---

## Research Context

**Project:** Multimodal sensor fusion for indoor parking SLAM
**Approach:** LiDAR + mmWave fused SLAM vs LiDAR-only SLAM
**Environment:** Indoor parking structure (one level)
**Method:** Human drives with gamepad, slam_toolbox builds map automatically
**Comparison:** Map quality, completeness, dynamic object detection

**Prof's goal (end of May 2026):** All sensors capturing data simultaneously, testbed ready for multimodal sensing model training.

---

## GitHub
**Repo:** https://github.com/Selaric/HISP_DOCs

**Docs folder contains:**
- `HARDWARE.md` — servo ID, PWM values, MACHINE_TYPE fix
- `MMWAVE_IWR6843AOP_SETUP.md` — complete mmWave setup with bpmCfg fix
- `FOXGLOVE_SSH_SETUP.md` — SSH + Foxglove visualization
- `SLAM_SETUP.md` — slam_toolbox setup and recording
- `MENTORPI_COMPLETE_GUIDE.md` — this file
