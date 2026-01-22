# Installation

This page walks you through setting up **TWIST2** from scratch.

TWIST2 uses **two separate conda environments**:

- `twist2` – for **controller training**, **controller deployment**, and **teleop data collection** (Isaac Gym + Mujoco, etc.).
- `gmr` – for **online motion retargeting** with [GMR](https://github.com/YanjieZe/GMR) and PICO streaming.

This separation is necessary because:

- **Isaac Gym** currently requires **Python 3.8**
- **Newest Mujoco + GMR** require **Python 3.10+**

If you just want to *run* the provided controller and teleop (and not train), you can still follow the same setup but skip the dataset download step.

---

## 1. Create the `twist2` conda environment

```bash
conda env remove -n twist2          # optional, only if it already exists
conda create -n twist2 python=3.8
conda activate twist2
```

This environment will be used for:

* Training the controller
* Exporting the policy to ONNX
* Sim-to-sim / sim-to-real deployment
* Teleop data collection (low-level control side)

---

## 2. Install Isaac Gym

Download Isaac Gym from the **official NVIDIA link**:

* [https://developer.nvidia.com/isaac-gym](https://developer.nvidia.com/isaac-gym)

Then install it into the **activated** `twist2` environment:

```bash
cd isaacgym/python
pip install -e .
```

Make sure `pip` is coming from the `twist2` environment (you can check with `which pip`).

---

## 3. Install TWIST2 Python dependencies (twist2 env)

With `twist2` activated, install the local packages:

```bash
cd rsl_rl && pip install -e . && cd ..
cd legged_gym && pip install -e . && cd ..
cd pose && pip install -e . && cd ..
```

Then install the remaining Python dependencies:

```bash
pip install "numpy==1.23.0" pydelatin wandb tqdm opencv-python ipdb pyfqmr \
           flask dill gdown hydra-core imageio[ffmpeg] mujoco \
           mujoco-python-viewer isaacgym-stubs pytorch-kinematics \
           rich termcolor zmq

# Redis communication
pip install "redis[hiredis]"

# Voice control
pip install pyttsx3

# ONNX inference
pip install onnx onnxruntime-gpu

# Simple GUI wrapper
pip install customtkinter
```

---

## 4. Install and configure Redis server

TWIST2 uses **Redis** as a lightweight communication layer between high-level and low-level components.

Install and start Redis on your machine:

```bash
sudo apt update
sudo apt install -y redis-server

sudo systemctl enable redis-server
sudo systemctl start redis-server
```

Then edit the Redis config (for local, trusted networks only):

```bash
sudo nano /etc/redis/redis.conf
```

Change the following lines:

```bash
bind 0.0.0.0
protected-mode no
```

> ⚠️ **Security note**
> This configuration exposes Redis on all interfaces without protection.
> Only use this in a **trusted internal network** or tighten it accordingly.

Restart Redis to apply changes:

```bash
sudo systemctl restart redis-server
```

If this is your **first time** using Redis, you’re done after this step.

---

## 5. (Optional) Install Unitree SDK for sim2real from laptop

If you want to do **sim-to-real deployment using a laptop** (controlling a Unitree robot from your laptop), you must install a modified Unitree SDK:

> You **do not** need this on your laptop if you only deploy from the robot’s onboard computer.

```bash
# From the TWIST2 repo root
cd ..
git clone https://github.com/YanjieZe/unitree_sdk2.git
cd unitree_sdk2
```

Install system-level dependencies:

```bash
sudo apt-get update
sudo apt-get install -y build-essential cmake python3-dev python3-pip pybind11-dev
```

Install Python dependencies:

```bash
pip install pybind11 pybind11-stubgen numpy
```

Build the Python SDK binding:

```bash
cd python_binding
export UNITREE_SDK2_PATH=$(pwd)/..
bash build.sh --sdk-path $UNITREE_SDK2_PATH
```

Install the compiled module into your active conda environment:

```bash
# Get the site-packages path for your current conda environment
SITE_PACKAGES=$(python -c "import site; print(site.getsitepackages()[0])")
echo "Installing to: $SITE_PACKAGES"

# Copy the compiled module (rename to remove version-specific suffix)
sudo cp build/lib/unitree_interface.cpython-*-linux-gnu.so \
        $SITE_PACKAGES/unitree_interface.so
```

Verify installation:

```bash
python -c "import unitree_interface; print('✓ Unitree SDK Python binding installed successfully')"
python -c "import unitree_interface; print('Available robot types:', list(unitree_interface.RobotType.__members__.keys()))"
```

Then go back to the TWIST2 repo:

```bash
cd ../..
```

---

## 6. (Optional) Download TWIST2 dataset

If you want to **train your own controller** instead of only using the provided checkpoint:

1. Download the TWIST2 dataset from:
   [https://drive.google.com/file/d/1JbW_InVD0ji5fvsR5kz7nbsXSXZQQXpd/view?usp=sharing](https://drive.google.com/file/d/1JbW_InVD0ji5fvsR5kz7nbsXSXZQQXpd/view?usp=sharing)

2. Unzip it to any folder you like.

3. Set the `root_path` in:

   ```text
   legged_gym/motion_data_configs/twist2_dataset.yaml
   ```

   to point to the unzipped dataset directory.

> 💡 We also provide a small set of example motions in `assets/example_motions`.
> These are enough to **test the system** without downloading the full dataset.

We also provide our trained controller checkpoint at:

```text
assets/ckpts/twist2_1017_20k.onnx
```

You can use this directly for deployment without training.

---

## 7. Create the `gmr` conda environment

The `gmr` environment is used for:

* **Online motion retargeting**
* **GMR inference**
* **PICO streaming Python SDK**

Create and activate it:

```bash
conda create -n gmr python=3.10 -y
conda activate gmr
```

---

## 8. Install GMR for online retargeting

Clone and install [GMR](https://github.com/YanjieZe/GMR):

```bash
git clone https://github.com/YanjieZe/GMR.git
cd GMR

# Install GMR in editable mode
pip install -e .
cd ..
```

Install additional dependency:

```bash
conda install -c conda-forge libstdcxx-ng -y
```

---

## 9. Install PICO SDK & XRoboToolkit (for VR teleop)

TWIST2 uses **PICO devices** with **XRoboToolkit** for VR-based teleoperation.

### 9.1 Install PICO SDK on the headset

On your **PICO headset**, install the PICO SDK:

* Follow the instructions / releases at:
  [https://github.com/XR-Robotics/XRoboToolkit-Unity-Client/releases/](https://github.com/XR-Robotics/XRoboToolkit-Unity-Client/releases/)

### 9.2 Install XRoboToolkit PC Service (Ubuntu)

On your **PC** (Ubuntu 22.04):

1. Download the `.deb` package:

   * [https://github.com/XR-Robotics/XRoboToolkit-PC-Service/releases/download/v1.0.0/XRoboToolkit_PC_Service_1.0.0_ubuntu_22.04_amd64.deb](https://github.com/XR-Robotics/XRoboToolkit-PC-Service/releases/download/v1.0.0/XRoboToolkit_PC_Service_1.0.0_ubuntu_22.04_amd64.deb)

2. Install it:

   ```bash
   sudo dpkg -i XRoboToolkit_PC_Service_1.0.0_ubuntu_22.04_amd64.deb
   ```

3. After installation, you should see `xrobotoolkit-pc-service` in your applications.
   Make sure to **start this app** before doing teleoperation.

### 9.3 Build PICO PC Service SDK & Python bindings

Still in the **`gmr`** environment:

```bash
conda activate gmr

git clone https://github.com/YanjieZe/XRoboToolkit-PC-Service-Pybind.git
cd XRoboToolkit-PC-Service-Pybind
```

Clone the XRoboToolkit PC Service source and build the C++ SDK:

```bash
mkdir -p tmp
cd tmp
git clone https://github.com/XR-Robotics/XRoboToolkit-PC-Service.git

cd XRoboToolkit-PC-Service/RoboticsService/PXREARobotSDK
bash build.sh
cd ../../../..
```

Prepare headers and library for the Python binding:

```bash
mkdir -p lib
mkdir -p include
cp tmp/XRoboToolkit-PC-Service/RoboticsService/PXREARobotSDK/PXREARobotSDK.h include/
cp -r tmp/XRoboToolkit-PC-Service/RoboticsService/PXREARobotSDK/nlohmann include/nlohmann/
cp tmp/XRoboToolkit-PC-Service/RoboticsService/PXREARobotSDK/build/libPXREARobotSDK.so lib/
# rm -rf tmp   # optional cleanup
```

Install Python dependencies and build the pybind module:

```bash
conda install -c conda-forge pybind11
pip uninstall -y xrobotoolkit_sdk
python setup.py install
```

Then go back:

```bash
cd ..
```

---

## 10. You’re ready to go

At this point, you should have:

* `twist2` environment with:

  * Isaac Gym
  * Legged Gym / RSL-RL / pose
  * TWIST2 dependencies
  * Redis server running locally

* `gmr` environment with:

  * GMR installed
  * XRoboToolkit PC service & Python bindings for PICO streaming

From here, you can:

* Follow the **[Training & Deployment](../UserGuide/TrainingAndDeployment.md)** guide to:

  * Train a new controller
  * Export ONNX
  * Run sim2sim / sim2real
* Follow the **[Teleop Pipeline](../UserGuide/TeleopPipeline.md)** guide to:

  * Set up PICO teleoperation
  * Record teleop data

