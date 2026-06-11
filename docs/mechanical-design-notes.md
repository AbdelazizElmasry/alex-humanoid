# Mechanical Design Notes

## Design Philosophy

ALEX's mechanical design prioritizes **modularity**, **rapid prototyping**, and **exhibition reliability**. The head is fully 3D printed (FDM) to allow fast iteration, with the internal structural frame designed in SolidWorks and the external aesthetic shell designed in Blender for organic surfacing.

---

## CAD Workflow

| Component | Tool | Method | Material | Purpose |
|---|---|---|---|---|
| Internal structural frame | SolidWorks | Parametric feature-based | PLA/PETG | Servo mounting, bearing supports, camera bracket, wiring channels |
| External head/face shell | Blender | Subdivision surface sculpting | PLA (painted) | Aesthetic outer skin, proportions, public-facing appearance |
| Neck differential linkage | SolidWorks | Kinematic sketch + extrude | PLA/PETG | Converts 2 servo inputs to 3-DOF output |

**Why two CAD tools?**
- SolidWorks for mechanical precision: mates, dimensions, interference checks, BOM generation.
- Blender for industrial design: freeform surfaces, proportional refinement, visual appeal. The AI-assisted development was used heavily in Blender to refine proportions and surface flow.

---

## Neck Mechanism: 3-DOF Differential Drive

### Overview
The neck provides pan (yaw) and dual tilt (pitch and roll) using 3× MG996R servos. One servo handles pan; the remaining two servos work together as a tilt pair for pitch and roll control.

### Differential Kinematics

```
        ┌─────────────┐
        │   Head      │
        │   Mass      │
        └──────┬──────┘
               │
    ┌──────────┼─────────┐
    │          │         │
    ▼          ▼         ▼
 ┌──────┐   ┌──────┐  ┌─────┐
 │Ch 0  │   │Ch 1  │  │Ch 2 │
 │MG996 │   │MG996 │  │MG996│
 │Tilt A│   |Tilt A│  | Yaw │
 └──────┘   └──────┘  └─────┘
    │           │        │
    └──────┬────┘        │
           │             │
      Tilt Pair         Pan
    (Pitch + Roll)    Bearing
```

**Servo 0 (Ch 0) and Servo 1 (Ch 1) -- Tilt Pair:**
- Both servos mounted on the pan platform (above the pan bearing)
- Work together to produce combined pitch and roll motion
- Pushrods connect to a shared tilt platform or gimbal ring
- Software coordinates the two servos to produce the desired tilt orientation
- Range: approximately ±30° pitch, ±15° roll (combined, depending on mechanical geometry)

**Servo 2 (Ch 2) -- Pan:**
- Mounted coaxially with the neck pivot
- Direct drive or belt/pulley to the pan bearing
- Range: approximately ±45° from center
- Independent yaw control

### Why a Dual-Tilt Pair?

| Advantage | Explanation |
|---|---|
| Independent pan | Pan axis is mechanically decoupled from tilt, simplifying control logic |
| Shared tilt loading | Two servos share the head weight during tilt motions, reducing individual load |
| Simpler kinematics | Pan-tilt coordination is more intuitive than differential decoding for software control |
| Compact packaging | Fits within a human-proportioned neck cylinder |

### Trade-offs

| Limitation | Mitigation |
|---|---|
| Synchronized tilt motion | Tilt A and Tilt B must move in coordinated pairs for pure pitch or pure roll. Software handles the coordination. |
| Backlash | 3D-printed tilt linkages and pin joints have inherent backlash. Tolerances were tightened. |
| Strength | PLA tilt parts under cyclic loading. PETG or ABS recommended for v1.1. |
| Range | Tilt pair geometry limits individual axis range. Current: approximately ±30° pitch, ±15° roll. |

---

## Face Actuator Layout (MG90S)

### Servo Placement

| Channel | Location | Mounting | Motion | Mechanical Coupling |
|---|---|---|---|---|
| 3 | Eyes pan | Gimbal frame inside head | Pan (left/right) | Direct horn or pushrod |
| 4 | Eyes tilt | Gimbal frame inside head | Tilt (up/down) | Direct horn or pushrod |
| 5 | Upper eyelid | SolidWorks-designed bracket, pivot at upper lid | Open/close (0-180°) | Direct horn linkage |
| 6 | Lower eyelid | SolidWorks-designed bracket, pivot at lower lid | Open/close (0-180°) | Direct horn linkage |
| 7 | Mouth (jaw) | Hinge at rear of skull | Open/close 0-60° | Lever arm to servo horn |
| 8 | Card ejector | Slide mechanism | Eject | Belt drive |

### Eyelid Linkage

The upper and lower eyelids each use a simple 4-bar linkage:
- Servo horn (input crank)
- Coupler link (3D printed, approximately 25 mm)
- Eyelid lever (output rocker, pivot at lid attachment point)
- Frame ground (structural)

**Design note**: Rapid motion (no gradual sweep) was chosen for blinking to match human eyelid dynamics. The servo moves at maximum speed for approximately 0.3 seconds, creating a sharp, natural-looking blink rather than a slow mechanical close. Upper and lower eyelids can be driven independently or together for different expression effects.

### Eye Gimbal

The eyes are mounted in a 2-DOF gimbal (pan and tilt) driven by two MG90S servos (Ch 3 and Ch 4). The gimbal frame is supported by shoulder screws forming two perpendicular pivot axes. Servo horns connect via short pushrods to the gimbal frame.

**Why 2-DOF eyes?**
- Pan and tilt eye motion allows the robot to track faces more naturally, making eye contact even when the visitor is not at the same height.
- Adds expressiveness to the face during interaction.
- Neck pan and tilt compensate for coarse positioning; eye pan and tilt provide fine gaze adjustment.

### Jaw Mechanism

- Hinge axis: horizontal
- Servo 7 drives a lever arm that pushes the jaw down
- Jaw closes passively (spring/gravity); opens actively (servo)
- This reduces servo load during speech -- the servo only works to open, not to hold closed

---

## Material Selection

| Part | Material | Rationale |
|---|---|---|
| Structural frame | PETG | Higher temperature resistance than PLA; better layer adhesion for load-bearing parts |
| Head shell (outer) | PLA (painted) | Easier to sand and finish; non-structural; lower cost |
| Neck linkage | PETG or ABS | Cyclic loading; needs fatigue resistance |
| Servo horns | Plastic (included with MG996R/MG90S) | Standard; do not use 3D-printed horns for reliability |
| Fasteners | M3 stainless steel | Standard robotics size; corrosion resistance |
| Bearings | 608ZZ or 623ZZ (where applicable) | Neck bearing |

## Wiring & Cable Management

### Neck Pass-through

The neck is the highest-stress area for wiring. All servo cables, and camera CSI ribbon must pass through or alongside the neck joint.

**Strategy**:
- Cables routed externally along flexible cable chain or spiral wrap
- Strain relief at both ends of neck (torso frame and head frame)
- Minimum bend radius enforced by printed cable guides

### Connector Strategy

| Connection | Type | Location | Rationale |
|---|---|---|---|
| Pi → Arduino | USB-B cable | Internal, fixed | Standard, reliable, no custom crimping |
| Arduino → PCA9685 | JST-XH 4-pin | Internal, fixed | I2C + power; locking connector |
| PCA9685 → Servos | 3-pin servo headers | Distributed | Standard servo connectors; allow swap |
| Pi → Camera | CSI ribbon | Internal, fixed | Flat flex; low profile |
| Pi → Display | HDMI micro-to-standard | Internal, fixed | Off-the-shelf cable |
| Power → Pi | USB-C 5V/3A | Internal | Standard Pi power |
| Power → Servo rail | XT60 or barrel jack | Internal | High-current; polarized |

---

## Fabrication Notes

### 3D Printing Parameters

| Parameter | Structural (PETG) | Shell (PLA) |
|---|---|---|
| Layer height | 0.2 mm | 0.2 mm |
| Infill | 40% | 15% |
| Wall count | 6 | 4 |
| Print orientation | Stress-aligned | Visual surface optimized |
| Supports | Tree supports, touch build plate only | Minimal (designed for support-free where possible) |
| Post-processing | Ream bearing bores, tap M3 holes | Sand, prime, paint |

### Assembly Order

1. Print and post-process all structural parts
2. Press-fit bearings into neck gimbal
3. Install MG996R neck servos into torso frame
4. Assemble differential linkage (check for binding)
5. Mount head frame to neck output
6. Install MG90S face servos into head frame
7. Connect all linkages (eyelids, eyes, jaw)
8. Route wiring through neck and cable management
9. Install Pi 4, Arduino, PCA9685, camera, display
10. Power-on test: home all servos, check range of motion
11. Software calibration: center all channels, verify kinematics

---

## Design Iterations & Prototype Evolution

### v1.0 -- Exhibition Build 
- Final PETG structural frame
- PLA head shell with paint finish
- All 9 servos integrated and calibrated
- Deployed at MACTECH 2025

### v1.1 -- Planned Improvements
- Evaluate machined aluminum or carbon-fiber nylon (PA12-CF) for the tilt linkage
- Add quick-release neck coupling for transport

---

## Future Mechanical Improvements

Based on MACTECH exhibition observations and post-deployment analysis:

1. **Silicone Skin**: R&D needed for a molded silicone face skin that covers the mechanical linkages and provides more natural facial expressions. The current 3D-printed shell is rigid and limits expression subtlety.

2. **Actuator Upgrade**:
   - Higher-grade digital servos with better thermal characteristics
   - Add cooling ducts or heat sinks to servo cases
   - Software duty-cycle limiting (periodic rest intervals)

3. **Material Upgrade**: Move structural frame from PETG to carbon-fiber-reinforced nylon (e.g., PA12-CF) for higher stiffness-to-weight ratio and lower backlash in the differential linkage.

4. **Quick-Release Neck**: Design a magnetic or mechanical quick-release for the head to allow rapid transport and maintenance without rewiring.

