# System Architecture

## Overview

ALEX is a stationary humanoid robot head built on a layered architecture that separates high-level perception and behavior (Raspberry Pi 4) from real-time actuator control (Arduino Mega 2560). This separation ensures deterministic servo timing while maintaining the flexibility of ROS2 for vision processing and behavior coordination.

---

## Hardware Stack

| Layer | Component | Role |
|---|---|---|
| **Compute** | Raspberry Pi 4 (4GB) | ROS2 Humble, vision processing, behavior logic, WebSocket UI server |
| **Control** | Arduino Mega 2560 | Real-time servo command parsing, PWM generation via I2C |
| **Drive** | Adafruit PCA9685 (16-ch, I2C) | PWM signal generation for 9 servo channels |
| **Sensor** | Raspberry Pi Camera Module v2 (IMX219) | Face detection and tracking input |
| **Display** | 7" HDMI Panel + Chromium Kiosk | WebSocket-based user interface |
| **Audio** | mpg123 subprocess | Synchronized speech and sound effects |
| **Actuators** | 3× MG996R (neck) + 5× MG90S (face) + 1× MG90S (card ejector) | 9-DOF head and face movement |

---

## Communication Topology

```
┌─────────────────────────────────────────────────────────────┐
│                    Raspberry Pi 4                           │
│  Ubuntu 22.04 | ROS2 Humble | Python | C++ | Chromium Kiosk │
│                                                             │
│  ┌─────────────────┐    ┌─────────────────┐                 │
│  │ face_tracking   │───→│ head_controller │                 │
│  │     node        │    │      node       │                 │
│  └─────────────────┘    └─────────────────┘                 │
│           │                      │                          │
│           ↓                      ↓                          │
│  ┌─────────────────────────────────────────┐                │
│  │      interaction_controller_node        │                │
│  │         (behavior state machine)        │                │
│  └─────────────────────────────────────────┘                │
│                      │                                      │
│                      ↓ USB Serial (115200 baud)             │
└──────────────────────┼──────────────────────────────────────┘
                       │
                       ↓
┌─────────────────────────────────────────────────────────────┐
│               Arduino Mega 2560                             │
│         (Real-time servo control loop)                      │
│                      │                                      │
│                      ↓ I2C (SDA/SCL)                        │
└──────────────────────┼──────────────────────────────────────┘
                       │
                       ↓
┌─────────────────────────────────────────────────────────────┐
│           Adafruit PCA9685 (0x40)                           │
│              16-Channel PWM Driver                          │
│                      │                                      │
│         ┌────────────┼────────────┐                         │
│         ↓            ↓            ↓                         │
│    ┌─────────┐  ┌─────────┐  ┌─────────┐                    │
│    │Ch 0-2   │  │Ch 3-7   │  │Ch 8     │                    │
│    │Neck     │  │Face     │  │Card     │                    │
│    │MG996R×3 │  │MG90S×5  │  │Ejector  │                    │
│    └─────────┘  └─────────┘  └─────────┘                    │
└─────────────────────────────────────────────────────────────┘
```

---

## Serial Protocol Specification

**Physical Layer**: USB Serial (UART-over-USB)  
**Baud Rate**: 115200  
**Data Format**: ASCII, newline-terminated  
**Frame Structure**:
```
ch:angle,ch:angle,ch:angle,...

```

Where:
- `ch` = PCA9685 channel number (0-15)
- `angle` = target angle in degrees (0-180, mapped to PWM pulse width)
- Channels omitted retain their last commanded value

**Example Frame**:
```
0:90,1:45,2:120,3:0,4:180,5:90,6:90,7:0

```

## Power Distribution

| Rail | Voltage | Load | Source |
|---|---|---|---|
| Pi 4 Main | 5V 3A | Pi board, Camera, Display | 5V/3A adapter |
| Servo Rail | 6V 5A+ | 9 servos (MG996R peak current under load) | Dedicated 6V PSU |
| Arduino | 5V (USB) | Mega board + logic | Pi USB port |
| PCA9685 VCC | 3.3V/5V | Logic + I2C | Arduino 5V pin |
| PCA9685 V+ | 6V | Servo power pass-through | Servo rail |

**Critical Note**: The MG996R servos (neck) draw significantly more current under load than the MG90S face servos. The 6V servo rail must be sized for peak simultaneous load (3× MG996R + 6× MG90S transient). During the MACTECH exhibition, the MG996R units became noticeably warm under continuous operation, indicating the need for either higher-current PSU capacity or duty-cycle limiting in software.

---

## ROS2 Topic Map

| Topic | Type | Publisher | Subscriber | Description |
|---|---|---|---|---|
| `/face_position` | `geometry_msgs/Point` | face_tracking_node | head_controller_node | Normalized face center (x,y) |
| `/locked_face` | `std_msgs/Bool` | face_tracking_node | interaction_controller_node | Face lock status (IOU match) |
| `/servo_commands` | `std_msgs/Float32MultiArray` | head_controller_node | (internal) | Smoothed target angles for 9 channels |
| `/serial_out` | `std_msgs/String` | head_controller_node | Arduino (serial) | ASCII servo frame string |
| `/behavior_state` | `std_msgs/String` | interaction_controller_node | head_controller_node | Current behavior mode |
| `/audio_trigger` | `std_msgs/String` | interaction_controller_node | audio subprocess | Audio file path to play |
| `/rosbridge_websocket` | WebSocket | rosbridge_server | Chromium Kiosk | UI telemetry and control |

---

## Failure Modes & Mitigations

| Failure | Impact | Mitigation |
|---|---|---|
| USB serial disconnect | All servo motion stops | Watchdog timer on Arduino; safe position default |
| I2C bus lock | PWM freeze | Arduino I2C timeout + PCA9685 reset |
| Camera failure | No face tracking | Fallback to idle scan behavior (pre-programmed) |
| MG996R overheat | Neck stiffness / jitter | Thermal monitoring planned; software duty-cycle limiter under consideration |
| Power sag | System reboot | Separate servo rail; Pi on dedicated 5V supply |
| ROS2 node crash | Behavior loss | systemd auto-restart; node health monitoring under consideration |

---

## Design Trade-offs

| Decision | Alternative Considered | Rationale |
|---|---|---|
| Pi 4 + Arduino split | All-in-one Pi with GPIO PWM | Arduino provides deterministic timing; Pi GPIO is non-real-time under Linux load |
| ASCII serial protocol | Binary fixed-frame protocol | ASCII is human-readable and self-synchronizing; binary failed in testing due to desync on dropped bytes |
| Haar cascades | DNN-based detection (YOLO, MediaPipe) | Haar cascades run reliably on Pi 4 CPU with lower load; DNN would require GPU acceleration or compute upgrade |
| 3D-printed structure | Machined aluminum or carbon fiber | 3D printing enabled rapid iteration during the 5-week timeline; v1.1 may upgrade to stronger materials |
| Differential neck | Independent 3-servo neck | Differential reduces wiring through the neck joint and actuator count, at the cost of coupled kinematics |
| WebSocket UI | Native ROS2 GUI (rviz, rqt) | WebSocket allows independent UI development and kiosk deployment without ROS expertise |

---

## Future Architecture Improvements

Based on MACTECH exhibition observations:

1. **Compute Upgrade**: Raspberry Pi 4 → NVIDIA Jetson Orin Nano or Intel NUC for faster face detection and lower latency.
2. **Actuator Upgrade**: Higher-torque, higher-reliability digital servos for continuous-duty operation.
3. **Real-time Bus**: Consider CAN bus or RS-485 for multi-drop actuator networks if expanding beyond 9 channels.
4. **Sensor Fusion**: Add IMU to neck frame for closed-loop position verification and vibration compensation.
5. **Power Monitoring**: Add current/voltage monitoring on servo rail for predictive thermal management.
