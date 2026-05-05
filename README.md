# HISP_DOCs — TAI Lab Robotics Research Documentation

**High-precision Indoor Smart Parking (HISP)**  
**TAI Lab — University of Michigan-Dearborn**  
**PI:** Prof. Xiao Zhang | **Researchers:** Selase Doku, Ali Mheidly

---

## Overview

This repository documents the hardware, software, and research setup for the HISP project — a multimodal sensor fusion system for indoor localization and perception using mmWave radar, LiDAR, and camera sensors on ground robots. The platform supports real-time data collection, SLAM, and edge computing for indoor smart parking research.

---

## Research Goal

Develop a low-cost, high-precision indoor cross-floor localization system using mmWave radar fused with LiDAR and camera data on edge devices. The system targets real-world deployment in multi-story indoor environments such as parking garages.

**Target Publication:** EAI SmartSP — November 2026

---

## Robot Platforms

### MentorPi (Primary Research Platform)
- **Chassis:** Hiwonder MentorPi A1 (Ackermann steering)
- **Compute:** Raspberry Pi 5
- **Container:** Docker (`adb8`), image: `mentorpi`
- **IP:** `192.168.0.112`
- **Sensors:** LD19 LiDAR, Angstrong HP60C depth camera, IWR6843AOP mmWave radar, IMU
- **Status:** Software stack complete. Pending RCC board replacement for steering servo.

### TurboPi (Secondary Testbed)
- **Chassis:** Hiwonder TurboPi (Mecanum wheels)
- **Compute:** Raspberry Pi 5
- **Container:** Docker (`TurboPi`), image: `mentorpi`
- **IP:** `192.168.0.102`
- **Sensors:** USB camera, ultrasonic sonar, LD19 LiDAR (pending purchase), IWR6843AOP mmWave radar
- **Status:** Camera, sonar, and LiDAR driver configured. mmWave and SLAM pending.

### Galaxy RVR Shield (Exploratory)
- **Chassis:** Arduino-based differential drive
- **Compute:** Arduino Uno + shield (no Raspberry Pi)
- **Sensors:** ESP camera
- **Status:** Exploratory — capabilities under investigation.

---

## Software Stack

| Component | Details |
|---|---|
| OS | Ubuntu 22.04 (inside Docker) |
| ROS | ROS 2 Humble |
| Visualization | Foxglove Studio |
| SLAM | slam_toolbox (async mode) |
| Data Recording | ROS 2 bag (MCAP format) |
| mmWave Driver | `iwr6843aop_pub` (nhma20) |
| LiDAR Driver | `ldlidar_stl_ros2` (built from source) |

---

## Docker Access

**MentorPi:**
```bash
ssh pi@192.168.0.112
docker exec -it -u ubuntu -w /home/ubuntu adb8 /bin/bash
source ros2_ws/install/setup.bash
```

**TurboPi:**
```bash
ssh pi@192.168.0.102
docker exec -it -u ubuntu -w /home/ubuntu TurboPi /bin/bash
source ros2_ws/install/setup.bash
```

---

## Foxglove

```bash
ros2 launch foxglove_bridge foxglove_bridge_launch.xml
```

| Robot | WebSocket URL |
|---|---|
| MentorPi | `ws://192.168.0.112:8765` |
| TurboPi | `ws://192.168.0.102:8765` |

---

## Sensors & Topics

### MentorPi
| Sensor | Topic | Rate |
|---|---|---|
| LiDAR LD19 | `/scan_raw` | 10Hz |
| Depth Camera | `/ascamera/camera_publisher/rgb0/image` | 30Hz |
| mmWave | `/iwr6843_pcl` | 30Hz |
| IMU | `/imu` | — |
| Odometry | `/odom_raw` | — |
| SLAM Map | `/map` | — |

### TurboPi
| Sensor | Topic | Rate |
|---|---|---|
| USB Camera | `/image_raw` | 30Hz |
| Ultrasonic | `/sonar_controller/get_distance` | ~1539Hz |
| LiDAR LD19 | `/scan` | 10Hz (pending purchase) |
| mmWave | `/iwr6843_pcl` | 30Hz (pending setup) |

---

## Port Mapping

### MentorPi
| Device | Port |
|---|---|
| LiDAR LD19 | `/dev/ttyUSB2` (`usb-1a86_USB_Serial`) |
| mmWave CLI | `usb-Silicon_Labs_CP2105...if00-port0` |
| mmWave Data | `usb-Silicon_Labs_CP2105...if01-port0` |
| RCC Board | `/dev/ttyACM0` |

### TurboPi
| Device | Port |
|---|---|
| LiDAR LD19 | `/dev/ttyUSB2` (`usb-1a86_USB_Serial`) |
| mmWave CLI | `usb-Silicon_Labs_CP2105...if00-port0` |
| mmWave Data | `usb-Silicon_Labs_CP2105...if01-port0` |

---

## SLAM (MentorPi)

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

Save map:
```bash
ros2 run nav2_map_server map_saver_cli -f parking_map
```

---

## Data Recording

```bash
cd /home/ubuntu && ros2 bag record --storage mcap \
  --max-cache-size 1000000000 \
  /scan_raw /map /odom_raw /iwr6843_pcl \
  /ascamera/camera_publisher/rgb0/image \
  /tf /tf_static \
  -o all_sensor_test_$(date +%Y%m%d_%H%M%S)
```

Transfer to laptop:
```bash
# Copy from Docker to Pi
docker cp adb8:/home/ubuntu/FOLDER_NAME /home/pi/

# Copy from Pi to Windows
scp -r pi@192.168.0.112:/home/pi/FOLDER_NAME C:\Users\USERNAME\Downloads\
```

---

## TF Tree (MentorPi)

```
map → odom → base_footprint → base_link → lidar_frame
                                         → depth_cam
                                         → imu_link
                                         → iwr6843_frame ✅
```

---

## Pending Items

| Item | Platform | Status |
|---|---|---|
| RCC board replacement | MentorPi | ❌ Need to order |
| Second LD19 LiDAR | TurboPi | ❌ Need to purchase (~$60-80) |
| mmWave driver setup | TurboPi | ❌ Pending |
| SLAM setup | TurboPi | ❌ Pending |
| TF tree | TurboPi | ❌ Pending |
| Parking lot experiment | MentorPi | ❌ Pending servo fix |
| Paper writing | Both | ❌ After experiments |

---

## Documentation

| File | Description |
|---|---|
| `MENTORPI_COMPLETE_GUIDE.md` | Full MentorPi setup guide |
| `TURBOPI_SETUP.md` | TurboPi setup guide |
| `MMWAVE_IWR6843AOP_SETUP.md` | mmWave sensor setup |
| `MMWAVE_NOTES.md` | mmWave troubleshooting notes |
| `SLAM_SETUP.md` | SLAM configuration guide |
| `FOXGLOVE_SSH_SETUP.md` | Foxglove + SSH setup |
| `HARDWARE.md` | Hardware inventory |

---

## Weekly Reports

Progress presentations (access restricted):  
[Google Drive – Weekly Reports](https://drive.google.com/drive/folders/1WDV2gzwt1jBjaSEnO-wJ-PeILImYkUhO)
