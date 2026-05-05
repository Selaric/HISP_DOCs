# TurboPi Setup Guide
**TAI Lab — University of Michigan Dearborn**  
**Researcher:** Selase | **PI:** Prof. Xiao Zhang  
**Last Updated:** May 5, 2026

---

## Hardware

| Component | Details |
|---|---|
| Robot | Hiwonder TurboPi (Mecanum wheel chassis) |
| Compute | Raspberry Pi 5 |
| Container | Docker — name: `TurboPi`, image: `mentorpi` |
| Camera | USB camera (`/dev/video0`) |
| Ultrasonic | Built-in sonar sensor |
| LiDAR | LD19 (borrowed from MentorPi — purchase second unit) |
| mmWave | IWR6843AOP |
| Gamepad | SHANWAN Android Gamepad |

---

## Network & Access

- **IP Address:** `192.168.0.102`
- **SSH:** `ssh pi@192.168.0.102` (password: raspberry)

**Docker entry:**
```bash
docker exec -it -u ubuntu -w /home/ubuntu TurboPi /bin/bash
source ros2_ws/install/setup.bash
```

**Auto-source ROS2 (permanent fix):**
```bash
echo "source /home/ubuntu/ros2_ws/install/setup.bash" >> ~/.bashrc
source ~/.bashrc
```

---

## Foxglove

```bash
ros2 launch foxglove_bridge foxglove_bridge_launch.xml
```

Connect: `ws://192.168.0.102:8765`

---

## Sensors — All Auto-Start on Boot ✅

| Sensor | Topic | Rate | Status |
|---|---|---|---|
| Camera | `/image_raw` | 30Hz | ✅ |
| Ultrasonic | `/sonar_controller/get_distance` | ~1539Hz | ✅ |
| LiDAR LD19 | `/scan` | 10Hz | ✅ |
| mmWave IWR6843AOP | `/iwr6843_pcl` | 30Hz | ✅ |

---

## Port Mapping

```bash
ls /dev/serial/by-id/
```

| Device | Port |
|---|---|
| LiDAR LD19 | `/dev/ttyUSB2` (`usb-1a86_USB_Serial-if00-port0`) |
| mmWave CLI | `usb-Silicon_Labs_CP2105...if00-port0` |
| mmWave Data | `usb-Silicon_Labs_CP2105...if01-port0` |

---

## LiDAR Setup (LD19)

Built from source — driver not in apt repo:

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

---

## mmWave Setup (IWR6843AOP)

**Install driver:**
```bash
cd ~/ros2_ws/src
git clone https://github.com/nhma20/iwr6843aop_pub.git
cd ~/ros2_ws
colcon build --packages-select iwr6843aop_pub
source install/setup.bash
```

**Apply fixes:**
```bash
# Fix 1 — empty read bug
sed -i 's/byte_buffer = self.data_port.read(self.data_port.in_waiting)/byte_buffer = self.data_port.read(self.data_port.in_waiting or 64)/' ~/ros2_ws/src/iwr6843aop_pub/iwr6843aop_pub/publisher_member_function.py

# Fix 2 — timeout fix
sed -i 's/if(self.warn>100):/if(self.warn>10000):/' ~/ros2_ws/src/iwr6843aop_pub/iwr6843aop_pub/publisher_member_function.py

# Fix 3 — numpy deprecation
sed -i 's/np\.float\b/float/g' ~/ros2_ws/src/iwr6843aop_pub/iwr6843aop_pub/publisher_member_function.py

# Fix 4 — array comparison bug
python3 -c "
content = open('/home/ubuntu/ros2_ws/src/iwr6843aop_pub/iwr6843aop_pub/publisher_member_function.py').read()
content = content.replace('if not xyzdata == []:', 'if len(xyzdata) != 0:')
open('/home/ubuntu/ros2_ws/src/iwr6843aop_pub/iwr6843aop_pub/publisher_member_function.py', 'w').write(content)
print('Done')
"
```

**Config file setup:**
```bash
cp ~/ros2_ws/src/iwr6843aop_pub/cfg_files/90deg_noGroup_18m_30Hz.cfg ~/ros2_ws/src/iwr6843aop_pub/cfg/6843AOP_3d.cfg
sed -i '/^clutterRemoval/a bpmCfg -1 0 0 1' ~/ros2_ws/src/iwr6843aop_pub/cfg/6843AOP_3d.cfg
```

**Rebuild after fixes:**
```bash
cd ~/ros2_ws && colcon build --packages-select iwr6843aop_pub && source install/setup.bash
```

**Manual launch (for testing):**
```bash
ros2 run iwr6843aop_pub pcl_pub --ros-args \
  -p cfg_path:=/home/ubuntu/ros2_ws/src/iwr6843aop_pub/cfg/6843AOP_3d.cfg \
  -p cli_port:=/dev/serial/by-id/usb-Silicon_Labs_CP2105_Dual_USB_to_UART_Bridge_Controller_01083319-if00-port0 \
  -p data_port:=/dev/serial/by-id/usb-Silicon_Labs_CP2105_Dual_USB_to_UART_Bridge_Controller_01083319-if01-port0
```

---

## Bringup Launch File

All sensors auto-start on boot via:
```
~/ros2_ws/src/bringup/launch/bringup.launch.py
```

Includes: camera, rosbridge, controller, LiDAR, LiDAR TF, mmWave, mmWave TF.

After any changes rebuild:
```bash
cd ~/ros2_ws && colcon build --packages-select bringup && source install/setup.bash
```

---

## TF Tree

```
base_link → base_laser     (LiDAR,  z=0.18m)           ✅
base_link → iwr6843_frame  (mmWave, x=0.14, y=-0.02, z=0.06) ✅
```

---

## Foxglove Visualization Confirmed

| Panel | Topic | Status |
|---|---|---|
| Camera | `/image_raw` | ✅ |
| Sonar Plot | `/sonar_controller/get_distance.data` | ✅ |
| LiDAR 3D | `/scan` | ✅ |
| mmWave 3D | `/iwr6843_pcl` | ✅ |

---

## Pending Items

- [ ] Purchase second LD19 LiDAR (~$60-80) for permanent install
- [ ] Set up SLAM (slam_toolbox)
- [ ] Add Foxglove bridge to bringup launch file
- [ ] Data recording bag file

---

## Notes

- TurboPi uses **Mecanum wheels** — can strafe and rotate in place
- MentorPi is primary research platform; TurboPi is secondary testbed
- mmWave config uses `noGroup` + `clutterRemoval OFF` for maximum raw points (better for AI training data)
- All 4 sensors confirmed working simultaneously in Foxglove
