# ROS2 Node Reference

## Node Overview

| Node | Package | Language | Purpose |
|---|---|---|---|
| `face_tracking_node` | `alex_bringup` | Python | Vision processing, face detection, IOU tracking |
| `head_controller_node` | `alex_bringup` | Python | Motion planning, smoothing, serial output |
| `interaction_controller_node` | `alex_bringup` | Python | Behavior state machine, audio coordination |

---

## face_tracking_node

### Purpose
Detects human faces using Haar cascades, tracks them across frames with IOU matching, and publishes the selected target's normalized position for head tracking.

### Subscribed Topics
None (reads directly from Raspberry Pi Camera Module v2 via OpenCV VideoCapture)

### Published Topics

| Topic | Type | Description |
|---|---|---|
| `/face_position` | `geometry_msgs/Point` | Normalized face center coordinates (x: [-1,1], y: [-1,1], z: estimated depth proxy) |
| `/locked_face` | `std_msgs/Bool` | Face lock status (IOU match) |

### Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `camera_index` | int | 0 | OpenCV camera device index (Pi Camera v2) |
| `haar_model_path` | string | `"/usr/share/opencv4/haarcascades/haarcascade_frontalface_default.xml"` | Cascade classifier model file |
| `detection_min_size` | tuple | (80, 80) | Minimum face size in pixels (width, height) |
| `iou_threshold` | float | 0.3 | Minimum IOU for face track matching across frames |
| `lock_min_duration` | float | 2.0 | Minimum seconds to maintain lock before allowing switch |
| `lock_loss_timeout` | float | 6.0 | Seconds without detection before releasing lock |
| `image_width` | int | 640 | Capture resolution width |
| `image_height` | int | 480 | Capture resolution height |

### Key Design Decisions

- **Haar cascades chosen over DNN**: Lower CPU load on Raspberry Pi 4. Trade-off is lower accuracy in poor lighting and false positives on profile faces. This was acceptable for the exhibition environment with controlled booth lighting, but a future compute upgrade would enable DNN-based detection.
- **IOU matching over Kalman filtering**: Simpler to tune and sufficient for slow-moving exhibition crowds. Kalman filtering would be warranted for faster target motion or mobile platforms.
- **Score-based selection with hysteresis**: Prevents rapid switching between two faces of similar size and distance. The 1.2× score threshold ensures confident switches only.

---

## head_controller_node

### Purpose
Receives face position targets, applies exponential smoothing and velocity limiting, maps Cartesian face coordinates to servo angles, and outputs ASCII servo frames to the Arduino Mega via serial.

### Subscribed Topics

| Topic | Type | Publisher | Description |
|---|---|---|---|
| `/face_position` | `geometry_msgs/Point` | face_tracking_node | Target face center in normalized coordinates |
| `/behavior_state` | `std_msgs/String` | interaction_controller_node | Override mode (e.g., "idle", "intro", "sleep") |

### Published Topics

| Topic | Type | Description |
|---|---|---|
| `/servo_commands` | `std_msgs/Float32MultiArray` | Smoothed target angles [ch0, ch1, ..., ch8] |
| `/serial_out` | `std_msgs/String` | ASCII servo frame for Arduino |

### Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `smoothing_alpha` | float | 0.15 | Exponential smoothing factor (0.15 = heavy smoothing, slower response) |
| `max_velocity` | float | 120.0 | Maximum servo velocity in degrees/second |
| `neck_pan_range` | tuple | (-45, 45) | Neck yaw (Ch 2) limits in degrees |
| `neck_tilt_range` | tuple | (-30, 30) | Neck pitch (Ch 0+1 differential) limits |
| `neck_roll_range` | tuple | (-15, 15) | Neck roll (Ch 0+1 differential) limits |
| `eye_range` | tuple | (-20, 20) | Eye left/right (Ch 5,6) limits |
| `eyelid_range` | tuple | (0, 180) | Eyelid open/close (Ch 3,4) limits |
| `jaw_range` | tuple | (0, 60) | Jaw open (Ch 7) limits |
| `serial_port` | string | "/dev/ttyACM0" | Arduino Mega USB serial device |
| `serial_baud` | int | 115200 | Serial baud rate |

### Servo Channel Mapping

| Channel | Hardware | Function | Notes |
|---|---|---|---|
| 0 | MG996R | Neck differential (pitch + roll) | Coupled with Ch 1 for pure pitch |
| 1 | MG996R | Neck differential (pitch - roll) | Coupled with Ch 0 for pure roll |
| 2 | MG996R | Neck yaw (pan) | Center = 90° |
| 3 | MG90S | Eyes Pan | Center = 90° |
| 4 | MG90S | Eyes Tilt | Center = 90° |
| 5 | MG90S | Upper Eyelid | 0 = closed, 180 = open |
| 6 | MG90S | Lower Eyelid | 0 = closed, 180 = open |
| 7 | MG90S | Jaw | 0 = closed, ~60° = open for speech |
| 8 | MG90S | Card ejector | Software-triggered, 38s delay after activation |

### Differential Neck Kinematics

The 3-DOF neck uses a differential mechanism where two MG996R servos (Ch 0, Ch 1) are mechanically coupled to produce independent pitch and roll.
This allows the head to nod (pitch) and tilt sideways (roll) with only two actuators. The third MG996R (Ch 2) provides yaw (left/right pan). This design reduces actuator count and wiring complexity through the neck joint.

### Key Design Decisions

- **Exponential smoothing (alpha=0.15)**: Chosen over moving average for lower memory footprint and smoother perceived motion. The low alpha means the robot appears to ease into movements rather than snapping to targets, which looks more natural in social interaction. This value was tuned empirically during bench testing.
- **Velocity cap (120°/s)**: Prevents mechanical shock and reduces current spikes. MG996R servos can move faster, but this risks linkage stress in the 3D-printed differential mechanism. The cap was validated by stress-testing the neck linkage.
- **Serial ASCII protocol**: Human-readable for debugging. Trade-off is slightly higher bandwidth than binary (~20 bytes/frame vs ~10 bytes), but at 20 Hz this is negligible on USB serial. More importantly, a binary protocol tested early in development failed due to desynchronization on dropped bytes.

---

## interaction_controller_node

### Purpose
Manages high-level behavior states, coordinates audio playback with jaw motion, handles the card ejector sequence, and interfaces with the WebSocket-based UI.

### Subscribed Topics

| Topic | Type | Publisher | Description |
|---|---|---|---|
| `/locked_face` | `std_msgs/Bool` | face_tracking_node | Presence of engaged visitor |
| `/face_position` | `geometry_msgs/Point` | face_tracking_node | Visitor position for greeting orientation |

### Published Topics

| Topic | Type | Description |
|---|---|---|
| `/behavior_state` | `std_msgs/String` | Current behavior mode (see state list) |
| `/audio_trigger` | `std_msgs/String` | Path to audio file for playback |

### Behavior States

| State | Trigger | Duration | Actions |
|---|---|---|---|
| `IDLE` | No face locked | Indefinite | Slow random scanning, periodic blinks (randomized interval between 2.5–6.5 seconds, 0.3-second duration) |
| `GREETING` | Face newly locked | 3–5s | Arabic intro audio, synchronized jaw motion, direct eye contact |
| `ENGAGED` | Face maintained | Variable | Follow face, micro-saccades, occasional blinks |
| `TALKING` | Audio playback active | Audio length | Jaw synced to audio amplitude, eye contact maintained |
| `CARD_EJECT` | Software trigger | 38s delay | Ch 7 activated, brief motion, return to idle |

### Key Design Decisions

- **State machine over behavior trees**: Simpler to debug and validate for a single-exhibition deployment. A behavior tree would be warranted for more complex multi-modal interaction in future versions.
- **Randomized idle behavior**: Prevents repetitive mechanical patterns that look robotic. The random direction flips, pauses, and micro-saccades give the impression of curiosity.
- **Audio via subprocess**: mpg123 was chosen for low CPU overhead on Pi 4. Trade-off is less precise synchronization than a dedicated audio library, but sufficient for speech-like jaw motion. A future improvement would integrate a proper audio pipeline.
- **Card ejector 38s delay**: prevent rapid repeated triggering.

## Performance Observations

| Resource | Observation | Notes |
|---|---|---|
| Pi 4 CPU | High load during peak face-detection periods | Multiple simultaneous faces increase processing; compute upgrade under consideration |
| Pi 4 RAM | ROS2 + OpenCV + Chromium runs within 4GB | No memory pressure observed during exhibition |
| USB Bandwidth | Serial + camera within USB 2.0 capacity | No bandwidth issues observed |
| Serial Latency | Low and consistent | ASCII protocol at 115200 baud proved reliable |
| Servo Update Rate | 20 Hz as configured | No missed updates observed |
| Face Detection Rate | Typically near the configured 30 Hz | Drops slightly under high CPU load |
