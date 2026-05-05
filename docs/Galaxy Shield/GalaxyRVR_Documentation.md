# Galaxy RVR — Setup & Documentation
**TAI Lab | UM-Dearborn | PI: Prof. Xiao Zhang | Researcher: Selase**
**Date:** May 2026

---

## Platform Overview

| Item | Detail |
|---|---|
| Robot | SunFounder Galaxy RVR |
| Chassis | 6-wheel rocker-bogie (Mars rover style) |
| Brain | Arduino Uno R3 |
| Shield | Galaxy RVR Shield (SunFounder) |
| Camera | ESP32-CAM (plugged into shield) |
| Drive type | Differential — 3 wheels per side |

---

## Hardware Pin Map

| Component | Pin(s) | Notes |
|---|---|---|
| Motor Left | 2, 3 | SoftPWM required |
| Motor Right | 4, 5 | SoftPWM required |
| IR Right | 7 | Digital input |
| IR Left | 8 | Digital input |
| Camera Servo | 9 | SoftPWM — handle carefully |
| Ultrasonic | 10 | Single-wire trigger+echo |
| RGB Blue | 11 | SoftPWM |
| RGB Red | 12 | SoftPWM |
| RGB Green | 13 | SoftPWM |

---

## Libraries Required

| Library | Version | Install Method |
|---|---|---|
| SoftPWM by Brett Hagman | 1.0.1 | Arduino Library Manager |
| SunFounder AI Camera | 1.1.1 | Arduino Library Manager |
| ESP32 by Espressif Systems | 3.3.8 | Boards Manager |
| GalaxyRVR source files | — | Manual copy from lesson_codes repo |

**GalaxyRVR library path:**
```
C:\Users\NEWUSER\Documents\Arduino\libraries\GalaxyRVR\
```
Files: `car_control.h`, `car_control.cpp`, `ir_obstacle.h`, `ultrasonic.h`, `rgb.h`, `soft_servo.h`, `battery.h`

**SunFounder AI Camera fix required** — library has type mismatch with ESP32 core 3.3.8.
In `SunFounder_AI_Camera.h` change:
```cpp
// Find:
uint8_t recvBuffer[BUFFER_SIZE];
// Change to:
char recvBuffer[WS_BUFFER_SIZE];
```

---

## Upload Procedure (Every Time)

> The RVR shield uses pins 0/1 (TX/RX) which blocks uploads when attached.

1. **Remove shield** from Uno
2. Upload sketch via Arduino IDE
3. **Reattach shield**
4. Power on robot battery

**Board:** Arduino UNO
**Port:** COM5
**Baud:** 9600 (sensors) or 115200 (ESP32-CAM)

---

## Confirmed Working Sensors

### Ultrasonic (Pin 10)
- Range: ~2cm to ~400cm
- Returns 0.00 when out of range or no echo
- Tested range: confirmed reading ~205cm to wall

```cpp
#define ULTRASONIC_PIN 10

void setup() {
  Serial.begin(9600);
}

void loop() {
  delay(4);
  pinMode(ULTRASONIC_PIN, OUTPUT);
  digitalWrite(ULTRASONIC_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(ULTRASONIC_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(ULTRASONIC_PIN, LOW);
  pinMode(ULTRASONIC_PIN, INPUT);
  float duration = pulseIn(ULTRASONIC_PIN, HIGH);
  float distance = duration * 0.034 / 2;
  Serial.print("Distance: ");
  Serial.print(distance);
  Serial.println(" cm");
  delay(200);
}
```

---

### IR Obstacle Sensors (Pins 7, 8)
- Output: 1 = clear, 0 = obstacle detected
- Left sensor: pin 8, Right sensor: pin 7

```cpp
#define IR_RIGHT 7
#define IR_LEFT 8

void setup() {
  pinMode(IR_RIGHT, INPUT);
  pinMode(IR_LEFT, INPUT);
  Serial.begin(9600);
}

void loop() {
  Serial.print("Right IR: ");
  Serial.println(digitalRead(IR_RIGHT));
  Serial.print("Left IR: ");
  Serial.println(digitalRead(IR_LEFT));
  delay(200);
}
```

---

### RGB LEDs (Pins 11, 12, 13)
- Requires SoftPWM
- All three colors confirmed: Red, Green, Blue, White

```cpp
#include <SoftPWM.h>

const int bluePin = 11;
const int redPin = 12;
const int greenPin = 13;

void setup() {
  SoftPWMBegin();
}

void loop() {
  // Red
  SoftPWMSet(redPin, 255); SoftPWMSet(greenPin, 0); SoftPWMSet(bluePin, 0);
  delay(1000);
  // Green
  SoftPWMSet(redPin, 0); SoftPWMSet(greenPin, 255); SoftPWMSet(bluePin, 0);
  delay(1000);
  // Blue
  SoftPWMSet(redPin, 0); SoftPWMSet(greenPin, 0); SoftPWMSet(bluePin, 255);
  delay(1000);
}
```

---

### 6-Wheel Drive (Pins 2, 3, 4, 5)
- Requires SoftPWM
- Power range: 0–100 via `car_control.h` functions, or 0–255 via raw SoftPWMSet

**Functions (car_control.h):**

| Function | Description |
|---|---|
| `carBegin()` | Initialize motors |
| `carForward(power)` | Drive forward, power 0–100 |
| `carBackward(power)` | Drive backward, power 0–100 |
| `carTurnLeft(power)` | Turn left, power 0–100 |
| `carTurnRight(power)` | Turn right, power 0–100 |
| `carStop()` | Stop all motors |

```cpp
#include <SoftPWM.h>
#include "car_control.h"

void setup() {
  SoftPWMBegin();
  carBegin();
  delay(1000);
}

void loop() {
  carForward(60);  delay(2000);
  carStop();       delay(1000);
  carTurnLeft(60); delay(1000);
  carStop();       delay(1000);
  carTurnRight(60);delay(1000);
  carStop();       delay(3000);
}
```

---

## Known Issues

| Issue | Status | Notes |
|---|---|---|
| Shield blocks upload | ⚠️ Workaround | Remove shield before every upload |
| ESP32-CAM USB chip | ❌ Faulty | Controller board USB not recognized — VID_0000 |
| Brittle connections | ⚠️ Ongoing | Shield seating causes intermittent motor issues |
| Camera servo twitching | ⚠️ Known | SoftPWM init bleeds into pin 9 — set LOW explicitly |

---

## ESP32-CAM Status

The ESP32-CAM is physically installed on the shield but cannot be flashed due to a faulty USB chip on the controller board (Device Descriptor Request Failed, VID_0000&PID_0002).

**Attempted methods:**
- Direct USB-C from controller board → laptop — failed (no device enumeration)
- Flash through Uno via COM5 — failed (port conflict / no response)
- CH340 driver install — did not resolve (chip not identified)

**Planned fix:** Replace controller board or use FTDI programmer directly on ESP32-CAM pins.

**AP Mode config (ready to flash when hardware fixed):**
```cpp
#define WIFI_MODE WIFI_MODE_AP
#define SSID "GalaxyRVR"
#define PASSWORD "12345678"
#define NAME "GalaxyRVR"
#define TYPE "AiCamera"
#define PORT "8765"
```

---

## Next Steps

| Priority | Task |
|---|---|
| 1 | Stream all sensor data simultaneously to laptop via serial |
| 2 | Fix ESP32-CAM flashing — replace controller board or use FTDI |
| 3 | Install Raspberry Pi companion board for LiDAR + mmWave |
| 4 | Connect LD19 LiDAR via SoftwareSerial |
| 5 | Pull sensor data into ROS2 topics on laptop |
| 6 | Integrate mmWave IWR6843AOP (requires Pi — 2 serial ports needed) |

---

## Lab Context

This robot is part of the HISP (High-precision Indoor Smart Parking) research project.
The Galaxy RVR serves as a secondary testbed platform alongside the MentorPi A1.
Primary sensor fusion research (LiDAR + Camera + mmWave) runs on the MentorPi A1 with Raspberry Pi 5 + ROS2 Humble.

**GitHub:** https://github.com/Selaric/HISP_DOCs
