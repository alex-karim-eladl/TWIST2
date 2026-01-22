# GUI Control Center

The TWIST2 GUI is a CustomTkinter-based control center for managing robot services and deployments.

## Quick Start

Launch the GUI from the `twist2` conda environment:

```bash
conda activate twist2
./gui.sh
```

## Prerequisites

- `customtkinter` installed in the `twist2` environment
- SSH alias `g1` configured for your robot PC
- Required scripts present in the repository root

## Overview

The GUI provides centralized control for:

- **Local Services**: Simulation, deployment, teleoperation, and recording
- **Remote G1 Services**: Camera feeds, neck control, and policy deployment via SSH
- **System Controls**: Theme selection, firewall management, and emergency stop

Each service panel displays:
- Real-time status (OFFLINE/ONLINE/STARTING/ERROR)
- Command being executed
- Live output logs
- Control buttons: START, KILL, CLEAR

## Local Services

| Service | Script | Description | Cleanup Command |
|---------|--------|-------------|-----------------|
| **Sim2Sim Deploy** | `sim2sim.sh` | Run simulation-to-simulation deployment | Auto |
| **Sim2Real Deploy** | `sim2real.sh` | Deploy to real robot | `pkill -f server_low_level_g1_real_future.py` |
| **Offline Motion** | `run_motion_server.sh` | Start motion server | Auto |
| **Online Teleop** | `teleop.sh` | Launch teleoperation interface | Auto |
| **Visuomotor Policy** | `deploy_policy.sh` | Deploy vision-based policy | Auto |
| **Data Recording** | `data_record.sh` | Record robot data | `pkill -f server_data_record.py` |

**Quick Launch**: "🚀 Start Sim2Real Deploy & Teleop & Record" starts all three services simultaneously.

## Remote G1 Services

All remote services connect via SSH to host alias `g1` with `StrictHostKeyChecking=no`.

| Service | Remote Script | Description | Cleanup Command |
|---------|---------------|-------------|-----------------|
| **G1 Neck Control** | `~/g1-onboard/docker_neck.sh` | Control robot neck | `pkill -f neck_teleop.py` |
| **G1 ZED Teleop** | `~/g1-onboard/docker_zed.sh` | ZED camera teleoperation | `pkill -9 OrinVideoSender` |
| **G1 ZED Policy** | `~/g1-onboard/docker_zed_policy.sh` | ZED-based policy execution | Auto |

**Additional Tools**:
- `kill_port.sh` - Kill processes on specific ports
- `test_zed.sh` - Test ZED camera connection
- Connection test - Verify SSH connectivity to `g1`

**Quick Launch**: "🚀 Start Neck & ZED Teleop" starts both neck control and ZED teleoperation.

## Global Controls

### Theme Selector
Switch between multiple dark/light/EVA theme variants. Restart may be required for full effect.

### Disable Firewall
Executes `sudo ufw disable`. Use with caution.

### Emergency Stop
Immediately kills all running panel processes.

## Usage Guide

### Starting Services

**Individual Service**:
1. Navigate to the desired service panel
2. Click **START**
3. Monitor the output log for status

**Multiple Services**:
- Use quick launch buttons for common combinations
- Or start services individually as needed

### Monitoring Services

- Check the status indicator (color-coded)
- Review the output log in each panel
- Watch for errors in the terminal view

### Stopping Services

**Individual Service**:
- Click **KILL** on the specific panel
- Verify cleanup in the output log

**All Services**:
- Click **EMERGENCY STOP** to terminate everything

**Clear Logs**:
- Click **CLEAR** to reset the output view

## Configuration

### SSH Host Configuration

If your robot is not reachable as `g1`, update the SSH target in [gui.py](../gui.py):
- Modify `_build_ssh_command` and related SSH calls
- Update the host alias throughout the file

### Script Paths

Edit the following paths in [gui.py](../gui.py) to match your setup:

**Visuomotor Policy Path**:
```bash
/home/ANT.AMAZON.COM/yanjieze/lab42/src/Improved-3D-Diffusion-Policy/deploy_policy.sh
```

**Remote Docker Scripts**:
- `~/g1-onboard/docker_neck.sh`
- `~/g1-onboard/docker_zed.sh`
- `~/g1-onboard/docker_zed_policy.sh`

**Cleanup Commands**:
Adjust `pkill` patterns if your script names differ.

### Sim2Real Configuration

Before launching Sim2Real from the GUI, edit [sim2real.sh](../../sim2real.sh):
- Set the correct network interface
- Update the ONNX checkpoint path

### Firewall Button

The firewall button executes `sudo ufw disable`. To remove or modify this functionality, edit the firewall handler in [gui.py](../gui.py).

## Troubleshooting

### SSH Connection Failures

**Symptoms**: Cannot connect to remote G1 services

**Solutions**:
1. Test manual SSH connection: `ssh g1`
2. Verify SSH key/credentials are configured
3. Check that host alias `g1` exists in `~/.ssh/config`
4. Ensure the robot PC is powered on and network-accessible

### Processes Not Stopping

**Symptoms**: Services remain running after clicking KILL

**Solutions**:
1. Review cleanup commands in the output log
2. Verify `pkill` patterns match your script names
3. Manually kill processes: `pkill -f <script_name>`
4. Use EMERGENCY STOP as last resort

### No Output in Logs

**Symptoms**: Panel shows no output after starting service

**Solutions**:
1. Verify the script exists in the repository root
2. Check script permissions (`chmod +x <script>.sh`)
3. Run the script manually to test: `bash <script>.sh`
4. Review terminal for error messages

### Theme or Display Issues

**Symptoms**: Colors incorrect, UI elements misaligned

**Solutions**:
1. Restart the GUI after changing themes
2. Check `customtkinter` is properly installed
3. Try a different theme variant
4. Restart with default theme

### Service Starts but Immediately Fails

**Symptoms**: Status shows ERROR shortly after STARTING

**Solutions**:
1. Review the output log for error messages
2. Check that all dependencies are installed
3. Verify configuration files are correct
4. Run the script manually to identify issues
