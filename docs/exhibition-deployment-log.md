# Exhibition Deployment Log

## MACTECH 2025

| Attribute | Detail |
|---|---|
| **Event Name** | MACTECH |
| **Website** | https://mactech-eg.com/ |
| **Location** | Egypt |
| **Date** | End of December 2025 |
| **Frequency** | Annual (held end of December yearly) |
| **Robot Version** | ALEX Humanoid v1.0 |
| **Deployment Status** | Successful demonstration completed |

---

## Pre-Deployment Checklist

### Hardware Verification
- [x] Raspberry Pi 4 boot test (Ubuntu 22.04 + ROS2 Humble)
- [x] Arduino Mega 2560 serial communication test (115200 baud)
- [x] Adafruit PCA9685 I2C address scan (0x40 default)
- [x] All 9 servo channels range-of-motion test
  - Ch 0: Neck pan (1× MG996R) -- yaw range verified
  - Ch 1-2: Neck tilt A/B (2× MG996R) -- pitch and roll coordination verified
  - Ch 3-4: Eyes pan/tilt (2× MG90S) -- 2-DOF gaze tracking verified
  - Ch 5-6: Upper/lower eyelids (2× MG90S) -- blink and expression range verified
  - Ch 7: Mouth (1× MG90S) -- talking sync verified
  - Ch 8: Card ejector (1× MG90S) -- trigger and delay verified (38s)
- [x] Raspberry Pi Camera Module v2 (IMX219) -- face detection latency check
- [x] 7" HDMI display + Chromium kiosk -- WebSocket connection verified
- [x] Audio system (mpg123) -- Arabic intro playback verified
- [x] Power supply stability under full load (5V/6V rail, all servos active)

### Software Verification
- [x] ROS2 Humble workspace build (`colcon build`)
- [x] `face_tracking_node` -- Haar cascade model loaded, IOU tracking active
- [x] `head_controller_node` -- exponential smoothing (alpha≈0.15), velocity caps active
- [x] `interaction_controller_node` -- behavior state machine, audio triggers mapped
- [x] Serial protocol test: `ch:angle,ch:angle,...` format verified at 115200 baud
- [x] rosbridge WebSocket server -- client connection from Chromium kiosk verified
- [x] Auto-start systemd services configured

### Mechanical Verification
- [x] 3-DOF differential neck mechanism -- pitch+roll from 2× MG996R, yaw from 1× MG996R
- [x] Head shell (Blender-designed, 3D printed) -- fit and clearance check
- [x] Internal structural frame (SolidWorks-designed, 3D printed) -- stress check
- [x] Cable routing -- no pinch points, strain relief on neck joints
- [x] Emergency stop accessible

---

## Deployment Timeline

| Time | Activity |
|---|---|
| T-2 hours | Arrival, booth setup, power and network verification |
| T-1 hour | Robot boot sequence, ROS2 launch, camera calibration in venue lighting |
| T-30 min | Full motion test in exhibition space, crowd-distance tuning |
| T-0 | Doors open, demonstration begins |
| T+4 hours | Mid-day servo temperature check, power supply monitoring |
| T+8 hours | End of day shutdown, observation notes, issue log |

---

## Post-Exhibition Observations

### Issues Identified
1. **Long-running reliability**: Servos showed noticeable warming under continuous crowd interaction. This suggests the need for higher-end actuators or duty-cycle limiting for future deployments.
2. **Compute limitation**: Raspberry Pi 4 CPU load increased during peak face-detection periods with multiple simultaneous faces in frame. This indicates the compute platform is near its practical limit for this application.
3. **Face realism**: Visitor feedback was positive, but the rigid 3D-printed shell limits expression range. Silicone skin and additional expression channels would improve naturalness.

### Modifications Required (Post-Exhibition)
- [ ] Thermal management review for servos
- [ ] Compute platform upgrade evaluation
- [ ] Silicone skin R&D for facial expressions
- [ ] Additional high-end actuators for reliability
- [ ] Software duty-cycle limiting for long-running demos

---

## Emergency Procedures

| Scenario | Response |
|---|---|
| Servo runaway / overheating | Cut power to 5V servo rail, check PCA9685 OE pin |
| Serial communication loss | Restart Arduino Mega, verify USB cable integrity |
| Camera failure | Switch to idle scan behavior (no face tracking) |
| ROS2 node crash | `ros2 node list` diagnostic, restart affected node |
| Power supply fault | Emergency shutdown, isolate Pi and servo rails |
| Mechanical binding | Stop motion, manually home servos, inspect neck linkage |
