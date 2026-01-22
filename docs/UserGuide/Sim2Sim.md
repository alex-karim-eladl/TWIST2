# Sim2Sim Verification

Validate your ONNX controller in MuJoCo simulation before deploying to real hardware.

## Overview

Sim2Sim allows you to test your trained policy in a safe simulation environment. The controller can be driven by:
- **Motion Server**: Pre-scripted motions for repeatable testing
- **Teleop**: Live teleoperation using PICO controller and GMR

Both methods publish identical Redis action keys consumed by the controller.

## Prerequisites

| Requirement | Details |
|-------------|---------|
| **Conda Environment** | `twist2` (Python 3.8) with MuJoCo, ONNX Runtime, and TWIST2 dependencies |
| **Redis Server** | Running and accessible at `localhost:6379` (default) |
| **ONNX Checkpoint** | Available at `assets/ckpts/twist2_1017_20k.onnx` (default) |
| **GPU Drivers** | CUDA drivers (optional but recommended for performance) |

## Quick Start

### Step 1: Launch the Controller

Start the low-level controller in MuJoCo:

```bash
conda activate twist2
./sim2sim.sh
```

This opens the MuJoCo viewer and starts the policy server.

### Step 2: Choose a Motion Source

**Option A: Scripted Motion** (recommended for initial testing)

```bash
conda activate twist2
./run_motion_server.sh
```

**Option B: Live Teleoperation** (requires PICO controller and GMR setup)

```bash
conda activate gmr
./teleop.sh
```

### Expected Result

The MuJoCo viewer displays the G1 robot executing motions driven by your chosen action source.

## System Architecture

### sim2sim.sh Script

The [sim2sim.sh](../../sim2sim.sh) script launches [server_low_level_g1_sim.py](../../deploy_real/server_low_level_g1_sim.py) with the following configuration:

| Component | Configuration |
|-----------|---------------|
| **MuJoCo Model** | `assets/g1/g1_sim2sim_29dof.xml` |
| **ONNX Policy** | `assets/ckpts/twist2_1017_20k.onnx` |
| **Policy Frequency** | 100 Hz (with internal decimation) |
| **Viewer** | MuJoCo passive viewer for visualization |

### Redis Communication

**Input (Actions Read)**:
- `action_body_unitree_g1_with_hands` - 35-D body action vector
- `action_hand_left_*` - Left hand pose (7-D)
- `action_hand_right_*` - Right hand pose (7-D)
- `action_neck_*` - Neck control commands

**Output (State Published)**:
- `state_body_*` - Body state information
- `state_hand_*` - Hand states (zeros in simulation)
- `state_neck_*` - Neck state
- `t_state` - Timestamp

## Configuration

### Using a Different ONNX Checkpoint

Edit the `ckpt_path` variable in [sim2sim.sh](../../sim2sim.sh):

```bash
ckpt_path="assets/ckpts/your_checkpoint.onnx"
```

### Running Headless

To run without the MuJoCo viewer:

1. Edit [server_low_level_g1_sim.py](../../deploy_real/server_low_level_g1_sim.py)
2. Remove or comment out the `launch_passive` call
3. Alternatively, use a virtual display (e.g., `xvfb-run`)

### Adjusting Policy Frequency

Modify the `--policy_frequency` argument in [sim2sim.sh](../../sim2sim.sh):

```bash
--policy_frequency 50  # Reduces frequency to 50 Hz
```

This affects simulation decimation and update rates.

### Remote Redis Server

If Redis is running on a different machine, update the `redis_ip` parameter in your action source scripts:

**For Motion Server**: Edit [run_motion_server.sh](../../run_motion_server.sh)

**For Teleop**: Edit [teleop.sh](../../teleop.sh)

```bash
redis_ip="192.168.1.100"  # Replace with your Redis server IP
```

## Troubleshooting

### Robot Not Moving

**Symptoms**: MuJoCo viewer shows robot but no motion occurs

**Diagnosis**:
```bash
redis-cli get action_body_unitree_g1_with_hands
```

**Solutions**:
1. Verify your action source (motion server or teleop) is running
2. Check that Redis keys are being published
3. Confirm Redis connection in both controller and action source
4. Review terminal logs for error messages

### Slow or Laggy Viewer

**Symptoms**: MuJoCo viewer has poor frame rate or stuttering

**Solutions**:
1. Lower policy frequency in [sim2sim.sh](../../sim2sim.sh)
2. Disable extra visualizations in MuJoCo
3. Verify GPU acceleration is available:
   ```bash
   nvidia-smi  # Check GPU is accessible
   ```
4. Close other GPU-intensive applications

### Shape or Dimension Errors

**Symptoms**: Controller crashes with shape mismatch errors

**Expected Dimensions**:
- Body actions: 35-D mimic observation vector
- Hand poses: 7-D vectors (position + quaternion)

**Solutions**:
1. Check controller logs for specific dimension mismatches
2. Verify action source is publishing correct shapes
3. Look for JSON parsing errors in logs
4. Confirm ONNX model input/output shapes match expected format

### Teleop Stutter or Jitter

**Symptoms**: Robot movements are jerky when using teleoperation

**Solutions**:
1. Enable smoothing in [teleop.sh](../../teleop.sh):
   ```bash
   --smooth
   ```
2. Reduce target frame rate:
   ```bash
   --target_fps 30
   ```
3. Check network latency if using remote Redis
4. Verify PICO controller connection stability

### ONNX Runtime Errors

**Symptoms**: Errors loading or executing the ONNX model

**Solutions**:
1. Re-export the model using [to_onnx.sh](../../to_onnx.sh)
2. Verify `onnxruntime-gpu` is installed:
   ```bash
   conda activate twist2
   pip list | grep onnxruntime
   ```
3. Check ONNX model file is not corrupted
4. Ensure CUDA version compatibility with ONNX Runtime

### Redis Connection Failed

**Symptoms**: Cannot connect to Redis server

**Solutions**:
1. Verify Redis is running:
   ```bash
   redis-cli ping  # Should return "PONG"
   ```
2. Start Redis if needed:
   ```bash
   redis-server
   ```
3. Check firewall rules if using remote Redis
4. Confirm correct Redis IP and port in scripts

## Next Steps

### Deploy to Real Hardware
Continue to [Sim2Real with Unitree](Sim2Real_Unitree_en.md) for real robot deployment.

### Understand Teleoperation
Review [Teleop Pipeline](TeleopPipeline.md) for detailed teleop setup and usage.

### Training and Export
See [Training & Deployment Overview](TrainingAndDeployment.md) for the complete workflow from training to deployment.
