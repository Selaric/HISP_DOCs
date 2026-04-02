# mmWave IWR6843AOP — ROS2 Setup & Integration Guide
**TAI Lab | University of Michigan-Dearborn**
**Last Updated:** April 2, 2026 | **Author:** Selase

---

## Hardware

| Component | Details |
|---|---|
| Sensor | TI IWR6843AOP (ES2) |
| Connection | USB → `/dev/ttyUSB0` (command), `/dev/ttyUSB1` (data) |
| Firmware | Out-of-box firmware (pre-flashed) |
| ROS2 | Humble (Ubuntu 22.04) |

---

## Driver Used

**Repository:** https://github.com/lightinfection/TI_IWR6843AOP

This driver is a ROS2-native substitute for TI's official Out-Of-Box Demo Visualizer. It supports:
- Live point cloud publishing (`PointCloud2`)
- Real-time Range-Doppler and Range-Azimuth heatmaps
- Passthrough filters + statistical outlier removal
- DBSCAN clustering for noise removal
- 3D object tracking

> Note: TI has since released an official ROS2 driver at `git://git.ti.com/mmwave_radar/mmwave_ti_ros.git` but this driver was used for initial setup and visualization.

---

## Prerequisites — Fix Apt Package Issues First

If you get GPG key errors when running `apt install`, run this first:

```bash
# Update apt keys and package servers
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys <key_id>
sudo apt update && sudo apt upgrade -y
```

This was a recurring issue on the Pi — all ROS2 package installations were failing until the keys and servers were updated.

---

## Installation (Host / Pi)

```bash
# 1. Clone into your ROS2 workspace src folder
cd ~/ros2_ws/src
git clone https://github.com/lightinfection/TI_IWR6843AOP.git

# 2. Build
cd ~/ros2_ws
colcon build --cmake-args '-DCMAKE_BUILD_TYPE=RelWithDebInfo' --packages-select object_detection
colcon build --packages-select ti_ros2_driver
source install/setup.bash
```

---

## Launching the Sensor

### Basic point cloud only
```bash
ros2 launch ti_ros2_driver 6843aop_base.launch.py
```
Publishes: `x, y, z + SNR + radial_velocity`

### Point cloud + Heatmaps
```bash
ros2 launch ti_ros2_driver 6843aop_HeatMap.launch.py
```
Publishes point cloud AND real-time Range-Doppler + Range-Azimuth heatmaps.

### Key ROS2 Parameters

| Parameter | Default | Description |
|---|---|---|
| `command_port` | `/dev/ttyUSB0` | Config port |
| `data_port` | `/dev/ttyUSB1` | Data port |
| `frame_id` | `ti_mmwaver_0` | TF frame ID |
| `topic` | `ti_mmwave_0` | Published topic name |
| `cfg_path` | `base.cfg` | Sensor config file |
| `output_RD_heatmap` | `False` | Enable Range-Doppler heatmap |
| `output_RA_heatmap` | `False` | Enable Range-Azimuth heatmap |

---

## Viewing in RViz2

```bash
# In Docker container
docker exec -it -u ubuntu -w /home/ubuntu adb8 /bin/bash
source ros2_ws/install/setup.bash
rviz2
```

In RViz2:
1. Set **Fixed Frame** → `base_footprint`
2. Add → By Topic → `/iwr6843_pcl` → **PointCloud2**
3. Set Size to `0.1`, Style to `Points`

**Note:** If you see `discarding message because the queue is full` warnings — this is RViz2 being too slow to render. Use Foxglove Studio instead (see below).

### Add TF Transform (Required for RViz2)

```bash
ros2 run tf2_ros static_transform_publisher 0 0 0 0 0 0 base_footprint iwr6843_frame
```

---

## Viewing with Foxglove Studio (Recommended)

Foxglove is much lighter than RViz2 on the Pi and handles point cloud visualization better.

1. Download Foxglove Studio: https://foxglove.dev
2. On the robot, start the Foxglove bridge:
```bash
ros2 launch foxglove_bridge foxglove_bridge_launch.xml
```
3. In Foxglove on your Windows laptop → Open Connection → Rosbridge → `ws://<robot_ip>:8765`

---

## Sensor Verification

To verify the sensor is responding before launching ROS2:

```python
# check_sensor.py
import serial
port = serial.Serial('/dev/ttyUSB0', 115200, timeout=1)
port.write(b'version\n')
response = port.read(200)
print("Response:", response.decode())
port.close()
```

Expected output:
```
Platform        : xWR68xx
mmWave SDK Version : 03.06.02.00
Device Info     : IWR68XX QM non-secure AOP ES 02.00
RF F/W Version  : 06.03.02.06.20.08.11
✓ Sensor responding!
```

---

## ROS2 Topics Published

| Topic | Type | Description |
|---|---|---|
| `/iwr6843_pcl` | `sensor_msgs/PointCloud2` | 3D point cloud (x,y,z,SNR,velocity) |
| `/ti_mmwave_0` | `sensor_msgs/PointCloud2` | Default topic name |
| Range-Doppler heatmap | (if enabled) | Velocity vs range map |
| Range-Azimuth heatmap | (if enabled) | Range vs angle map |

---

## Recording mmWave Data

```bash
# Record mmWave + LiDAR + camera + IMU together
ros2 bag record /iwr6843_pcl /scan_raw /ascamera/camera_publisher/rgb0/image /imu /odom -o dataset_with_mmwave
```

---

## Known Issues

| Issue | Status | Fix |
|---|---|---|
| `discarding message because queue is full` | Active | Use Foxglove instead of RViz2 |
| Apt GPG key errors | Fixed | Updated apt keys and servers |
| Point cloud sparse | Active | Config file tuning needed |
| RViz2 slow on Pi | Active | Use HDMI direct + kill Chrome |

---

## References

- Driver repo: https://github.com/lightinfection/TI_IWR6843AOP
- TI official ROS2 driver: `git://git.ti.com/mmwave_radar/mmwave_ti_ros.git`
- IWR6843AOP datasheet: https://www.ti.com/product/IWR6843AOP
- Radarize paper (MobiSys 2024): https://radarize.github.io
