# System Overview

A high-level architectural guide to the TWIST2 stack: system components, data flow, and deployment workflows.

## Architecture Overview

TWIST2 is a modular robotics framework that separates motion generation, policy execution, and robot control into distinct components communicating via Redis.

### System Components

The TWIST2 stack consists of five primary components:

1. **Motion Sources** - Generate high-level motion commands
2. **Message Bus** - Distribute actions and states across processes
3. **Policy Controllers** - Execute trained policies and compute robot commands
4. **Robot/Simulator** - Physical hardware or simulation environment
5. **Optional Services** - Data recording and GUI control

## Component Details

### Motion Sources (High-Level)

Motion sources generate action commands published to Redis.

| Source Type | Implementation | Launch Script | Environment | Description |
|-------------|----------------|---------------|-------------|-------------|
| **Teleoperation** | [xrobot_teleop_to_robot_w_hand.py](../../deploy_real/xrobot_teleop_to_robot_w_hand.py) | [teleop.sh](../../teleop.sh) | `gmr` | PICO VR + XRoboToolkit + GMR retargeting |
| **Scripted Motion** | [server_motion_lib.py](../../deploy_real/server_motion_lib.py) | [run_motion_server.sh](../../run_motion_server.sh) | `twist2` | Pre-recorded motion playback |

### Message Bus

| Component | Technology | Purpose |
|-----------|------------|---------|
| **Redis** | Redis server | Publish/subscribe message bus for actions and states |

**Key Features**:
- Enables process and machine distribution
- Allows swappable motion sources and destinations
- Supports sim-to-real transfer without code changes

### Policy Controllers (Low-Level)

Policy controllers execute ONNX models and send PD targets to robots/simulators.

| Controller Type | Implementation | Launch Script | Environment | Target |
|-----------------|----------------|---------------|-------------|--------|
| **Simulation** | [server_low_level_g1_sim.py](../../deploy_real/server_low_level_g1_sim.py) | [sim2sim.sh](../../sim2sim.sh) | `twist2` | MuJoCo simulation |
| **Real Robot** | [server_low_level_g1_real.py](../../deploy_real/server_low_level_g1_real.py) | [sim2real.sh](../../sim2real.sh) | `twist2` | Unitree G1/H1/H1_2 |

### Robot/Simulator Platforms

| Platform | Models | Description |
|----------|--------|-------------|
| **Physical Robots** | Unitree G1, H1, H1_2 | Real humanoid robots receiving PD targets |
| **Simulation** | MuJoCo | Physics-based simulation environment |

### Optional Services

| Service | Implementation | Launch Script | Environment | Purpose |
|---------|----------------|---------------|-------------|---------|
| **Data Recording** | [server_data_record.py](../../deploy_real/server_data_record.py) | Direct Python | `twist2` | Episode recording with stereo vision |
| **GUI Control** | [gui.py](../../gui.py) | [gui.sh](../../gui.sh) | `twist2` | Centralized launch interface |

## Data Flow

### Communication Architecture

![Data Flow Pipeline](../media/Dataflow_Pipeline.png)

### Redis Channels

**Action Channels** (Motion Source → Controller):
- `action_body_unitree_g1_with_hands` - Body motion commands
- `action_hand_left_*` - Left hand pose
- `action_hand_right_*` - Right hand pose
- `action_neck_*` - Neck control (optional)
- `t_action` - Action timestamp

**State Channels** (Controller → System):
- `state_body_unitree_g1_with_hands` - Robot body state
- `state_hand_left_*` - Left hand state
- `state_hand_right_*` - Right hand state
- `state_neck_*` - Neck state (optional)
- `t_state` - State timestamp

### Modularity Benefits

The Redis-based architecture enables:
- **Swappable motion sources**: Switch between teleop and scripted motions without changing controllers
- **Swappable destinations**: Test in simulation before deploying to real hardware
- **Distributed deployment**: Run components on different machines over LAN
- **Independent development**: Develop and test components in isolation

## Prerequisites

### Software Requirements

| Requirement | Details |
|-------------|---------|
| **Conda Environment: `twist2`** | Python 3.8, Isaac Gym, MuJoCo, ONNX Runtime, policy deployment |
| **Conda Environment: `gmr`** | Python 3.10+, GMR retargeting, XRoboToolkit, PICO SDK (teleoperation only) |
| **Redis Server** | Running and accessible on localhost or LAN IP |
| **GPU Drivers** | CUDA required for Isaac Gym, ONNX GPU acceleration, GMR |
| **XRoboToolkit** | PICO PC Service and headset app (teleoperation only) |

**Setup Guides**:
- [Installation](../GettingStarted/Installation.md) - Complete installation instructions
- [Environments](../GettingStarted/Environments.md) - Conda environment setup

### Hardware Requirements (Teleoperation)

| Component | Specification |
|-----------|---------------|
| **VR Headset** | PICO 4 Ultra (enterprise mode with VST) |
| **Controllers** | PICO VR controllers (left and right) |
| **Trackers** | PICO Motion Trackers |

### Hardware Requirements (Real Robot)

| Component | Specification |
|-----------|---------------|
| **Robot** | Unitree G1/H1/H1_2 |
| **Workstation** | PC with Ethernet port, NVIDIA GPU |
| **Network** | Wired Ethernet connection |
| **Safety** | Physical E-stop button |

## Next Steps

### Getting Started
- **Installation**: [Installation Guide](../GettingStarted/Installation.md)
- **Environment Setup**: [Conda Environments](../GettingStarted/Environments.md)

### Deployment Guides
- **Simulation Testing**: [Sim2Sim Verification](../UserGuide/Sim2Sim.md)
- **Real Robot Deployment**: [Sim2Real with Unitree](../UserGuide/Sim2Real_Unitree_en.md)
- **Teleoperation Pipeline**: [Teleop Pipeline](../UserGuide/TeleopPipeline.md)

### Advanced Topics
- **Neck Control Module**: [Neck Module](NeckModule.md)
- **GUI Control Interface**: [GUI Guide](../UserGuide/GUI.md)
- **Training & Deployment**: [Training Overview](../UserGuide/TrainingAndDeployment.md)
