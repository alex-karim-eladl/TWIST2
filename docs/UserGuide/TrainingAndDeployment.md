# Training & Deployment Overview

This guide shows how to train a TWIST2 controller, export it to ONNX, and run it in simulation or on the real robot. It assumes you already followed the environment setup in [Installation](../GettingStarted/Installation.md) (both `twist2` and `gmr` conda envs) and have Redis running.

---

## Prerequisites

- **Envs:** `twist2` (Python 3.8) for training/deployment; `gmr` (Python 3.10+) for teleop retargeting.
- **Dependencies:** Isaac Gym, Mujoco, Legged Gym, RSL-RL, Redis client packages installed inside `twist2`.
- **Assets:** Checkpoint/output directory writable; optional dataset if you plan to train from scratch.
- **Redis:** `redis-server` running and reachable by all processes.
- **Hardware (real deployment):** Unitree robot reachable over the specified NIC; Dex hands if used; emergency stop available.

---

## Training a controller (high-level steps)

1. **Activate env**  
   ```bash
   conda activate twist2
   ```
2. **Prepare data (optional)**  
   - Use provided motions in `assets/example_motions` for quick tests.  
   - For custom training, place your datasets/motions where the config expects them.
3. **Configure training**  
   - Update configs in `legged_gym` / `pose` / `rsl_rl` as needed (reward, curriculum, model, logging).
4. **Launch training**  
   ```bash
   bash train.sh
   ```
   - Monitor logs/W&B if enabled.  
   - Training produces PyTorch checkpoints (e.g., `outputs/.../model_<step>.pt`).
5. **Evaluate in sim (optional)**  
   - Use your trained checkpoint with the sim loop (see “Sim2Sim verification” below).

---

## Export to ONNX

Once you have a trained PyTorch checkpoint, export to ONNX for real-time deployment:

```bash
conda activate twist2
bash to_onnx.sh   # adjust paths inside if needed
```

Output goes to `assets/ckpts/<name>.onnx`. Keep this file for both sim and real runs.

---

## Deployment recipes

Pick a motion source (teleop or scripted) and a controller target (sim or real). All processes share Redis.

### 1) Sim2Sim verification (MuJoCo)

- Terminal 1 (controller, `twist2`):
  ```bash
  conda activate twist2
  ./sim2sim.sh        # uses assets/ckpts/twist2_1017_20k.onnx by default
  ```
- Terminal 2 (motion source):
  - **Scripted motion:** `conda activate twist2 && ./run_motion_server.sh`
  - **Teleop:** `conda activate gmr && ./teleop.sh` (PICO + GMR)

### 2) Real robot (Sim2Real)

- Terminal 1 (low-level controller to robot, `twist2`):
  ```bash
  conda activate twist2
  ./sim2real.sh       # set NIC in the script; uses ONNX ckpt
  ```
- Terminal 2 (motion source):
  - **Teleop:** `conda activate gmr && ./teleop.sh`  
  - **Scripted motion:** `conda activate twist2 && ./run_motion_server.sh`

### 3) Data recording (optional)

Add a recorder alongside any setup:
```bash
conda activate twist2
python deploy_real/server_data_record.py --task_name <name> --data_folder <path>
```
It captures vision + `state_*` + `action_*` from Redis.

---

## Teleop vs. scripted motion

- **Teleop path:** PICO + XRoboToolkit + GMR (`teleop.sh`) publishes retargeted mimic obs and hand/neck actions to Redis. Use this for live control and data collection.
- **Scripted path:** Motion server (`run_motion_server.sh`) replays example motions as the high-level source; useful for quick validation without VR.
- Both paths feed the same Redis keys, so you can swap sources without changing the controller.

---

## Safety checklist (real robot)

- Verify NIC and IP settings in `sim2real.sh`; ensure stable wired link.
- Keep E-stop accessible; test with low gains if you modify PD settings.
- Start from default stance; avoid sending aggressive commands until confidence is built.
- Confirm Redis keys are updating before enabling torque (use `redis-cli monitor` sparingly).

---

## Troubleshooting

- **ONNX issues:** Re-export after updating deps; ensure `onnxruntime-gpu` is installed in `twist2`.
- **Isaac Gym/Mujoco import errors:** Check that `twist2` is active and `LD_LIBRARY_PATH` includes the conda `lib` directory.
- **No data on Redis:** Ensure `redis-server` is running; verify IP/port in scripts (`redis_ip` in `teleop.sh`, `run_motion_server.sh`).
- **Low FPS/lag in teleop:** Reduce `target_fps` in `teleop.sh`; check network; ensure PICO services are healthy.
- **Robot not moving:** Confirm `sim2real.sh` NIC matches your interface; verify ONNX path; check that actions are non-zero in Redis.

---

## Related docs

- [Installation](../GettingStarted/Installation.md)  
- [Environments (`twist2` & `gmr`)](../GettingStarted/Environments.md)  
- [Teleop Pipeline](TeleopPipeline.md)  
- [Sim2Real with Unitree (EN)](Sim2Real_Unitree_en.md)  
- [Sim2Sim Verification](Sim2Sim.md)  
