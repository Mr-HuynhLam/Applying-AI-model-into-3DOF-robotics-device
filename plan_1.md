# VLA-Driven Compliant Robot: Project Plan

> Status: Initial draft (v0.1)
> Last updated: 2026-10-07

## Assumptions for time estimates

- One engineer working full time (~40 h/week)
- Hardware (robotic device, EtherCAT motor driver, RGB-D camera, host PC) is already available
- Moderate prior experience with ROS2 and Linux; limited prior experience with EtherCAT real-time and VLA models
- Estimates are ranges (optimistic to realistic); hardware debugging is the biggest source of slippage
- Each stage ends with a short buffer for documentation and cleanup

## Summary

| Stage | Name | Estimate | Depends on |
|-------|------|----------|------------|
| 0 | Communication setup | 1-2 weeks | None |
| 1 | Position control | 1-2 weeks | 0 |
| 2 | Kinematics and state estimation (ROS2) | 2-3 weeks | 1 |
| 3 | Vision setup | 1-2 weeks | 0 (can run parallel to 1-2) |
| 4 | VLA integration | 3-5 weeks | 2, 3 |
| 5 | Dynamic compliance | 3-4 weeks | 4 |
| 6 | System validation | 2-3 weeks | 5 |
| 7 | Optional: digital twin and simulator | 2-4 weeks | 2 (can start earlier) |
| | **Core total (0-6)** | **13-21 weeks** | |
| | **With optional (0-7)** | **15-25 weeks** | |

Stage 3 is independent of stages 1-2, so with a second person (or by interleaving) the core timeline could shrink by 1-2 weeks.

---

## Stage 0: Communication setup (1-2 weeks)

- [ ] Set up host PC
  - Install Ubuntu LTS with a PREEMPT_RT kernel
  - Isolate CPU cores, set CPU governor to performance, disable power saving
  - Dedicated NIC for EtherCAT
- [ ] Configure real-time EtherCAT communication
  - Choose a master (IgH EtherCAT Master, SOEM, or ros2_control EtherCAT driver)
  - Scan the bus, confirm slaves are detected and reach OP state
  - Measure cycle time jitter (e.g. cyclictest, target < 50 us at 1 kHz)

**Done when:** the driver stays in OPERATIONAL state at the target cycle rate with stable jitter for 1+ hour.

**Risks:** RT kernel and NIC driver compatibility; ESI/PDO mapping mismatches with the motor driver.

---

## Stage 1: Position control (1-2 weeks)

- [ ] Control the robotic device using commands
- [ ] Receive motor position feedback
- [ ] Test position, velocity, and torque tracking
  - Step and sine reference tests
  - Log tracking error, latency, and saturation behavior

**Done when:** all three control modes track references within agreed error bounds, and feedback is read every cycle.

**Risks:** Gain tuning, encoder offsets and homing, safety limits during first motion.

---

## Stage 2: Kinematics and state estimation, ROS2 (2-3 weeks)

- [ ] Connect host PC to ROS2 to motor driver via EtherCAT (ros2_control hardware interface)
- [ ] Control robotic device via ROS2 (controllers, joint command and state topics)
- [ ] Calculate state estimation, forward kinematics, and inverse kinematics
  - Write the URDF and verify against the real device
  - Choose a solver (KDL, Pinocchio, or analytic IK)
  - Joint velocity estimation and filtering

**Done when:** a Cartesian target gives the correct joint motion on hardware, and FK/IK round-trip error is negligible.

**Risks:** URDF inaccuracies (inertia, offsets); IK singularities and joint limits; real-time safety of the hardware interface.

---

## Stage 3: Vision setup (1-2 weeks)

- [ ] Set up camera (RGB-D)
- [ ] Camera calibration
  - Intrinsics and depth alignment
  - Hand-eye (extrinsic) calibration relative to the robot base
- [ ] Connect camera to ROS2 (driver node, TF frames, image and point cloud topics)

**Done when:** a known object's position in the camera frame converts to the correct robot-frame position.

**Risks:** Calibration accuracy; USB bandwidth; frame rate and timestamp sync with the robot.

---

## Stage 4: VLA integration (3-5 weeks)

- [ ] Deploy a VLA model into ROS2
  - Pick a model and check GPU/VRAM requirements and inference latency
  - Wrap inference as a ROS2 node (observation in, action out)
- [ ] Integrate the whole system in ROS2 (robotic device + camera + VLA + controller)
  - Define action space (joint deltas, end-effector poses, or targets)
  - Rate matching: VLA is slow (a few Hz) versus controller (kHz); add interpolation or action chunking
  - Launch files, parameter configs, logging

**Done when:** a language instruction produces robot motion end to end, with measured latency recorded.

**Risks:** Model fit to your robot (embodiment gap, may need fine-tuning); inference latency; action safety filtering.

---

## Stage 5: Dynamic compliance (3-4 weeks)

- [ ] Map VLA semantic output to dynamic driver
  - Impedance/admittance parameters (stiffness, damping) as VLA or task-level output
  - Implement the compliance controller (impedance or admittance)
- [ ] Pass interaction torque back into the VLA observation space
  - Torque estimation (current-based or force/torque sensor) and filtering
  - Add it to the observation vector and time-align with images

**Done when:** the arm yields to external force in a controlled way, and torque data appears in the VLA observations.

**Risks:** Stability under contact; torque estimate quality without a dedicated sensor; the VLA may need retraining to use torque input.

---

## Stage 6: System validation (2-3 weeks)

- [ ] Add virtual boundary limits to the controller (workspace, velocity, torque, joint limits)
- [ ] Human collision test to verify compliance
  - Define the test protocol first (speeds, contact points, force thresholds)
  - Review against ISO/TS 15066 or equivalent guidance
  - Record force, torque, and stop-time data

**Done when:** limits are enforced under fault injection, and collision tests pass the agreed thresholds with documented results.

**Risks:** Safety scope; need for an emergency stop and safe-torque-off hardware; safe test setup with a dummy or force gauge before any human contact.

---

## Stage 7: Optional (2-4 weeks)

- [ ] Build a digital twin for the whole system
- [ ] Simulator using ROS2 (Gazebo, Isaac Sim, or MuJoCo with a ROS2 bridge)

**Benefits:** Safer testing of the VLA and compliance behavior; data collection; regression tests. Consider starting a basic simulator right after Stage 2 so Stages 4-5 can be tested virtually first.

---

## Cross-cutting items (not yet in plan)

- [ ] Safety: emergency stop, watchdog, safe start-up and shutdown
- [ ] Version control, CI, and a reproducible environment (Docker or setup scripts)
- [ ] Data logging (rosbag2) for debugging and VLA fine-tuning
- [ ] Documentation and demo video

## Open questions

- Which robotic device and motor drivers (affects EtherCAT master choice and torque sensing)?
- Which VLA model, and will it be fine-tuned?
- GPU available on the host PC, or a separate inference machine?
- Is there a hard deadline or demo date?

## Change log

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2026-10-07 | Initial plan with time estimates |
