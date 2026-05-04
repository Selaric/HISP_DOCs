# TurboPi Setup Guide
**TAI Lab — University of Michigan Dearborn**  
**Researcher:** Selase | **PI:** Prof. Xiao Zhang  
**Date:** May 4, 2026

---

## Hardware

| Component | Details |
|---|---|
| Robot | Hiwonder TurboPi (Mecanum wheel chassis) |
| Compute | Raspberry Pi 5 |
| Container | Docker — name: `TurboPi`, image: `mentorpi` |
| Camera | icSpring HP60C USB camera |
| Ultrasonic | Sonar sensor (built-in) |
| LiDAR | LD19 (borrowed from MentorPi — purchase second unit) |
| mmWave | IWR6843AOP (ports confirmed, driver pending) |
| Gamepad | SHANWAN Android Gamepad |

---

## Network & Access

- **IP Address:** `192.168.0.102`
- **SSH:** `ssh pi@192.168.0.102` (password: raspberry)
- **Docker entry:**

```bash
docker exec -it -u ubuntu -w /home/ubuntu TurboPi /bin/bash
source ros2_ws/install/setup.bash
```

---

## Foxglove

```bash
ros2 launch foxglove_bridge foxglove_bridge_launch.xml
```

Connect in browser: `ws://192.168.0.102:8765`

---

## Sensors Confirmed Working

| Sensor | Topic | Rate | Status |
|---|---|---|---|
| Camera | `/image_raw` | ~30Hz | ✅ |
| Ultrasonic | `/sonar_controller/get_distance` | ~1539Hz | ✅ |
| LiDAR LD19 | `/scan` | 10Hz | ✅ (temp from MentorPi) |

---

## Port Mapping

```bash
ls /dev/serial/by-id/
```

| Device | Port |
|---|---|
| LiDAR LD19 | `/dev/ttyUSB2` (`usb-1a86_USB_Serial-if00-port0`) |
| mmWave CLI | `/dev/serial/by-id/usb-Silicon_Labs_CP2105...if00-port0` |
| mmWave Data | `/dev/serial/by-id/usb-Silicon_Labs_CP2105...if01-port0` |

---

## LiDAR Setup (LD19)

Driver not included — built from source:

```bash
cd ~/ros2_ws/src
git clone https://github.com/ldrobotSensorTeam/ldlidar_stl_ros2.git
cd ~/ros2_ws
colcon build --packages-select ldlidar_stl_ros2
source install/setup.bash
```

Port fix (TurboPi uses `/dev/ttyUSB2` not `/dev/ttyUSB0`):

```bash
python3 -c "
content = open('/home/ubuntu/ros2_ws/src/ldlidar_stl_ros2/launch/ld19.launch.py').read()
content = content.replace('/dev/ttyUSB0', '/dev/ttyUSB2')
open('/home/ubuntu/ros2_ws/src/ldlidar_stl_ros2/launch/ld19.launch.py', 'w').write(content)
print('Done')
"
cd ~/ros2_ws && colcon build --packages-select ldlidar_stl_ros2 && source install/setup.bash
```

Launch:

```bash
ros2 launch ldlidar_stl_ros2 ld19.launch.py
```

---

## Auto-source ROS2 (fix new terminal sessions)

```bash
echo "source /home/ubuntu/ros2_ws/install/setup.bash" >> ~/.bashrc
source ~/.bashrc
```

---

## Nodes Running (bringup)

```
/avoidance_node
/gesture_control_node
/line_following
/mecanum_chassis_node
/object_tracking
/qrcode
/ros_robot_controller
/rosbridge_websocket
/sonar_controller
/startup_check_node
/usb_cam
/web_video_server
```

---

## Pending Items

- [ ] Purchase second LD19 LiDAR (~$60-80)
- [ ] Set up mmWave driver (same config as MentorPi)
- [ ] Add SLAM (slam_toolbox)
- [ ] Build TF tree
- [ ] Add LiDAR + mmWave + SLAM to bringup launch file
- [ ] Record first multimodal bag file

---

## Notes

- TurboPi has **Mecanum wheels** — can strafe, rotate in place (better than MentorPi Ackermann for indoor navigation)
- MentorPi is primary research platform; TurboPi is secondary testbed
- Same Docker/ROS2 setup as MentorPi — knowledge transfers directly
