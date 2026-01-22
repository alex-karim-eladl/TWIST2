# Sim2Real Deployment with Unitree

Deploy the TWIST2 ONNX controller to Unitree G1/H1/H1_2 robots for real-world execution.

## Overview

This guide covers the complete deployment workflow for running trained policies on physical Unitree robots:

1. **Network Configuration**: Set up communication between workstation and robot
2. **Robot Preparation**: Safely initialize the robot in debugging mode
3. **Controller Deployment**: Launch the low-level ONNX policy controller
4. **Motion Source**: Drive the robot via teleoperation or scripted motions
5. **Safe Operation**: Follow proper startup and shutdown procedures

**Important**: Always test your policy in [Sim2Sim](Sim2Sim.md) before real robot deployment.

## Prerequisites

### Hardware Requirements

| Component | Specification |
|-----------|---------------|
| **Robot** | Unitree G1/H1/H1_2 in working condition |
| **Network Cable** | Wired Ethernet connection (Wi-Fi not recommended for control) |
| **Safety Equipment** | Physical E-stop button, robot hoist/support stand |
| **Workstation** | PC with Ethernet port and NVIDIA GPU (recommended) |
| **Remote Control** | Unitree remote controller |

### Software Requirements

| Component | Details |
|-----------|---------|
| **Conda Environment: `twist2`** | Python 3.8 with ONNX Runtime (GPU), MuJoCo/Isaac dependencies, Redis client |
| **Conda Environment: `gmr`** | Python 3.10+ with GMR, XRoboToolkit, PICO SDK (for teleoperation) |
| **Redis Server** | Running and accessible (default: localhost:6379) |
| **ONNX Checkpoint** | Trained policy (default: `assets/ckpts/twist2_1017_20k.onnx`) |

### Network Configuration

Configure your workstation with a static IP on the Unitree network:

| Device | Default IP | Notes |
|--------|------------|-------|
| **Robot** | `192.168.123.164` | Unitree default |
| **LiDAR** | `192.168.123.120` | If equipped |
| **Workstation** | `192.168.123.222` | Set static IP with subnet `255.255.255.0` |

**Find Your Network Interface**:
```bash
ifconfig  # or: ip a
```

Look for the interface connected to the robot (e.g., `enp3s0`, `eno1`). You'll need this name for [sim2real.sh](../../sim2real.sh).

## Initial Setup

### Step 1: Network Connection

1. Connect workstation to robot via wired Ethernet
2. Configure static IP on your network interface:
   ```bash
   sudo ip addr add 192.168.123.222/24 dev enp3s0  # Replace enp3s0 with your interface
   ```
3. Verify connection:
   ```bash
   ping 192.168.123.164  # Should receive responses from robot
   ```

### Step 2: Robot Preparation

1. **Hoist the robot**: Securely suspend the robot to prevent falls during initialization
2. **Power on**: Start the robot and wait for boot completion
3. **Enter debugging mode**: Press `L2 + R2` on the Unitree remote controller
   - Joints become damped
   - Robot enters zero torque mode (joints feel loose)
4. **Verify zero torque mode**: Manually move joints to confirm they're unpowered

### Step 3: Configure Deployment Script

Edit [sim2real.sh](../../sim2real.sh) to match your setup:

```bash
# Network interface name (from Step 1)
NIC="enp3s0"

# ONNX checkpoint path
CKPT="assets/ckpts/twist2_1017_20k.onnx"

# Enable hand control (only if hands are powered)
# --use_hand

# Enable body motion smoothing (recommended)
# --smooth_body
```

## Deployment Workflow

### Teleoperation Mode

**Terminal 1**: Start the low-level controller

```bash
conda activate twist2
./sim2real.sh  # Edit NIC and checkpoint path inside
```

**What it does**:
- Connects to robot via specified network interface
- Reads `action_*` commands from Redis
- Builds TWIST2 observations from robot state
- Runs ONNX policy inference
- Sends PD targets to robot actuators
- Publishes `state_*` feedback to Redis

**Terminal 2**: Start teleoperation publisher

```bash
conda activate gmr
./teleop.sh  # Edit redis_ip/height/target_fps if needed
```

**What it does**:
- Captures body/hand poses from PICO VR headset and trackers
- Retargets human motion to robot actions via GMR
- Publishes `action_body`, `action_hand_left`, `action_hand_right`, `action_neck` to Redis

**Terminal 3** (Optional): Start data recording

```bash
conda activate twist2
python deploy_real/server_data_record.py --task_name sim2real_run --data_folder ./logs
```

### Scripted Motion Mode

**Terminal 1**: Start the low-level controller

```bash
conda activate twist2
./sim2real.sh
```

**Terminal 2**: Start scripted motion server

```bash
conda activate twist2
./run_motion_server.sh  # Streams pre-defined motions to Redis
```

## Robot Operation

### Startup Sequence

1. **Controller initialized**: Terminal 1 shows successful connection
   - Robot remains in zero torque mode
   - Joints are loose and passive

2. **Move to default pose**: Press **Start** on Unitree remote
   - Robot moves to default standing position
   - **Keep robot hoisted during this step**
   - Wait for stable pose before proceeding

3. **Lower robot**: Once default pose is stable, gently lower from hoist
   - Ensure all feet contact ground evenly
   - Robot should maintain balance

4. **Activate motion control**: Press **A** button on Unitree remote
   - Robot enters stepping mode
   - Ready to receive motion commands

### Control Mapping (Unitree Remote)

| Button/Stick | Function |
|--------------|----------|
| **L2 + R2** | Enter debugging mode (damped joints) |
| **Start** | Move to default pose |
| **A** | Activate stepping/motion control |
| **Select** | Exit to damping mode |
| **Left Stick** | XY velocity control |
| **Right Stick** | Yaw velocity control |

### Shutdown Sequence

**Safe Shutdown**:
1. Press **Select** on Unitree remote
2. Robot returns to damping mode
3. Press `Ctrl+C` in Terminal 1 to stop controller
4. Press `Ctrl+C` in Terminal 2 to stop motion source

**Emergency Shutdown**:
1. Press physical E-stop button immediately
2. Kill all processes: `pkill -f server_low_level_g1_real_future.py`

## Redis Communication

### Action Channels (Controller Inputs)

| Channel | Description | Dimension |
|---------|-------------|-----------|
| `action_body_unitree_g1_with_hands` | Body mimic observations | 35-D vector |
| `action_hand_left_unitree_g1_with_hands` | Left hand pose | 7-D vector |
| `action_hand_right_unitree_g1_with_hands` | Right hand pose | 7-D vector |
| `action_neck_unitree_g1_with_hands` | Neck control commands | Variable |
| `t_action` | Action timestamp | Scalar |

### State Channels (Controller Outputs)

| Channel | Description |
|---------|-------------|
| `state_body_unitree_g1_with_hands` | Robot body state (joint positions, velocities) |
| `state_hand_left_unitree_g1_with_hands` | Left hand state |
| `state_hand_right_unitree_g1_with_hands` | Right hand state |
| `state_neck_unitree_g1_with_hands` | Neck state |
| `t_state` | State timestamp |

### Motion Server Gating Signals

| Channel | Description |
|---------|-------------|
| `motion_start_signal` | Trigger to start scripted motion |
| `motion_exit_signal` | Signal to exit scripted motion |

**Note**: Channel suffixes change based on robot alias. Replace `unitree_g1_with_hands` with your configured robot name if different.

## Safety Guidelines

### Pre-Deployment Checklist

- [ ] Policy validated in Sim2Sim environment
- [ ] E-stop button accessible and tested
- [ ] Robot securely hoisted for initial startup
- [ ] Wired Ethernet connection established (no Wi-Fi)
- [ ] Network interface configured correctly in [sim2real.sh](../../sim2real.sh)
- [ ] ONNX checkpoint path verified
- [ ] Redis server running: `redis-cli ping`
- [ ] Redis action keys publishing: `redis-cli get action_body_unitree_g1_with_hands`
- [ ] Workspace cleared of obstacles and bystanders
- [ ] Hand controllers powered on (if using `--use_hand`)

### During Operation

1. **Always maintain E-stop access**: Keep button within immediate reach
2. **Start hoisted**: Never enable torque with robot on the ground initially
3. **Verify default pose**: Ensure stable standing position before lowering
4. **Lower gently**: Control descent from hoist carefully
5. **Monitor continuously**: Watch for unexpected behaviors or instability
6. **Use wired connection**: Never rely on Wi-Fi for control loop
7. **Disable unused features**: Keep `--use_hand` off if hands are unpowered
8. **Test in stages**: Start with small motions before complex tasks

### Emergency Procedures

1. **Physical E-stop**: Primary emergency response
2. **Select button**: Software exit to damping mode
3. **Ctrl+C**: Terminate controller process
4. **Power off**: Robot main power switch as last resort

## Troubleshooting

### Robot Not Moving

**Symptoms**: Controller running but robot remains motionless

**Diagnosis**:
```bash
# Check network interface
ifconfig enp3s0  # Verify IP is configured

# Check Redis actions
redis-cli get action_body_unitree_g1_with_hands

# Check robot connectivity
ping 192.168.123.164
```

**Solutions**:
1. Verify network interface name in [sim2real.sh](../../sim2real.sh) matches actual interface
2. Confirm workstation IP is correctly set (`192.168.123.222/24`)
3. Check action source (teleop/motion server) is publishing to Redis
4. Verify ONNX checkpoint path is correct and file exists
5. Ensure robot is in motion control mode (pressed **A** button)
6. Review controller terminal output for errors

### Motion Jitter or Lag

**Symptoms**: Jerky movements, delayed response, unstable control

**Solutions**:
1. Enable body smoothing in [sim2real.sh](../../sim2real.sh):
   ```bash
   --smooth_body
   ```
2. Verify GPU is available and not overloaded:
   ```bash
   nvidia-smi
   ```
3. Check network quality:
   ```bash
   ping -c 100 192.168.123.164  # Look for packet loss or high latency
   ```
4. Inspect Ethernet cable and switch/hub quality
5. Close unnecessary GPU-intensive applications
6. Reduce teleoperation target FPS in [teleop.sh](../../teleop.sh) if using teleop

### Hands or Neck Not Responding

**Symptoms**: Hand or neck modules remain idle despite commands

**Solutions**:
1. Verify `--use_hand` flag is enabled in [sim2real.sh](../../sim2real.sh)
2. Check hand controllers are powered on and connected
3. Verify neck action updates in Redis:
   ```bash
   redis-cli get action_neck_unitree_g1_with_hands
   ```
4. Ensure hand batteries are charged
5. Review hand controller logs for errors
6. Confirm neck module is enabled in deployment configuration

### ONNX Runtime Errors

**Symptoms**: Controller fails to load or execute ONNX model

**Solutions**:
1. Re-export the ONNX model:
   ```bash
   ./to_onnx.sh
   ```
2. Verify `onnxruntime-gpu` is installed in `twist2` environment:
   ```bash
   conda activate twist2
   pip list | grep onnxruntime
   ```
3. Check CUDA version compatibility with ONNX Runtime
4. Confirm ONNX checkpoint file is not corrupted:
   ```bash
   ls -lh assets/ckpts/twist2_1017_20k.onnx
   ```
5. Review controller logs for specific error messages

### Connection Drops

**Symptoms**: Controller loses connection to robot intermittently

**Solutions**:
1. Inspect Ethernet cable for damage
2. Test cable with different devices
3. Replace network switch/hub if present
4. Verify cable is properly seated in both workstation and robot
5. Restart controller after network link is stable:
   ```bash
   pkill -f server_low_level_g1_real_future.py
   ./sim2real.sh
   ```
6. Check for electromagnetic interference near cables
7. Use shielded Ethernet cable if interference suspected

### Redis Connection Issues

**Symptoms**: Controller cannot connect to Redis server

**Solutions**:
1. Verify Redis is running:
   ```bash
   redis-cli ping  # Should return "PONG"
   ```
2. Start Redis if needed:
   ```bash
   redis-server
   ```
3. Check Redis IP configuration in scripts
4. Verify firewall allows Redis port (6379)
5. Test Redis connectivity:
   ```bash
   redis-cli -h localhost -p 6379
   ```

### Robot Falls or Becomes Unstable

**Symptoms**: Robot loses balance or falls during operation

**Immediate Actions**:
1. **Press E-stop immediately**
2. Support robot to prevent damage
3. Press **Select** on remote to enter damping mode
4. Kill controller process: `Ctrl+C`

**Prevention**:
1. Always start with robot hoisted
2. Verify stable default pose before lowering
3. Test policy thoroughly in simulation first
4. Start with slow, conservative motions
5. Ensure even ground surface
6. Monitor battery level (low battery affects stability)


## Next Steps

### Validate in Simulation First
**Always start here**: [Sim2Sim Verification](Sim2Sim.md) - Test your policy safely in MuJoCo before real robot deployment.

### Master Teleoperation
Learn the complete teleoperation pipeline: [Teleop Pipeline](TeleopPipeline.md)

### Understand the Full Workflow
Review training and deployment: [Training & Deployment Overview](TrainingAndDeployment.md)

### Advanced Topics
- Custom policy training
- Multi-robot coordination
- Vision-based policies
- Task-specific retargeting  
