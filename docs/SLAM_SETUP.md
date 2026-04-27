# SLAM Setup Guide — LiDAR Mapping with slam_toolbox
**TAI Lab | University of Michigan-Dearborn**
**Last Updated:** April 27, 2026 | **Author:** Selase

---

## Overview

This guide documents how we set up real-time SLAM (Simultaneous Localization and Mapping) on the robot using `slam_toolbox` with the LD19 LiDAR. The robot is driven manually with the gamepad while the map builds automatically in Foxglove.

---

## What is SLAM?

SLAM builds a map of the environment while simultaneously tracking the robot's location within it. As you drive the robot:
- LiDAR scans the environment
- slam_toolbox matches scans to build a map
- The map updates in real time in Foxglove
- The robot's position is tracked on the map

---

## Installation

Inside Docker:

```bash
docker exec -it -u ubuntu -w /home/ubuntu adb8 /bin/bash
sudo apt install ros-humble-slam-toolbox -y
source ros2_ws/install/setup.bash
```

---

## Terminal Setup — Every Session

You need **3 terminals** all inside Docker:

### Terminal 1 — Bringup (LiDAR + all sensors)
```bash
docker exec -it -u ubuntu -w /home/ubuntu adb8 /bin/bash
source ros2_ws/install/setup.bash
# Bringup launches automatically on boot — verify with:
ros2 topic hz /scan_raw  # Should show 10Hz
```

### Terminal 2 — Launch slam_toolbox
```bash
docker exec -it -u ubuntu -w /home/ubuntu adb8 /bin/bash
source ros2_ws/install/setup.bash
ros2 run slam_toolbox async_slam_toolbox_node --ros-args \
  -r scan:=/scan_raw \
  -p use_sim_time:=false \
  -p base_frame:=base_footprint \
  -p odom_frame:=odom \
  -p map_frame:=map \
  -p map_update_interval:=1.0 \
  -p transform_timeout:=0.5
```

Expected output:
```
[slam_toolbox]: Using solver plugin solver_plugins::CeresSolver
[slam_toolbox]: CeresSolver: Using SCHUR_JACOBI preconditioner.
Registering sensor: [Custom Described Lidar]
```

### Terminal 3 — Record bag file (mcap format)
```bash
docker exec -it -u ubuntu -w /home/ubuntu adb8 /bin/bash
source ros2_ws/install/setup.bash
ros2 bag record --storage mcap \
  /scan_raw /map /odom /iwr6843_pcl \
  -o slam_run_$(date +%Y%m%d_%H%M%S)
```

Press **Ctrl+C** to stop recording when done.

### Terminal 4 — Foxglove bridge
```bash
docker exec -it -u ubuntu -w /home/ubuntu adb8 /bin/bash
source ros2_ws/install/setup.bash
ros2 launch foxglove_bridge foxglove_bridge_launch.xml
```

---

## Foxglove Visualization Setup

1. Open Foxglove Studio on laptop
2. Connect to `ws://raspberrypi:8765`
3. Add **3D panel**
4. In 3D panel settings → Add topic → `/map`
5. Add topic → `/scan_raw` (LaserScan)
6. Add topic → `/iwr6843_pcl` (PointCloud2)
7. Set Fixed Frame to `map`

The map builds as you drive — white = free space, black = walls/obstacles, grey = unknown.

---

## Save the Map After Driving

When done driving, save the map as an image file:

```bash
ros2 run nav2_map_server map_saver_cli -f parking_map
```

This saves:
- `parking_map.pgm` — map image (black = wall, white = free, grey = unknown)
- `parking_map.yaml` — map metadata (resolution, origin)

---

## Transfer Recording to Laptop

### Step 1 — Copy from Docker to Pi host
```bash
# On Pi host (not Docker)
docker cp adb8:/home/ubuntu/slam_run_YYYYMMDD_HHMMSS /home/pi/slam_run_YYYYMMDD_HHMMSS
```

### Step 2 — Copy from Pi to Windows laptop
```bash
# On Windows PowerShell
scp -r pi@192.168.0.112:/home/pi/slam_run_YYYYMMDD_HHMMSS C:\Users\YourName\Downloads\
```

### Step 3 — Open in Foxglove
- Open Foxglove Studio
- File → Open → select the mcap file inside the folder
- Hit play to replay the full mapping session
- Scrub through timeline to see map building frame by frame

---

## Key Topics

| Topic | Type | Description |
|---|---|---|
| `/scan_raw` | LaserScan | LiDAR scan at 10Hz |
| `/map` | OccupancyGrid | SLAM map (updates every 1 second) |
| `/odom` | Odometry | Robot position and velocity |
| `/iwr6843_pcl` | PointCloud2 | mmWave point cloud at 30Hz |
| `/slam_toolbox/scan_visualization` | LaserScan | What slam_toolbox is processing |

---

## Troubleshooting

| Problem | Cause | Fix |
|---|---|---|
| `/map` not publishing | slam_toolbox not receiving scan | Check `ros2 topic info /scan_raw` — subscription count must be 1 |
| Map not updating | Robot not moving | Drive with gamepad — map only updates when robot moves |
| `Message Filter dropping` warning | TF timing issue | Add `-p transform_timeout:=0.5` to launch command |
| slam_toolbox subscribes to `/scan` not `/scan_raw` | Default config | Always use `-r scan:=/scan_raw` remap |
| Multiple slam_toolbox instances | Old instance still running | `pkill -f slam_toolbox` before relaunching |

---

## TF Tree

slam_toolbox requires these transforms to exist:

```
odom → base_footprint → base_link → lidar_frame
```

Verify with:
```bash
ros2 run tf2_ros tf2_echo odom base_footprint
ros2 run tf2_ros tf2_echo base_link lidar_frame
```

Both must return valid transforms before slam_toolbox will publish the map.

---

## Research Context

This SLAM implementation is part of a comparative study:
- **Condition 1** — LiDAR only SLAM (current)
- **Condition 2** — LiDAR + mmWave fused SLAM (next phase)

The goal is to demonstrate that mmWave augmentation improves SLAM map quality in indoor parking environments where LiDAR struggles with reflective surfaces and low light.

---

## References

- slam_toolbox: https://github.com/SteveMacenski/slam_toolbox
- ROS2 Humble slam_toolbox docs: https://docs.nav2.org/tutorials/docs/navigation2_with_slam.html
