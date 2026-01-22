# Environments

TWIST2 uses **two conda environments** because its dependencies span different Python versions and GPU stacks:

- **`twist2` (Python 3.8)** – Isaac Gym + Legged Gym + RSL-RL + Mujoco for controller **training**, **policy export**, **sim2sim**, **sim2real**, GUI control, and low-level teleop/data-collection scripts.
- **`gmr` (Python 3.10+)** – [GMR](https://github.com/YanjieZe/GMR), XRoboToolkit bindings, and PICO streaming for **online motion retargeting** during teleop.

See the full install steps in [Installation](Installation.md). This page summarizes what each env is for and the minimal commands to create/use them.

---

## Quick setup (one-time per machine)

```bash
# 1) Create the TWIST2 control/training env
conda create -n twist2 python=3.8

# 2) Create the GMR/teleop env
conda create -n gmr python=3.10
```

Then follow the package installs in the Installation guide for each env (Isaac Gym, Mujoco, RSL-RL, Legged Gym, pose, Redis client, ONNX in `twist2`; GMR + XRoboToolkit + PICO SDK in `gmr`).

---

## When to use each environment

- Use **`twist2`** when you:
  - Train controllers or export ONNX policies.
  - Run **sim2sim** (`sim2sim.sh`) or **sim2real** (`sim2real.sh`) loops.
  - Launch the GUI (`gui.sh`) to drive sim/real/recording.
  - Run low-level control + data collection on the robot side.
- Use **`gmr`** when you:
  - Stream teleop from the PICO headset/trackers.
  - Run retargeting/streaming scripts such as `teleop.sh` (which calls `xrobot_teleop_to_robot_w_hand.py`).
  - Test GMR visualization or retargeting offline.

Both envs rely on a running Redis server for pub/sub of actions and state.

---

## Day-to-day usage

Activate the right env before running a script:

```bash
# Control / training / sim2sim / sim2real / GUI
conda activate twist2

# Teleop streaming + retargeting (PICO + GMR)
conda activate gmr
```

Typical command pairs:

- **Teleop to robot**: 
```bash
conda activate twist2 && ./sim2real.sh
``` 
(low-level control) in one terminal; 
```bash
conda activate gmr && ./teleop.sh
``` 
(teleop publisher) in another.
- **Motion playback**: 
```bash
conda activate twist2 && ./sim2real.sh
```
plus
```bash
conda activate twist2 && ./run_motion_server.sh
```
although it's not really recommended to do it this way as the bot may cause mayhem!
- **Simulation check**: 
```bash
conda activate twist2 && ./sim2sim.sh
```
, optionally driven by `teleop.sh` (from `gmr`) or `run_motion_server.sh`.

---

## Tips and common issues

- Verify `python` points to the expected env (`which python`) before installing packages.
- Isaac Gym requires **Python 3.8** and CUDA support; if imports fail, re-check that you installed it inside `twist2` and that `LD_LIBRARY_PATH` includes your conda `lib` (see `run_motion_server.sh` for an example).
- If Redis keys are missing at runtime, confirm Redis is running locally (`redis-cli ping`) and that both envs point to the same Redis host/IP.
- For PICO/XRoboToolkit streams, keep `gmr` active and ensure the PICO SDK services are running before launching teleop.

---

## Where to go next

- Full installation instructions: [Installation](Installation.md)
- Teleop walkthrough: [Teleop Pipeline](../UserGuide/TeleopPipeline.md)
- Sim2real deployment: [Sim2Real with Unitree (EN)](../UserGuide/Sim2Real_Unitree_en.md)
