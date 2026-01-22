# Teleoperation Pipeline

Control your robot through live teleoperation using PICO VR headset, XRoboToolkit, and GMR motion retargeting.

## Overview

The teleoperation pipeline enables real-time control of the robot through human body movements:

1. **VR Capture**: PICO headset and trackers capture body/hand poses
2. **Motion Retargeting**: GMR converts human motion to robot actions
3. **Redis Communication**: Actions are published to Redis channels
4. **Policy Execution**: Low-level controller runs the ONNX policy
5. **Robot Control**: Commands are sent to real hardware or simulation

For installation details, see [Installation](../GettingStarted/Installation.md).

## Prerequisites

### Hardware Requirements

| Component | Specification |
|-----------|---------------|
| **VR Headset** | PICO 4 Ultra (enterprise mode with VST enabled) |
| **Controllers** | PICO VR controllers (left and right) |
| **Trackers** | PICO Motion Trackers compatible with XRoboToolkit |
| **Robot** | Unitree G1/H1/H1_2 (for real deployment) |
| **Safety** | Physical E-stop button (mandatory for real robot) |

### Software Requirements

| Component | Details |
|-----------|---------|
| **XRoboToolkit PC Service** | Installed and running on PC |
| **XRoboToolkit App** | Installed on PICO headset (requires developer mode + adb) |
| **Redis Server** | Running and accessible to both teleop and controller PCs |
| **Conda Environments** | `gmr` (Python 3.10+) for teleop, `twist2` (Python 3.8) for controller |

### Network Requirements

- Stable LAN connection (wired recommended)
- Low latency between teleop PC and controller PC
- Redis server accessible from both machines
- Configure `redis_ip` in [teleop.sh](../../teleop.sh) if using remote Redis

## Quick Start

Choose your deployment target:

### Real Robot Deployment

**Terminal 1**: Start the low-level controller

```bash
conda activate twist2
./sim2real.sh  # Edit NIC inside if needed
```

**Terminal 2**: Start teleoperation

```bash
conda activate gmr
./teleop.sh    # Edit redis_ip/height/target_fps if needed
```

### Simulation Deployment

**Terminal 1**: Start the simulation controller

```bash
conda activate twist2
./sim2sim.sh
```

**Terminal 2**: Start teleoperation

```bash
conda activate gmr
./teleop.sh
```

### Optional: Data Recording

Add a third terminal to record teleoperation data:

```bash
conda activate twist2
python deploy_real/server_data_record.py --task_name teleop_run --data_folder ./logs
```

## Controller Interface

### State Machine

The teleop system operates with the following states:

```
idle → teleop → pause → teleop → ... → exit
```

- **idle**: Waiting for input, no commands sent
- **teleop**: Active control, sending commands to robot
- **pause**: Holding current position, no new commands
- **exit**: Shutdown teleop system

**Auto-start**: The system automatically transitions from idle to teleop when motion data is detected.

### PICO Controller Mapping

Implemented in [xrobot_teleop_to_robot_w_hand.py](../../deploy_real/xrobot_teleop_to_robot_w_hand.py):

#### Left Controller

| Input | Function |
|-------|----------|
| **key_one** | Exit teleop system |
| **axis_click** | Emergency stop (kills sim2real.sh process) |
| **Joystick** | Root XY velocity + yaw velocity |
| **Trigger/Grip** | Left hand open/close (stepwise interpolation) |

#### Right Controller

| Input | Function |
|-------|----------|
| **key_one** | Cycle states (idle → teleop → pause) |
| **Joystick** | Fine-tune root XY/yaw velocity |
| **Trigger/Grip** | Right hand open/close (stepwise interpolation) |

### Control Tips

- Start in a neutral standing pose
- Make smooth, gradual movements
- Keep the E-stop within reach at all times
- Avoid sudden, large-step inputs
- Practice in simulation before real robot deployment

## Redis Communication

### Action Channels (Published by Teleop)

| Channel | Description | Dimension |
|---------|-------------|-----------|
| `action_body_unitree_g1_with_hands` | Body mimic observations | 35-D vector |
| `action_hand_left_unitree_g1_with_hands` | Left hand pose | 7-D vector |
| `action_hand_right_unitree_g1_with_hands` | Right hand pose | 7-D vector |
| `action_neck_unitree_g1_with_hands` | Neck control (optional) | Variable |
| `t_action` | Action timestamp | Scalar |

### State Channels (Published by Controller)

| Channel | Description |
|---------|-------------|
| `state_body_unitree_g1_with_hands` | Robot body state |
| `state_hand_left_*` | Left hand state |
| `state_hand_right_*` | Right hand state |
| `state_neck_*` | Neck state |
| `t_state` | State timestamp |

### Telemetry Channels

| Channel | Description |
|---------|-------------|
| `controller_data` | Button/axis state from PICO controllers |

**Note**: Channel suffixes change based on robot alias. Replace `unitree_g1_with_hands` with your configured robot name.

## Configuration

### Frame Rate Adjustment

Edit [teleop.sh](../../teleop.sh) to adjust target FPS:

```bash
--target_fps 60  # Adjust based on network/GPU performance
```

Enable FPS measurement for debugging:

```bash
--measure_fps
```

### Human Height Calibration

Improve retargeting accuracy by setting your actual height in [teleop.sh](../../teleop.sh):

```bash
--actual_human_height 1.75  # Height in meters
```

### Motion Smoothing

Enable smoothing for steadier commands if motion is jittery:

```bash
--smooth
```

Edit [teleop.sh](../../teleop.sh) to add this flag.

### Remote Redis Configuration

If Redis is on a different machine, update the Redis IP in [teleop.sh](../../teleop.sh):

```bash
redis_ip="192.168.1.100"  # Replace with your Redis server IP
```

Ensure the same IP is used in controller scripts ([sim2real.sh](../../sim2real.sh) or [sim2sim.sh](../../sim2sim.sh)).

### Hand Control Configuration

Enable hand control in [sim2real.sh](../../sim2real.sh):

```bash
--use_hand
```

Ensure hand controllers are powered on and connected.

## Troubleshooting

### Robot Not Responding

**Symptoms**: No motion despite teleop input

**Diagnosis**:
```bash
redis-cli get action_body_unitree_g1_with_hands
```

**Solutions**:
1. Verify teleop script is running in Terminal 2
2. Check Redis IP matches in both terminals
3. Confirm Redis server is running: `redis-cli ping`
4. Review teleop terminal for error messages
5. Verify XRoboToolkit PC Service is running

### Slow Simulation Viewer

**Symptoms**: MuJoCo viewer has low frame rate during teleop

**Solutions**:
1. Lower policy frequency in [sim2sim.sh](../../sim2sim.sh):
   ```bash
   --policy_frequency 50
   ```
2. Disable extra MuJoCo visualizations
3. Verify GPU acceleration is available
4. Close other GPU-intensive applications

### Jerky or Teleporting Motion

**Symptoms**: Robot movements are erratic or jump unexpectedly

**Solutions**:
1. Enable smoothing in [teleop.sh](../../teleop.sh):
   ```bash
   --smooth
   ```
2. Calibrate VR trackers properly
3. Verify correct human height is set:
   ```bash
   --actual_human_height 1.75
   ```
4. Check for wireless interference with PICO headset
5. Ensure stable network connection

### Hand Commands Ignored

**Symptoms**: Hand movements not reflected in robot

**Solutions**:
1. Confirm `--use_hand` flag is set in [sim2real.sh](../../sim2real.sh)
2. Verify hand controllers are powered on and paired
3. Check hand controller battery levels
4. Review controller telemetry in Redis:
   ```bash
   redis-cli get controller_data
   ```
5. Ensure hand action channels are being published

### Neck Not Moving

**Symptoms**: Neck commands not affecting robot head

**Solutions**:
1. Verify neck action updates in Redis:
   ```bash
   redis-cli get action_neck_unitree_g1_with_hands
   ```
2. Confirm neck controller is running (see [Neck Module](../Concepts/NeckModule.md))
3. Check neck module is enabled in deployment script
4. Review neck controller logs for errors

### XRoboToolkit Connection Issues

**Symptoms**: Cannot connect to PICO headset or trackers

**Solutions**:
1. **Ensure same network**: Verify both PICO headset and PC are connected to the same network (WiFi/LAN)
2. Verify XRoboToolkit PC Service is running
3. Check PICO headset is in developer mode
4. Confirm XRoboToolkit app is installed on headset
5. Restart XRoboToolkit PC Service
6. Re-pair PICO headset with PC
7. Check firewall settings for XRoboToolkit
8. **Consult original documentation**: See [XRoboToolkit Unity Client](https://github.com/XR-Robotics/XRoboToolkit-Unity-Client) for detailed setup and troubleshooting

### High Latency

**Symptoms**: Noticeable delay between movement and robot response

**Solutions**:
1. Switch to wired LAN connection
2. Reduce `--target_fps` in [teleop.sh](../../teleop.sh)
3. Move Redis server closer to controller PC
4. Check network bandwidth utilization
5. Verify no packet loss: `ping <redis_ip>`

### Safety Best Practices

1. Always keep physical E-stop accessible
2. Start with slow, small movements
3. Test thoroughly in simulation first
4. Clear the workspace of obstacles
5. Have a second person monitor during testing
6. Practice emergency procedures before deployment

## Next Steps

### Simulation Testing
Start with [Sim2Sim Verification](Sim2Sim.md) to test teleoperation safely.

### Real Robot Deployment
Progress to [Sim2Real with Unitree](Sim2Real_Unitree_en.md) for hardware deployment.

### Training and Development
Review [Training & Deployment Overview](TrainingAndDeployment.md) for the complete workflow.

### Environment Setup
See [Environments](../GettingStarted/Environments.md) for `twist2` and `gmr` configuration details.

### Installation
Refer to [Installation](../GettingStarted/Installation.md) for XRoboToolkit and dependency setup.
