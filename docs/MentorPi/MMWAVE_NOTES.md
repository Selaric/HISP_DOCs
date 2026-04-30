# mmWave Radar — Integration Notes
**Sensor:** TI IWR6843AOP  
**Platform:** MentorPi A1 (ROS2 Humble)  
**Researcher:** Selase Eric Doku, Ali Mheidly 
**PI:** Xiao Zhang  
**Last Updated:** March 2026

---

## 1. What is mmWave?

mmWave radar operates at 76–81 GHz (millimeter wave frequency). Unlike LiDAR (laser) or camera (light), mmWave uses radio waves to sense the environment.

**Key capabilities:**
- Measures **range** (distance to object)
- Measures **velocity** (Doppler effect — how fast object is moving)
- Measures **angle** (direction of object)
- Works through **fog, dust, smoke, rain, and low light** — conditions where LiDAR and camera fail
- Can detect **through thin walls** in some configurations

---

## 2. Sensor: TI IWR6843AOP

| Parameter | Value |
|---|---|
| Model | IWR6843AOP (Antenna on Package) |
| Frequency | 76–81 GHz FMCW |
| Interface | USB (2 ports) |
| Linux device | `/dev/ttyUSB1` (data), `/dev/ttyUSB2` (config) |
| ROS2 output | `PointCloud2` |

**AOP = Antenna on Package** — the antenna is integrated directly into the chip, making it compact and easy to mount on a robot.

---

## 3. Comparison: mmWave vs LiDAR vs Camera

| Capability | LiDAR | Camera | mmWave |
|---|---|---|---|
| Range measurement | ✅ High res | ❌ Indirect | ✅ Direct |
| Velocity measurement | ❌ | ❌ | ✅ Doppler |
| Works in darkness | ✅ | ❌ | ✅ |
| Works in fog/rain | ❌ | ❌ | ✅ |
| Semantic understanding | ❌ | ✅ | ❌ |
| Resolution | High | Very High | Low |
| Cost | High | Low | Medium |
| ROS2 topic type | LaserScan / PointCloud2 | Image | PointCloud2 |

**Key insight:** mmWave fills the gaps that LiDAR and camera leave — especially in adverse weather and for velocity sensing. This makes it ideal for outdoor or real-world autonomous navigation.

---

## 4. Physical Setup

**Connections:**
- USB1 → Raspberry Pi (data port)
- USB2 → Raspberry Pi (config port)

**Linux detection:**
```bash
ls /dev/ttyUSB*
# Returns: /dev/ttyUSB1  /dev/ttyUSB2
```

**Give USB permissions:**
```bash
sudo chmod 666 /dev/ttyUSB1
sudo chmod 666 /dev/ttyUSB2
```

---

## 5. ROS2 Driver Installation

**Driver used:** `mmwave_ti_ros` (kimsooyoung)

**Clone (on Pi host, outside Docker):**
```bash
git clone https://github.com/kimsooyoung/mmwave_ti_ros.git
docker cp ~/mmwave_ti_ros MentorPi:/home/ubuntu/ros2_ws/src/
```

**Build (inside Docker):**
```bash
cd ~/ros2_ws
source install/setup.bash
colcon build --packages-select serial
source install/setup.bash
colcon build --packages-select ti_mmwave_ros2_interfaces
source install/setup.bash
colcon build --packages-select ti_mmwave_ros2_pkg
source install/setup.bash
```

**Launch:**
```bash
ros2 launch ti_mmwave_ros2_pkg eloquent_composition.launch.py
```

---

## 6. ROS2 Topics (Expected)

| Topic | Type | Description |
|---|---|---|
| `/ti_mmwave/radar_scan` | Custom msg | Individual detected points with range, velocity, angle |
| `/ti_mmwave/radar_scan_pcl` | PointCloud2 | Point cloud for RViz2 visualization |

---

## 7. Visualizing in RViz2

1. Launch mmWave node
2. Open RViz2: `rviz2`
3. Add → By topic → `/ti_mmwave/radar_scan_pcl` → PointCloud2
4. Set Fixed Frame to `ti_mmwave`
5. You should see detected points appear as a point cloud

---

## 8. Current Status

| Task | Status |
|---|---|
| Sensor physically connected | ✅ |
| Linux USB detection | ✅ `/dev/ttyUSB1` |
| ROS2 driver built | ✅ |
| Node launches | ✅ |
| Point cloud data visible | 🔧 In progress — configuration tuning needed |

---

## 9. Next Steps

- Tune sensor configuration file for IWR6843AOP
- Verify point cloud data publishing on `/ti_mmwave/radar_scan_pcl`
- Visualize alongside LiDAR scan in RViz2
- Compare mmWave point cloud vs LiDAR scan in same environment
- Document sensor fusion potential for research direction

---

## 10. Research Relevance

The IWR6843AOP directly supports the lab's research direction on **Integrated Sensing and Communication (ISAC)**. By combining mmWave radar with LiDAR and camera on the MentorPi platform, we can:

- Study multimodal sensor fusion for robust environment perception
- Explore all-weather autonomous navigation
- Investigate velocity-aware obstacle detection
- Build a foundation for V2X (Vehicle-to-Everything) research scenarios
