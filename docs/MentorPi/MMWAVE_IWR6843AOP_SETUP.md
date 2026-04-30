# mmWave IWR6843AOP — Complete Setup Guide
**TAI Lab | University of Michigan-Dearborn**
**Last Updated:** April 20, 2026 | **Author:** Selase

---

## Hardware

| Component | Details |
|---|---|
| Sensor | TI IWR6843AOP (ES2) — xWR68xx platform |
| Firmware | Out-of-box (pre-flashed, SDK 03.06.02.00) |
| Connection | 1 USB cable → creates 2 virtual serial ports (CP2105 chip) |
| ROS2 | Humble (Ubuntu 22.04, inside Docker container) |

---

## Port Mapping

The IWR6843AOP uses a **Silicon Labs CP2105 Dual UART Bridge** — one USB cable creates two ports:

| Port | Interface | Purpose | Baud Rate |
|---|---|---|---|
| `/dev/ttyUSB0` (or shifts) | CP2105 if00 | **Command** — sends config to sensor | 115200 |
| `/dev/ttyUSB2` (or shifts) | CP2105 if01 | **Data** — receives point cloud output | 921600 |

> ⚠️ **Port numbers shift every time the sensor is unplugged or the system reboots.** Always verify with:
> ```bash
> ls -la /dev/serial/by-id/
> ```
> Look for `Silicon_Labs_CP2105` entries — `if00` = command, `if01` = data.

---

## Driver Used

**Repository:** https://github.com/nhma20/iwr6843aop_pub

Located at: `~/ros2_ws/src/iwr6843aop_pub/`

This is a Python-based ROS2 publisher that reads the sensor and publishes to `/iwr6843_pcl` as `sensor_msgs/PointCloud2`.

---

## Config File

**Location:** `~/ros2_ws/src/mmwave_ti_ros/ti_mmwave_ros2_pkg/cfg/6843AOP_3d.cfg`

This is the correct config for the IWR6843AOP in 3D mode. It was modified with one critical addition — `bpmCfg` — which is **required for SDK 3.06** but missing from the original SDK 3.02 config.

### Critical Fix — bpmCfg

Without `bpmCfg -1 0 0 1`, `sensorStart` returns:
```
Error: Full configuration must be provided before sensor can be started the first time
```

The fix was added permanently:
```bash
sed -i '/extendedMaxVelocity/a bpmCfg -1 0 0 1' ~/ros2_ws/src/mmwave_ti_ros/ti_mmwave_ros2_pkg/cfg/6843AOP_3d.cfg
```

The config now includes (in order):
```
sensorStop
flushCfg
dfeDataOutputMode 1
channelCfg 15 7 0
adcCfg 2 1
adcbufCfg -1 0 1 1 1
profileCfg 0 60 43 7 40 0 0 100 1 224 7000 0 0 30
chirpCfg 0 0 0 0 0 0 0 1
chirpCfg 1 1 0 0 0 0 0 2
chirpCfg 2 2 0 0 0 0 0 4
frameCfg 0 2 16 0 33.333 1 0
lowPower 0 0
guiMonitor -1 1 0 0 0 0 0
cfarCfg -1 0 2 8 4 3 0 15 1
cfarCfg -1 1 0 4 2 3 1 15 1
multiObjBeamForming -1 1 0.5
clutterRemoval -1 1
calibDcRangeSig -1 0 -5 8 256
extendedMaxVelocity -1 0
bpmCfg -1 0 0 1          ← CRITICAL — required for SDK 3.06
lvdsStreamCfg -1 0 0 0
compRangeBiasAndRxChanPhase 0.0 1 0 -1 0 1 0 -1 0 1 0 -1 0 1 0 -1 0 1 0 -1 0 1 0 -1 0
measureRangeBiasAndRxChanPhase 0 1.5 0.2
CQRxSatMonitor 0 3 4 99 0
CQSigImgMonitor 0 111 4
analogMonitor 0 0
aoaFovCfg -1 -90 90 -90 90
cfarFovCfg -1 0 0 8.40
cfarFovCfg -1 1 -5.02 5.02
sensorStart
```

---

## Driver Fix — publisher_member_function.py

The original driver had a bug where it crashed when the data port had no data yet. Fixed with:

```bash
# Fix 1 — read at least 1 byte instead of 0
sed -i 's/byte_buffer = self.data_port.read(self.data_port.in_waiting)/byte_buffer = self.data_port.read(self.data_port.in_waiting or 64)/' ~/ros2_ws/src/iwr6843aop_pub/iwr6843aop_pub/publisher_member_function.py

# Fix 2 — increase timeout before giving up
sed -i 's/if(self.warn>100):/if(self.warn>10000):/' ~/ros2_ws/src/iwr6843aop_pub/iwr6843aop_pub/publisher_member_function.py
```

Then rebuild:
```bash
cd ~/ros2_ws && colcon build --packages-select iwr6843aop_pub && source install/setup.bash
```

---

## Step-by-Step Launch Sequence

Every session follow this exact sequence:

### Step 1 — Enter Docker
```bash
docker exec -it -u ubuntu -w /home/ubuntu adb8 /bin/bash
source ros2_ws/install/setup.bash
```

### Step 2 — Verify sensor is connected and find ports
```bash
ls -la /dev/serial/by-id/
```

Look for:
```
usb-Silicon_Labs_CP2105_...-if00-port0 → ../../ttyUSBX   ← command port
usb-Silicon_Labs_CP2105_...-if01-port0 → ../../ttyUSBY   ← data port
```

Also verify sensor is responding:
```bash
python3 -c "
import serial
port = serial.Serial('/dev/ttyUSBX', 115200, timeout=1)
port.write(b'version\n')
import time; time.sleep(1)
print(port.read(500).decode(errors='ignore'))
port.close()
"
```
Expected: `Platform: xWR68xx`, `mmWave SDK Version: 03.06.02.00`, `✓ Sensor responding`

### Step 3 — Launch mmWave driver
Replace `ttyUSBX` with command port and `ttyUSBY` with data port from Step 2:

```bash
ros2 run iwr6843aop_pub pcl_pub --ros-args \
  -p cfg_path:=/home/ubuntu/ros2_ws/src/mmwave_ti_ros/ti_mmwave_ros2_pkg/cfg/6843AOP_3d.cfg \
  -p cli_port:=/dev/ttyUSBX \
  -p data_port:=/dev/ttyUSBY
```

Expected output:
```
sensorStart
[INFO] [iwr6843_pcl_pub]: Publishing 7 points
[INFO] [iwr6843_pcl_pub]: Publishing 4 points
...
```

### Step 4 — Verify topic is publishing
In another terminal (inside Docker):
```bash
ros2 topic hz /iwr6843_pcl
```
Expected: `average rate: 30.0 Hz`

### Step 5 — Launch Foxglove bridge
```bash
ros2 launch foxglove_bridge foxglove_bridge_launch.xml
```

### Step 6 — Connect Foxglove on laptop
Open Foxglove Studio → Open Connection → Foxglove WebSocket → `ws://raspberrypi:8765`

Add topic `/iwr6843_pcl` as PointCloud2 in the 3D panel.

---

## Sensor Parameters

| Parameter | Value |
|---|---|
| Frequency | 60 GHz |
| Max Range | 9.02 m |
| Range Resolution | 0.047 m |
| Max Velocity | ±4.99 m/s |
| Frame Rate | 30 Hz |
| Frame ID | `iwr6843_frame` |
| Published Topic | `/iwr6843_pcl` |
| Message Type | `sensor_msgs/PointCloud2` |
| Fields | x, y, z |

---

## Full Data Pipeline — Record All Sensors

With mmWave, LiDAR, and Camera all running:

```bash
ros2 bag record /iwr6843_pcl /scan_raw /ascamera/camera_publisher/rgb0/image /imu /odom -o full_sensor_dataset
```

---

## Port Conflict Warning

The LiDAR (LD19) also uses a USB serial port. When both LiDAR and mmWave are connected:

- LiDAR appears on `ttyUSB1` (CH340 chip, `1a86:7523`)
- mmWave appears on two ports (CP2105 chip, `10c4:ea70`)

Check with:
```bash
lsusb
ls -la /dev/serial/by-id/
```

Always confirm port assignments before launching — **do not guess**.

---

## Known Issues

| Issue | Cause | Fix |
|---|---|---|
| `sensorStart` fails with "Full configuration" error | Missing `bpmCfg` command for SDK 3.06 | Already fixed in config file |
| Port numbers shift after replug | Linux assigns USB ports dynamically | Always check `/dev/serial/by-id/` |
| 0 points published | `clutterRemoval -1 1` filters static objects | Wave hand in front of sensor to see detections |
| Driver crashes after 100 empty reads | Bug in original driver | Already fixed in source |

---

## References

- Driver: https://github.com/nhma20/iwr6843aop_pub
- TI IWR6843AOP product page: https://www.ti.com/product/IWR6843AOP
- Config reference: TI mmWave SDK 3.x User Guide
- Radarize paper (MobiSys 2024): https://radarize.github.io
