# DC Motor Speed and Position Controller for Arduino UNO Q

## Overview

This project demonstrates a complete DC motor control system using the Arduino UNO Q, a DRV8871 H-Bridge motor driver, and a quadrature encoder.

The project includes:

* Open-loop PWM motor control
* Quadrature encoder monitoring
* Motor RPM and gearbox output RPM calculation
* Closed-loop speed PID control
* Direct position PID control
* Trapezoidal motion profile with speed PID
* Live browser-based trend graphs
* MPU-to-MCU communication using Arduino RouterBridge RPC
* Web UI using Arduino App Lab WebUI and Socket.IO

The project is useful for learning motor control, encoder feedback, PID tuning, and motion profiling.

---

## System Architecture

```text
Browser Web UI
     ↓
Socket.IO
     ↓
Python App on MPU
     ↓
RouterBridge RPC
     ↓
STM32U585 MCU
     ↓
PWM Signals
     ↓
DRV8871 H-Bridge
     ↓
DC Motor + Gearbox
     ↓
Quadrature Encoder Feedback
     ↓
MCU Control Loop
```

---

## Hardware

### Controller

* Arduino UNO Q

### Motor Driver

* DRV8871 H-Bridge motor driver

### Motor

* Brushed DC motor with quadrature encoder
* Encoder CPR: 44
* Gearbox ratio: 162.75:1

### Encoder Output CPR

```text
Output CPR = Motor Encoder CPR × Gearbox Ratio

Output CPR = 44 × 162.75 = 7161 counts/output revolution
```

---

## Wiring

### DRV8871

| UNO Q Pin | DRV8871 Pin |
| --------- | ----------- |
| D9        | IN1         |
| D6        | IN2         |
| GND       | GND         |

### Encoder

| Encoder Signal | UNO Q Pin              |
| -------------- | ---------------------- |
| Channel A      | D2                     |
| Channel B      | D3                     |
| GND            | GND                    |
| VCC            | Encoder supply voltage |

Make sure the encoder output voltage is compatible with the UNO Q MCU input pins.

---

## Project Structure

```text
dc_motor_controller/
│
├── index.html
├── app.js
├── style.css
│
├── python/
│   └── main.py
│
├── sketch/
│   └── sketch.ino
│
└── README.md
```

---

## Control Modes

## 1. Open-Loop Motor Control

The user directly commands motor PWM.

```text
PWM range: -255 to +255
```

Meaning:

```text
+255 = full forward
0    = stop
-255 = full reverse
```

This mode disables all closed-loop controllers.

---

## 2. Encoder Live Monitor

The MCU decodes the quadrature encoder using interrupts.

Displayed in the browser:

* Encoder count
* Direction
* Raw speed in counts/second
* Filtered speed in counts/second
* Motor RPM
* Output shaft RPM
* Output shaft angle

---

## 3. Closed-Loop Speed PID

This mode controls output shaft speed.

The user commands:

```text
Target output RPM
```

The MCU compares target speed to measured speed and adjusts PWM using PID.

```text
Target RPM
    ↓
Speed PID
    ↓
PWM
    ↓
DRV8871
    ↓
Motor
    ↓
Encoder speed feedback
```

Tunable gains:

```text
Kp
Ki
Kd
```

Recommended starting values:

```text
Kp = 0.05
Ki = 0.10
Kd = 0.00
```

---

## 4. Direct Position PID

This mode controls position directly.

The user commands:

```text
Target output angle in degrees
```

The MCU converts target angle to encoder counts:

```text
target_counts = target_degrees / 360 × 7161
```

Then the position PID directly generates PWM.

```text
Target Angle
    ↓
Position Error
    ↓
Position PID
    ↓
PWM
    ↓
DRV8871
    ↓
Motor
    ↓
Encoder position feedback
```

Tunable gains:

```text
Kp
Ki
Kd
```

Recommended starting values:

```text
Kp = 0.20
Ki = 0.00
Kd = 0.01
```

This mode is useful for simple position experiments, but it may move aggressively for large position changes.

---

## 5. Trapezoidal Motion Profile + Speed PID

This is the preferred position-control mode for smoother motion.

Instead of driving PWM directly from position error, this mode generates a smooth target speed profile.

```text
Target Angle
    ↓
Trapezoidal Motion Profile
    ↓
Target RPM
    ↓
Speed PID
    ↓
PWM
    ↓
DRV8871
    ↓
Motor
```

The trapezoidal profile has three phases:

```text
Accelerate
Cruise
Decelerate
```

Speed shape:

```text
RPM

     ________
    /        \
   /          \
__/            \__
```

The user controls:

```text
Target angle
Maximum RPM
Acceleration in RPM/s
```

Recommended starting values:

```text
Max RPM = 20
Acceleration = 30 RPM/s
```

This mode gives smoother starts and stops and is better for robotics, puppets, steering mechanisms, and mechanical systems with gears.

---

## Live Trend Graphs

The web interface includes live trend graphs for:

### Speed PID

* Target RPM
* Actual output RPM
* Speed PID PWM

### Direct Position PID

* Target angle
* Actual angle
* Position PID PWM

### Trapezoidal Profile

* Target angle
* Actual angle
* Generated target RPM

These graphs make PID tuning and motion-profile tuning much easier.

---

## MCU RPC Services

### Motor

```text
motor/set
motor/stop
motor/get
```

### Encoder

```text
encoder/count
encoder/dir
encoder/raw_speed
encoder/filtered_speed
encoder/motor_rpm
encoder/output_rpm
encoder/reset
```

### Speed PID

```text
pid/enable
pid/disable
pid/set_target_output_rpm
pid/get_target_output_rpm
pid/get_enabled
pid/get_output_pwm
pid/set_kp
pid/set_ki
pid/set_kd
```

### Direct Position PID

```text
position/enable
position/disable
position/set_target_deg
position/get_target_deg
position/get_target_counts
position/get_angle_x10
position/get_enabled
position/get_output_pwm
position/set_kp
position/set_ki
position/set_kd
```

### Trapezoidal Profile

```text
profile/enable
profile/disable
profile/set_target_deg
profile/get_target_deg
profile/get_target_counts
profile/get_enabled
profile/get_target_rpm
profile/get_velocity_rpm
profile/get_error_counts
profile/set_max_rpm
profile/set_accel
profile/get_max_rpm
profile/get_accel
```

---

## Web UI Sections

The browser interface includes:

1. Open-loop motor control
2. Encoder live data
3. Closed-loop speed control
4. Live speed trend graph
5. Direct position PID
6. Live position trend graph
7. Trapezoidal profile + speed PID
8. Live profile trend graph
9. Status display

---

## Important UNO Q Note

During development, the motor driver input pins worked best when controlled consistently with:

```cpp
analogWrite()
```

For logic-low and logic-high states:

```cpp
analogWrite(pin, 0);
analogWrite(pin, 255);
```

This avoided issues caused by mixing:

```cpp
digitalWrite()
```

and:

```cpp
analogWrite()
```

on the same PWM-capable pins.

---

## Running the Project

### 1. Upload MCU Sketch

Upload:

```text
sketch/sketch.ino
```

to the UNO Q MCU.

The serial monitor should show:

```text
DC Motor Controller with Trapezoid Profile ready.
```

---

### 2. Start the MPU App

Run the Arduino App Lab project.

The Python application uses:

```python
from arduino.app_utils import *
from arduino.app_bricks.web_ui import WebUI
```

---

### 3. Open the Browser

Open the UNO Q web interface from your computer browser.

Example:

```text
http://<uno-q-ip-address>
```

---

## Suggested Testing Order

### Step 1: Encoder Test

Turn the motor shaft slowly by hand.

Verify:

* Encoder count changes
* Direction changes
* Output angle changes

---

### Step 2: Open Loop Test

Use PWM slider:

```text
+80
+120
-80
-120
0
```

Verify motor direction and encoder feedback.

---

### Step 3: Speed PID Test

Start with:

```text
Target RPM = 5
Kp = 0.05
Ki = 0.10
Kd = 0.00
```

Then try:

```text
10 RPM
20 RPM
```

Watch the speed trend graph.

---

### Step 4: Direct Position PID Test

Start with small moves:

```text
10°
30°
45°
```

Use:

```text
Kp = 0.20
Ki = 0.00
Kd = 0.01
```

---

### Step 5: Trapezoidal Profile Test

Start with:

```text
Target = 90°
Max RPM = 10
Acceleration = 20 RPM/s
```

Then test:

```text
180°
360°
-90°
```

Watch the generated target RPM and actual position trend.

---

## Tuning Tips

### Speed PID

If motor is slow to respond:

```text
Increase Kp
```

If motor does not reach target RPM:

```text
Increase Ki
```

If motor oscillates:

```text
Reduce Kp or Ki
```

Usually start with:

```text
Kd = 0
```

---

### Direct Position PID

If position moves too slowly:

```text
Increase Kp
```

If it overshoots:

```text
Reduce Kp
```

If it oscillates near target:

```text
Increase Kd slightly
```

Keep:

```text
Ki = 0
```

at first.

---

### Trapezoidal Profile

If motion is too slow:

```text
Increase Max RPM
```

If acceleration is too gentle:

```text
Increase Acceleration
```

If motion jerks or overshoots:

```text
Lower Max RPM
Lower Acceleration
Tune Speed PID first
```

---

## Future Improvements

Possible future enhancements:

* STM32 hardware timer encoder mode
* S-curve motion profile
* Position limits
* Soft end stops
* Homing switch
* Emergency stop input
* Multiple motor support
* Data logging
* CSV export
* Auto PID tuning
* Command scripting
* ROS2 integration

---

## Educational Topics Covered

This project demonstrates:

* H-Bridge motor driving
* Quadrature encoder feedback
* Interrupt-based encoder decoding
* RPM measurement
* Digital low-pass filtering
* PID speed control
* PID position control
* Cascaded position-speed control
* Trapezoidal motion profiling
* Browser-based embedded control
* Arduino UNO Q MPU/MCU cooperation
* RouterBridge RPC communication

---

## License

Educational motor-control demonstration project for Arduino UNO Q, DRV8871, and quadrature encoder based DC motors.
