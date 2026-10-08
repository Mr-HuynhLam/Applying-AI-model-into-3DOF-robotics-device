# VLA-Driven Compliant Robot: Project Plan

> Status: Draft v0.2 (split architecture)
> Last updated: 2026-10-07

## Architecture (new in v0.2)

Because the office PC runs security software and WSL2 cannot provide hard real-time EtherCAT, the system is split across two machines:

| Machine | OS | Role |
|---------|----|------|
| **Control PC** (dedicated mini/industrial PC) | Ubuntu 24.04 + PREEMPT_RT, dedicated NIC | EtherCAT master, ros2_control, kinematics, state estimation, compliance controller, safety limits |
| **Workstation** (office PC, GPU) | Dual: Windows + Ubuntu 24.04 | VLA inference, camera processing, simulation / digital twin, development |

The two machines communicate over ROS2 (DDS) on a wired network link. The fast loop (kHz) stays entirely on the Control PC; only slow, high-level data (images, VLA actions, torque summaries) crosses the network. The robot must stay safe if the network or the VLA node drops out.

```
 Workstation                            Control PC 
 Camera -> VLA node  --- ROS2/DDS --->  Compliance + position controller
 Simulator / twin    <-- (wired LAN) -- State, torque feedback
                                         |  EtherCAT (dedicated NIC)
                                         v
                                       Motor drivers / robotic device
```

## Assumptions for time estimates

- One engineer working full time (~40 h/week)
- Hardware (robotic device, EtherCAT motor driver, RGB-D camera, Control PC, workstation with GPU) is available
- Moderate prior experience with ROS2 and Linux; limited prior experience with EtherCAT real-time and VLA models
- Estimates are ranges (optimistic to realistic); hardware debugging is the biggest source of slippage
- IT approves a separate Control PC; if not, see Open questions
- Each stage ends with a short buffer for documentation and cleanup

## Summary

| Stage | Name | Estimate | Depends on |
|-------|------|----------|------------|
| 0 | Communication setup | 1-2 weeks | None |
| 1 | Position control | 1-2 weeks | 0 |
| 2 | Kinematics and state estimation (ROS2) | 2-3 weeks | 1 |
| 3 | Vision setup | 1-2 weeks | 0 (can run parallel to 1-2) |
| 4 | VLA integration (incl. cross-machine networking) | 4-6 weeks | 2, 3 |
| 5 | Dynamic compliance | 3-4 weeks | 4 |
| 6 | System validation | 2-3 weeks | 5 |
| 7 | Optional: digital twin and simulator | 2-4 weeks | 2 (can start earlier) |
| | **Core total (0-6)** | **14-22 weeks** | |
| | **With optional (0-7)** | **16-26 weeks** | |

Change from v0.1: +1 week for two-machine setup (mostly Stage 4 networking and DDS tuning). Stage 3 is independent of stages 1-2, so interleaving or a second person could shrink the timeline by 1-2 weeks.

---

## Stage 0: Communication setup (1-2 weeks)

- [ ] Set up Control PC
  - Install Ubuntu LTS
  - Dedicated NIC for EtherCAT (no other traffic on it)
  - Second NIC (or port) for the link to the workstation
- [ ] Configure real-time EtherCAT communication
  - Choose a master (IgH EtherCAT Master, SOEM, or ros2_control EtherCAT driver)
  - Scan the bus, confirm slaves are detected and reach OP state
  - Measure cycle time jitter (e.g. cyclictest, target < 50 us at 1 kHz)
- [ ] Set up workstation
  - Install dual OS with Ubuntu, NVIDIA GPU driver + CUDA in dual OS
  - Install ROS2 (same distro as Control PC)
  - Confirm with IT which tools are allowed (usbipd-win, WSL networking mode)

**Done when:** the driver stays in OPERATIONAL state at the target cycle rate with stable jitter for 1+ hour, and the workstation has a working ROS2 + GPU environment.

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

- [ ] Connect Control PC to ROS2 to motor driver via EtherCAT (ros2_control hardware interface)
- [ ] Control robotic device via ROS2 (controllers, joint command and state topics)
- [ ] Calculate state estimation, forward kinematics, and inverse kinematics
  - Write the URDF and verify against the real device
  - Choose a solver (KDL, Pinocchio, or analytic IK)
  - Joint velocity estimation and filtering

**Done when:** a Cartesian target gives the correct joint motion on hardware, and FK/IK round-trip error is negligible.

**Risks:** URDF inaccuracies (inertia, offsets); IK singularities and joint limits; real-time safety of the hardware interface.

---

## Stage 3: Vision setup (1-2 weeks)

- [ ] Decide where the camera connects
  - Option A: Workstation via usbipd-win (images stay next to the VLA, less network load)
  - Option B: Control PC, streaming compressed images over the LAN (use if USB passthrough is blocked by IT)
- [ ] Set up camera (RGB-D)
- [ ] Camera calibration
  - Intrinsics and depth alignment
  - Hand-eye (extrinsic) calibration relative to the robot base
- [ ] Connect camera to ROS2 (driver node, TF frames, image and point cloud topics)

**Done when:** a known object's position in the camera frame converts to the correct robot-frame position.

**Risks:** Calibration accuracy; USB passthrough stability and bandwidth; frame rate and timestamp sync with the robot.

---

## Stage 4: VLA integration (4-6 weeks)

- [ ] Cross-machine ROS2 networking (new)
  - Wired link between workstation and Control PC.
  - Matching ROS_DOMAIN_ID, DDS choice (Cyclone DDS or Fast DDS) and discovery config; consider Zenoh if multicast is blocked
  - Time sync (chrony or PTP) so images, torque, and actions share timestamps
  - Measure round-trip latency and packet loss; define behavior on link loss
- [ ] Deploy a VLA model into ROS2
  - Pick a model and check GPU/VRAM requirements and inference latency
  - Wrap inference as a ROS2 node (observation in, action out)
- [ ] Integrate the whole system in ROS2 (robotic device + camera + VLA + controller)
  - Define action space (joint deltas, end-effector poses, or targets)
  - Rate matching: VLA is slow (a few Hz) versus controller (kHz); add interpolation or action chunking on the Control PC
  - Watchdog: robot holds or safe-stops if VLA actions stop arriving
  - Launch files, parameter configs, logging

**Done when:** a language instruction produces robot motion end to end across both machines, with measured latency recorded and link-loss behavior tested.

**Risks:** DDS discovery issues across WSL2/NAT or corporate firewall; model fit to your robot (embodiment gap, may need fine-tuning); inference latency; action safety filtering.

---

## Stage 5: Dynamic compliance (3-4 weeks)

- [ ] Map VLA semantic output to dynamic driver
  - Impedance/admittance parameters (stiffness, damping) as VLA or task-level output
  - Implement the compliance controller (impedance or admittance) on the Control PC, not across the network
- [ ] Pass interaction torque back into the VLA observation space
  - Torque estimation (current-based or force/torque sensor) and filtering
  - Downsample and send to the workstation; time-align with images

**Done when:** the arm yields to external force in a controlled way, even if the VLA node is stalled, and torque data appears in the VLA observations.

**Risks:** Stability under contact; torque estimate quality without a dedicated sensor; the VLA may need retraining to use torque input.

---

## Stage 6: System validation (2-3 weeks)

- [ ] Add virtual boundary limits to the controller (workspace, velocity, torque, joint limits), enforced on the Control PC
- [ ] Human collision test to verify compliance
  - Define the test protocol first (speeds, contact points, force thresholds)
  - Review against ISO/TS 15066 or equivalent guidance
  - Record force, torque, and stop-time data
- [ ] Network fault tests: unplug the link, kill the VLA node, add latency; confirm safe behavior

**Done when:** limits are enforced under fault injection, and collision tests pass the agreed thresholds with documented results.

**Risks:** Safety scope; need for an emergency stop and safe-torque-off hardware; safe test setup with a dummy or force gauge before any human contact.

---

## Stage 7: Optional (2-4 weeks)

- [ ] Build a digital twin for the whole system
- [ ] Simulator using ROS2 (Gazebo, Isaac Sim, or MuJoCo with a ROS2 bridge), running on the workstation

**Benefits:** Safer testing of the VLA and compliance behavior; data collection; regression tests. Consider starting a basic simulator right after Stage 2 so Stages 4-5 can be tested virtually first. The workstation side is a good fit for this.

---

## Cross-cutting items (not yet scheduled)

- [ ] Safety: emergency stop, watchdog, safe start-up and shutdown
- [ ] Version control, CI, and a reproducible environment (Docker or setup scripts for both machines)
- [ ] Data logging (rosbag2) for debugging and VLA fine-tuning
- [ ] Documentation and demo video

## Open questions

- Will IT approve a separate Control PC on an isolated network? If not, fallbacks: dual-boot/external SSD with Ubuntu RT, or a dedicated lab PC exception.
- Which robotic device and motor drivers (affects EtherCAT master choice and torque sensing)?
- Which VLA model, and will it be fine-tuned?
- GPU specs on the workstation (VRAM for the chosen VLA)?
- Is there a hard deadline or demo date?

## Change log

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2026-10-07 | Initial plan with time estimates |
| 0.2 | 2026-10-07 | Split architecture (RT Control PC + WSL2 workstation); added networking/DDS tasks, camera placement options, network fault tests; estimates +1 week (core 14-22 weeks) |
| 0.3 | 2026-10-08 | Try to install Dual-boot on workstation PC instead of WSL2 for a stable simulation & training progress |
