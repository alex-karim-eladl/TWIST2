# Neck Module

This page describes the add-on 2-DoF neck used with TWIST2. It covers the hardware, wiring, controller setup, and how the neck integrates with the existing Redis-based pipeline for both teleop and scripted motions.

---

## Overview

- Two Dynamixel actuators provide **yaw** and **pitch** (ID 0 = yaw, ID 1 = pitch).
- The onboard neck controller reads Redis **action_neck** commands and publishes **state_neck** feedback.
- The neck is optional: if no neck controller is running, other components continue to work and should treat missing keys as zeros.

---

## Hardware at a glance

- **Actuators:** 2 × Dynamixel (e.g., XM/XL series) set to 2 Mbps, IDs 0 (yaw) and 1 (pitch).
- **BOM (typical):**
  - 2 × Dynamixel servos + horn/mounts
  - 1 × U2D2 or compatible USB2DXL interface
  - 12 V (or actuator-rated) power supply
  - Aluminum/carbon brackets for pan-tilt
  - Fasteners and wiring (signal + power)
- **Assembly notes:**
  - Mount yaw at the base, pitch on top.
  - Keep wiring strain-relieved; avoid hard end-stop hits.
  - Align the “zero” pose: yaw forward, pitch level.

---

## Actuator setup (Dynamixel Wizard)

1. Install Dynamixel Wizard 2.0 from the Robotis site.
2. Plug the actuator chain via U2D2/USB2DXL.
3. Give serial access to your user:
   ```bash
   sudo chmod 777 /dev/ttyUSB0
   ```
4. In Wizard:
   - Set **baud** to **2 Mbps**.
   - Set **IDs**: yaw = **0**, pitch = **1**.
   - Verify motion and note the mechanical zero angles.

---

## Onboard controller setup

Run the neck control program on the robot-side PC (see your onboard repo). The controller should:

- Read Redis key `action_neck_unitree_g1_with_hands` (JSON, typically `[yaw, pitch]` in radians or actuator-native units as configured).
- Publish Redis key `state_neck_unitree_g1_with_hands` (same ordering).
- Use a safe rate limiter and joint limits to avoid over-rotation.

If you need a basic test loop, ensure Redis is running and send commands manually (example below).

---

## Software integration points

- **Teleop source:** `deploy_real/xrobot_teleop_to_robot_w_hand.py`
  - Computes neck targets (e.g., head retargeting) and writes `action_neck_unitree_g1_with_hands`.
- **Motion playback:** `deploy_real/server_motion_lib.py`
  - Sends zeros by default; can be extended to stream scripted neck poses.
- **Low-level controllers:**
  - `deploy_real/server_low_level_g1_real.py` and `deploy_real/server_low_level_g1_sim.py` read `state_neck_*` for logging; they do not drive the neck directly.
- **Recorder:** `deploy_real/server_data_record.py` logs both `action_neck_*` and `state_neck_*` if present.

Key names are consistent with the rest of the pipeline; swap `unitree_g1_with_hands` for your robot alias if you use a custom naming scheme.

---

## Quick test (manual Redis write)

With Redis running and the neck controller listening:

```bash
redis-cli set action_neck_unitree_g1_with_hands "[0.0, 0.2]"
```

You should see the neck pitch up slightly. Use small angles first; keep within mechanical limits.

---

## Safety notes

- Keep an E-stop reachable; avoid commanding large instantaneous jumps.
- Configure software limits in the neck controller to respect the mechanical range.
- Start from a neutral pose (yaw = 0, pitch = 0) after power-on; recalibrate if the horn slips.

---

## Troubleshooting

- **No motion:** Check `/dev/ttyUSB*` permissions and that IDs/baud match (2 Mbps). Confirm Redis key is being updated (`redis-cli get action_neck_unitree_g1_with_hands`).
- **Jitter/latency:** Add filtering in the neck controller; verify the publishing rate from teleop/motion server.
- **Wrong orientation:** Swap sign or axis mapping in the controller; confirm yaw is ID 0 and pitch is ID 1.
- **Over-travel or hitting stops:** Lower command limits, add soft limits in firmware, and verify zero alignment.
