# Lessons Learned

## 1. Multi-Axis Joints Require Tight Tolerances

The 3-DOF neck (pan + dual tilt) was the most mechanically complex part of ALEX. The initial 3D-printed tilt linkage had noticeable backlash, which manifested as head wobble during fast panning. Tightening the tolerances and switching from PLA to PETG reduced this significantly. For v1.1, machined aluminum or a preloaded design should be evaluated.

**Engineering principle**: Multi-axis joints amplify backlash. In a dual-tilt arrangement, backlash in either tilt servo appears as coupled error in the combined output. Tolerance stack-up must be analyzed as a system, not per-part.

---

## 2. Separate Real-Time and Non-Real-Time Compute

The decision to put servo control on the Arduino Mega while running ROS2 on the Raspberry Pi 4 was correct and validated under exhibition conditions. The Pi's non-deterministic Linux scheduler occasionally caused jitter in user-space processes. If servo PWM had been generated directly from the Pi, this would have caused visible servo twitching.

The Arduino's dedicated control loop (running at a fixed rate) guarantees consistent PWM timing regardless of Pi CPU load. The simple ASCII serial protocol is robust enough that occasional USB latency spikes do not cause visible motion artifacts due to the head_controller_node's smoothing buffer.

**Engineering principle**: Never run real-time motor control on a general-purpose OS without a real-time kernel or co-processor. The small serial latency is a worthwhile trade-off for deterministic actuator behavior.

---

## 3. Simple Protocols Beat Complex Protocols

The `ch:angle,ch:angle,...` ASCII protocol was initially considered a temporary solution. However, it proved to be the most reliable part of the system. It is human-readable (critical for debugging with `screen /dev/ttyACM0 115200`), requires no parsing library, and gracefully handles dropped characters because the newline frame delimiter allows easy resynchronization.

An early attempt at a binary protocol (fixed-length frame) failed because a single dropped byte desynchronized the entire stream, requiring an Arduino reboot. The ASCII protocol's verbosity is negligible at the configured 20 Hz update rate and 115200 baud.

**Engineering principle**: For low-bandwidth, low-rate control links, human-readable protocols are often more reliable than binary protocols due to self-synchronizing frame boundaries and debuggability.

---

## 4. Exhibition Environments Are Harder Than Lab Environments

MACTECH revealed issues that did not appear during lab testing:

| Lab Behavior | Exhibition Behavior | Root Cause | Fix |
|---|---|---|---|
| Face detection reliable | False positives on booth banners | Haar cascade triggered by banner text/graphics | Increase `detection_min_size` |
| Quiet audio | Audio drowned by crowd noise | 3.5mm jack output insufficient | Add small powered speaker or amplifier |
| Stable face lock | Lock switching between visitors | Two visitors at similar distance/confidence | Increase score hysteresis threshold |
| Cool servos | MG996R neck servos warm to touch | Continuous operation for 8+ hours vs. short lab tests | Add thermal monitoring; software duty-cycle limiter |

**Engineering principle**: Always budget significantly more testing time for exhibition conditions than for lab validation. Environmental variables (lighting, noise, crowd density, power quality) dominate system behavior.

---

## 5. AI-Assisted Development Has Limits

AI tools were used extensively in ALEX's development for:
- Blender shape refinement and proportion feedback
- ROS2 node scaffolding and debugging
- Wiring diagram review and hardware selection
- Documentation and code comments

**What AI did well**:
- Accelerated ROS2 node boilerplate and callback structure
- Suggested the exponential smoothing algorithm based on common robotics patterns
- Helped debug serial timing issues by suggesting buffer size adjustments
- Generated the differential kinematics equations correctly

**What AI did not do**:
- Could not predict the MG996R thermal issue (requires physical testing)
- Suggested a binary serial protocol that failed in practice (theory vs. reality)
- Blender proportion advice was generic; final proportions required human aesthetic judgment
- Could not tune the Haar cascade parameters for the specific exhibition lighting

**Engineering principle**: AI is a powerful accelerator for known patterns and debugging, but physical validation, environmental testing, and aesthetic judgment remain human responsibilities. Document AI assistance honestly -- it demonstrates modern engineering tool fluency.

---

## 6. Velocity Limiting Is Safety-Critical

During early testing without velocity caps, a rapid face-position jump caused the neck servos to slam to a new angle. The resulting shock load cracked a 3D-printed servo bracket. The velocity cap was added immediately and prevented all subsequent mechanical failures.

**Engineering principle**: Always rate-limit actuator commands, even if the actuator itself is capable of higher speeds. Mechanical systems have impulse limits that are lower than their steady-state torque limits.

---

## 7. Power Supply Design Is System Design

The initial single-rail power design (5V for everything) failed when all 9 servos moved simultaneously. The voltage sag caused the Raspberry Pi to brown-out and reboot. Separating the Pi onto a dedicated 5V/3A supply and the servos onto a 6V/5A+ supply completely eliminated the issue.

**Engineering principle**: Actuator power and logic power must be isolated in all but the smallest systems. The peak current draw of multiple servos under load is always higher than the sum of their idle currents.

---

## 8. Randomization Makes Robots Feel Alive

The deterministic idle behavior (scan left, pause, scan right, pause) tested in the lab looked mechanical and predictable. Adding randomized scan directions, durations, pause lengths, and micro-saccades transformed the robot's presence from "machine" to "entity." Visitors at MACTECH consistently described ALEX as "curious" or "attentive" -- adjectives that map directly to the randomization parameters.

**Engineering principle**: Perceived intelligence in social robots is often a function of behavioral variance, not complexity. A simple state machine with rich randomization outperforms a complex deterministic script in human-robot interaction.

---

## 9. The Blink Timing Matters

Human blink intervals follow a roughly log-normal distribution with a mean of approximately 4 seconds. The initial fixed 3-second blink interval looked artificial. Switching to a uniform random distribution between approximately 2.5–6.5 seconds (updated each blink) produced a more natural rhythm. The brief blink duration with rapid eyelid motion matches human ballistic blink kinematics.

**Engineering principle**: Biomechanical fidelity in social robots requires research into human motor patterns, not just functional correctness. The difference between "works" and "feels right" is often in the timing distributions.

---

## 10. WebSocket UI Decouples Development

Using rosbridge WebSocket for the display UI allowed the frontend (HTML/JavaScript) to be developed independently from the ROS2 backend. The Chromium kiosk simply connects to `ws://localhost:9090` and subscribes to topics. This meant UI changes could be tested without rebuilding the ROS2 workspace, and the UI could be redesigned by a web developer without ROS knowledge.

**Engineering principle**: Decouple subsystems via standard protocols. The WebSocket abstraction layer allowed parallel development and reduced integration risk.

---

## 11. Document As You Build

Attempting to write documentation after the MACTECH exhibition required significant reverse-engineering of the code and hardware. The wiring diagram was reconstructed from the actual build rather than from a pre-existing plan. For future projects, maintaining a living `docs/` directory with:
- Daily build notes
- Photos of each assembly stage
- Test results and parameter changes
- Decisions and rationale

...would reduce documentation effort substantially and improve accuracy.

**Engineering principle**: Documentation is a byproduct of the build process, not a post-build task. The best technical documentation is written while the engineer's mental model is fresh.

---

## 12. Plan for Obsolescence

The Raspberry Pi 4, while adequate for ALEX v1, is already the limiting factor. Face detection at 640×480 consumes most of the available CPU, leaving little headroom for future features (e.g., emotion recognition, speech recognition, larger vision models). The mechanical design accommodates a larger compute module (Jetson Orin Nano has a similar mounting footprint), but the software stack will require migration effort.

**Engineering principle**: Design mechanical and electrical interfaces for the next generation of compute, not the current one. The Pi 4 was chosen for availability and cost; the next iteration should target multi-year capability headroom.
