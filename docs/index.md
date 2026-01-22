# TWIST2: Scalable, Portable, and Holistic Humanoid Data Collection System

By Yanjie Ze, Siheng Zhao, Weizhuo Wang, Angjoo Kanazawa†, Rocky Duan†, Pieter Abbeel†, Guanya Shi†, Jiajun Wu†, C. Karen Liu†, 2025  
(† Equal Advising)

[Website](https://yanjieze.com/TWIST2) ·
[arXiv](https://arxiv.org/abs/2511.02832) ·
[Video](https://youtu.be/lTtEvI0kUfo)

![Banner for TWIST](media/TWIST2.png)

---

## Overview

**TWIST2** is a scalable, portable, and holistic data collection system for humanoid robots.  
It combines:

- **Whole-body motion tracking** using VR devices and body trackers  
- **Teleoperation** of real humanoid robots (e.g. Unitree G1 / H1 / H1_2)  
- **RL-based low-level controllers** trained in simulation  
- A modular pipeline for **sim-to-sim** and **sim-to-real** deployment

If you are new to the project, start with:

- **[Installation](GettingStarted/Installation.md)** – set up environments, dependencies, Redis, GMR, and PICO SDK  
- **[Training & Deployment Overview](UserGuide/TrainingAndDeployment.md)** – train a policy, export to ONNX, run sim2sim/sim2real  
- **[Teleop Pipeline](UserGuide/TeleopPipeline.md)** – run online teleoperation + data recording with PICO  
- **[Sim2Real with Unitree](UserGuide/Sim2Real_Unitree_en.md)** – deploy to real Unitree robots

---

## News

- **2025-12-02** – First successful reproduction of TWIST2 appears. Check [this bilibili video](https://www.bilibili.com/video/BV1UbSeBNETw/?share_source=copy_web&vd_source=c76e3ab14ac3f7219a9006b96b4b0f76).
- **2025-12-02** – A video tutorial for TWIST2 will be released to show how to use the system. Stay tuned.
- **2025-12-02** – TWIST2 is open-sourced. Consider giving it a star on GitHub!
  - **Disclaimer 1**: With the current repo, you should be able to control the Unitree G1 via cable connection in both simulation and real, using a PICO VR headset.
  - **Disclaimer 2**: Documentation and some onboard streaming/inference code are still being cleaned up (the teleop pipeline is complex and requires specific hardware setup).
  - **Disclaimer 3**: The high-level policy learning part will be released in a separate repo. It is modified from [iDP3](https://github.com/YanjieZe/Improved-3D-Diffusion-Policy).
- **2025-11-05** – TWIST2 is released. Full code will be released within ~1 month (mostly ready and under internal process).

---

## Core Components

TWIST2 is built around several key components:

- **Environments**
  - `twist2` conda env (Python 3.8) for:
    - Controller training (Isaac Gym + Legged Gym + RSL-RL)
    - Low-level controller deployment (sim and real)
    - Teleop data collection
  - `gmr` conda env (Python 3.10+) for:
    - Online motion retargeting via [GMR](https://github.com/YanjieZe/GMR)
    - PICO streaming and XRoboToolkit bindings

- **Simulation & Control**
  - Isaac Gym-based simulation for humanoid control
  - RL policy training and ONNX export for real-time deployment
  - Sim2Sim and Sim2Real verification pipelines

- **Teleoperation & Data Collection**
  - PICO VR headset + controllers + body trackers
  - XRoboToolkit PC Service + Python bindings
  - Teleop control (online streaming) and data recording scripts
  - Optional neck module and ZED Mini streaming

- **Real Robot Deployment**
  - Support for Unitree G1, H1, and H1_2
  - Ethernet-based deployment from a workstation
  - Redis-based high-level / low-level communication

---

## Quick Links

- **Setup**
  - [Installation](GettingStarted/Installation.md) – full setup, including conda envs, Redis, Unitree SDK, GMR, and PICO SDK
  - *(optional)* Environments overview: `GettingStarted/Environments.md` (if present in your nav)

- **Using TWIST2**
  - [Training & Deployment Overview](UserGuide/TrainingAndDeployment.md)
  - [Teleop Pipeline](UserGuide/TeleopPipeline.md)
  - [Sim2Real with Unitree (EN)](UserGuide/Sim2Real_Unitree_en.md)
  - [GUI Usage](UserGuide/GUI.md) – control everything via `gui.sh` (simulation, real robot, data collection, neck, ZED, etc.)

- **Hardware & Concepts**
  - [Neck Module](Concepts/NeckModule.md)
  - *(optional)* System overview: `Concepts/SystemOverview.md`
  
---

## Citation

If you use TWIST2 in your work, please cite:

```bibtex
@article{ze2025twist2,
  title   = {TWIST2: Scalable, Portable, and Holistic Humanoid Data Collection System},
  author  = {Yanjie Ze and Siheng Zhao and Weizhuo Wang and Angjoo Kanazawa and
             Rocky Duan and Pieter Abbeel and Guanya Shi and Jiajun Wu and
             C. Karen Liu},
  year    = {2025},
  journal = {arXiv preprint arXiv:2511.02832}
}
```

Related works:

```bibtex
@article{ze2025twist,
  title   = {TWIST: Teleoperated Whole-Body Imitation System},
  author  = {Yanjie Ze and Zixuan Chen and Jo{\~a}o Pedro Ara{\'u}jo and Zi-ang Cao and
             Xue Bin Peng and Jiajun Wu and C. Karen Liu},
  year    = {2025},
  journal = {arXiv preprint arXiv:2505.02833}
}

@article{joao2025gmr,
  title   = {Retargeting Matters: General Motion Retargeting for Humanoid Motion Tracking},
  author  = {Joao Pedro Araujo and Yanjie Ze and Pei Xu and Jiajun Wu and C. Karen Liu},
  year    = {2025},
  journal = {arXiv preprint arXiv:2510.02252}
}
```

For questions about the original project, you can contact the authors at `yanjieze@stanford.edu`.