# HISP Lab Handoff Plan — May 2026
**Prepared by:** Selase Doku  
**Effective:** May 8, 2026  
**Contact:** [Selase's phone/email]

---

## Context

Selase is leaving for a full-time internship in Texas on May 8. The robot platform is fully set up and ready for data collection. This document assigns clear responsibilities to each team member for the month of May.

---

## Robot Status

| Robot | Status | Responsible |
|---|---|---|
| TurboPi | ✅ Fully working — ready for data collection | Ali |
| MentorPi | ⚠️ RCC board arriving May 19 — pending steering fix | Mark |
| GalaxyRVR | ⚠️ ESP32-CAM loose connection | Ali |

---

## Mark's Tasks (Software)

### Task 1 — Remote Access Setup (By May 15)
Set up remote SSH access so Selase can connect to both robots from Texas. Options:
- Configure VPN on the lab network
- Or set up port forwarding on the lab router

Test by having Selase SSH in from outside the lab network:
```bash
ssh pi@[lab-external-ip-or-vpn] -p [port]
```
Confirm both robots are accessible remotely before May 15.

### Task 2 — TurboPi TF Transforms & SLAM (By May 19)
TurboPi is missing proper TF transforms for SLAM. Fix the transform tree:

```
map → odom → base_footprint → base_link → base_laser
                                         → iwr6843_frame
```

Install and configure slam_toolbox:
```bash
sudo apt-get install ros-humble-slam-toolbox -y
```

Launch SLAM and confirm map builds while driving:
```bash
ros2 run slam_toolbox async_slam_toolbox_node --ros-args \
  -r scan:=/scan \
  -p use_sim_time:=false \
  -p base_frame:=base_footprint \
  -p odom_frame:=odom \
  -p map_frame:=map \
  -p map_update_interval:=1.0 \
  -p transform_timeout:=0.5
```

Add SLAM to TurboPi bringup launch file once confirmed working.

### Task 3 — MentorPi RCC Board (When it arrives May 19)
1. Install new RCC board
2. Test steering with gamepad
3. Mount LD19 LiDAR back on MentorPi
4. Confirm all sensors running — LiDAR, camera, mmWave
5. Message Selase immediately when done

---

## Ali's Tasks (Data Collection)

### Task 1 — TurboPi Data Collection (May 8-16)
Collect multimodal sensor data on TurboPi in the lab and hallway.

**Step 1 — Connect to TurboPi**
```bash
ssh pi@192.168.0.102
# password: raspberry
docker exec -it -u ubuntu -w /home/ubuntu TurboPi /bin/bash
source ros2_ws/install/setup.bash
```

**Step 2 — Confirm sensors in Foxglove**
```bash
ros2 launch foxglove_bridge foxglove_bridge_launch.xml
```
Open browser: `ws://192.168.0.102:8765`
Confirm: camera ✅, LiDAR scan ✅, mmWave points ✅

**Step 3 — Record data**
```bash
cd /home/ubuntu
ros2 bag record --storage mcap --max-cache-size 1000000000 \
  /scan /iwr6843_pcl /image_raw /tf /tf_static \
  /sonar_controller/get_distance \
  -o data_$(date +%Y%m%d_%H%M%S)
```
Press **Ctrl+C** to stop.

**Step 4 — Drive routes (use WonderPi phone app)**
Record each route separately with a descriptive name:
- [ ] `hallway_empty_run1` — straight path, full hallway length
- [ ] `hallway_empty_run2` — repeat for consistency
- [ ] `hallway_empty_run3` — repeat again
- [ ] `hallway_people_run1` — hallway with people walking
- [ ] `lab_empty_run1` — drive around lab perimeter
- [ ] `lab_crowded_run1` — lab with people and objects
- [ ] `corners_run1` — multiple corner turns

Each run = 3-5 minutes. **Label every recording.**

**Step 5 — Back up to Google Drive immediately after each session**
```bash
docker cp TurboPi:/home/ubuntu/data_XXXXXXXX /home/pi/
```
Then upload to shared Google Drive folder.

### Task 2 — MentorPi Data Collection (After May 19)
Once Mark confirms MentorPi steering is fixed:
- Same recording procedure as TurboPi
- IP: `192.168.0.112`, Container: `adb8`
- Use `/scan_raw` instead of `/scan`
- Record same routes as TurboPi for comparison

### Task 3 — GalaxyRVR ESP32-CAM Fix
Fix the loose camera connection:
- Option A: Solder ESP32-CAM pins to shield (best)
- Option B: Use FTDI programmer to bypass shield
- Option C: Contact SunFounder for replacement shield

Once fixed confirm camera stream at `http://[robot-ip]/`

---

## What Good Data Looks Like

Open Foxglove and confirm all four panels:

| Panel | Topic | Should show |
|---|---|---|
| Camera | `/image_raw` | Live video feed |
| LiDAR 3D | `/scan` | Colored point cloud of room |
| mmWave 3D | `/iwr6843_pcl` | Scattered colored dots |
| Sonar Plot | `/sonar_controller/get_distance.data` | Distance graph |

If any sensor missing — restart bringup:
```bash
docker exec -it -u ubuntu -w /home/ubuntu TurboPi /bin/bash
source ros2_ws/install/setup.bash
export need_compile=False
ros2 launch bringup bringup.launch.py
```

---

## Selase's Remote Tasks (From Texas)

| Task | Timeline |
|---|---|
| Analyze collected bag files | As data arrives |
| Train ghost point mitigation model | May 15 - June 30 |
| Write paper — intro, related work, methodology | May - July |
| Generate results and figures | July |
| Complete paper draft | August |

---

## Weekly Check-in

Every **Sunday evening** — quick WhatsApp update:
- What data was collected that week
- Any issues with robots
- Photo of Foxglove showing data recording

---

## Important Rules

- **Do NOT modify any code** without checking with Selase first
- **Do NOT unplug sensors** from TurboPi
- **Label every bag file** with environment and conditions
- **Back up immediately** after every recording session
- Message Selase on WhatsApp for any issues

---

*Lead: Selase Doku | Software: Mark | Data Collection: Ali*
