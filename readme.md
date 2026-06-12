# Intelligent Task Offloading in Edge-Cloud Continuum Environment

<div align="center">

![Python](https://img.shields.io/badge/Python-3.10+-blue?logo=python)
![Kubernetes](https://img.shields.io/badge/MicroK8s-Kubernetes-326CE5?logo=kubernetes)
![OpenStack](https://img.shields.io/badge/Cloud-OpenStack-ED1944?logo=openstack)
![Open5GS](https://img.shields.io/badge/5G_Core-Open5GS-00ADEF)
![srsRAN](https://img.shields.io/badge/RAN-srsRAN-orange)
![Docker](https://img.shields.io/badge/Deploy-Docker-2496ED?logo=docker)
![License](https://img.shields.io/badge/License-CC%20BY--NC%204.0-lightgrey)

</div>

---

## System Architecture

<!-- Add your architecture diagram here -->
<img width="378" height="640" alt="system" src="https://github.com/user-attachments/assets/42f2405e-b18d-4347-a126-ef8d08f264a5" />


---

## Overview

This project implements a **real-time intelligent task offloading framework** for a private 5G network connected to an edge-cloud computing continuum. Mobile users (UEs) connected to a private 5G SA network send image classification requests; the system dynamically decides — for every request — whether to process it on an edge node, offload it to a cloud backup, or handle it locally.

The system combines three scheduling strategies:

- **MILP-based Task Scheduling** — Mixed Integer Linear Programming that minimizes latency subject to deadline and resource constraints, using real-time UE bandwidth and edge utilization.
- **DQN-based AI Scheduler** — A Deep Q-Network trained on historical scheduling data that selects the best edge pod based on predicted reward.
- **Heuristic Fallback** — CPU / memory / queue threshold-based routing when the ML model is unavailable.

Processed results and inference outputs are uploaded to a persistent cloud storage server for future model retraining.

## Testbed Configuration

| Component | Software/Hardware Version/Model |
|-----------|---------------------------------|
| Operating System | Ubuntu LTS 22.04 |
| Rewritable SIM | Programmable SIM LTE/5G |
| SDR Hardware | USRP B210 |
| UE Stack | srsRAN Latest |
| 5G Core Network | Open5GS 2.7.x |
| Edge Orchestration | MicroK8s Latest |
| Container Runtime | Docker Latest |
| Optimization Solver | CBC/HiGHS Latest |
| Programming Language | Python 3.10 |
| API Framework | FastAPI Latest |
| Cloud Virtualization | VMware Workstation |

---

## Repository Structure

```
Offloading/
│
├── Build/
│   ├── DQL_Scheduler/              # Deep Q-Network based AI Scheduler
│   │   ├── ai_scheduler.py         # FastAPI scheduler with DQN inference
│   │   ├── Dockerfile
│   │   ├── DQL_Training.ipynb      # Training notebook
│   │   ├── hybrid_dqn.pt           # Trained DQN model weights
│   │   ├── state_scaler.pkl        # MinMaxScaler for state normalization
│   │   ├── scheduler_dataset.csv   # Training dataset
│   │   └── requirements.txt
│   │
│   ├── Edge-node/                  # CIFAR-10 Edge Inference Application
│   │   ├── server.py               # FastAPI inference server
│   │   ├── cnn_model_converted.h5  # CIFAR-10 trained CNN model
│   │   ├── edge-deployments.yaml   # K8s Deployment + Service (9 nodes)
│   │   ├── Dockerfile
│   │   ├── requirements.txt
│   │   └── steps.md
│   │
│   └── MILP_Selection/             # MILP-based Task Offloading Gateway
│       ├── milp_core.py            # Latency model + Pyomo MILP solver
│       ├── milp_gateway.py         # FastAPI gateway + routing logic
│       └── throughput_server.py    # Live UE bandwidth from ogstun interface
│
└── Setup/                          # Complete setup documentation
    ├── img/                        # Architecture diagrams
    ├── 1. 5g-Setup-Guide.md
    ├── 2. Microk8s_edge_deployment.md
    ├── 3. scheduler-deployment.md
    ├── 4. milp-gateway-setup.md
    ├── 5. DevStack-Single Node Setup Guide.md
    ├── 5.1 VM launch guide.md
    ├── 6. cloud-storage-vm-setup.md
    └── 7. cloud-backup-docker-setup.md
```

---

## System Layers

### UE Layer
Mobile devices connected to the private 5G SA network via a USRP B210 SDR and srsRAN gNB. Devices send image classification requests to the Task Offloading Scheduler over the 5G data plane.

### RAN Layer
- **srsRAN Project** gNB running on the localhost machine
- **USRP B210** as the software-defined radio front-end
- Band n78 · ARFCN 632628 · 20 MHz · 30 kHz SCS · PLMN 99970

### 5G Core Layer
- **Open5GS** running on a dedicated Ubuntu VM (`172.16.23.129`)
- **UPF** handles GTP-U tunnelling; `ogstun` interface carries UE traffic
- **Throughput Server** monitors `ogstun` live and exposes real-time UE bandwidth at `:7000/latest`

### Edge Layer (MicroK8s · `172.16.23.129`)
- **9 Edge Nodes** (`edge-node1` to `edge-node9`) — CIFAR-10 CNN inference pods, NodePorts `30081–30090`
- **AI Scheduler** (DQN) — NodePort `30084` — routes requests to the least-loaded edge pod
- **MILP Gateway** — Port `6001` — runs MILP optimization per request using real-time metrics

### Cloud Layer (OpenStack DevStack)
- **Cloud Backup Node** — Docker container (`amogh0709/fastapi-application:v5`) on port `8000` — handles requests when all edge nodes exceed thresholds
- **Cloud Storage VM** — Node.js Express server (`172.16.23.128:8080`) — receives and persists processed results uploaded by edge/cloud nodes

---

## Network Configuration

| Component | IP Address | Port(s) |
|-----------|-----------|---------|
| gNB (srsRAN) | `172.16.23.1` | — |
| Open5GS Core VM | `172.16.23.129` | — |
| Edge Nodes (K8s) | `172.16.23.129` | `30081–30083, 30085–30090` |
| AI Scheduler | `172.16.23.129` | `30084` |
| MILP Gateway | `172.16.23.129` | `6001` |
| Throughput Server | `172.16.23.129` | `7000` |
| Cloud Backup | `172.16.23.100` | `8000` |
| Cloud Storage VM | `172.16.23.128` | `8080` |
| UE IP Pool | `10.45.0.0/16` | — |

---

## Setup Guide

Follow the numbered guides inside `Setup/` in order:

| Step | Guide | Description |
|------|-------|-------------|
| 1 | `1. 5g-Setup-Guide.md` | srsRAN gNB + Open5GS Core + SIM programming |
| 2 | `2. Microk8s_edge_deployment.md` | MicroK8s install + 9 edge node deployment |
| 3 | `3. scheduler-deployment.md` | AI Scheduler (DQN) K8s deployment |
| 4 | `4. milp-gateway-setup.md` | MILP Gateway + Throughput Server |
| 5 | `5. DevStack-Single Node Setup Guide.md` | OpenStack DevStack single-node install |
| 5.1 | `5.1 VM launch guide.md` | Launch Ubuntu VM on OpenStack |
| 6 | `6. cloud-storage-vm-setup.md` | Node.js Cloud Storage server setup |
| 7 | `7. cloud-backup-docker-setup.md` | Docker-based Cloud Backup inference node |

---

## Quick Test

Once all components are running, send a test inference request from the 5G network:

```bash
# Via AI Scheduler (DQN):
curl -X POST http://10.45.0.1:30084/predict \
  -F "file=@image.jpg"

# Via MILP Gateway:
curl -X POST http://172.16.23.129:6001/milp_predict \
  -F "file=@image.jpg"
```

---

## Tech Stack

| Category | Technology |
|----------|-----------|
| 5G RAN | srsRAN Project, USRP B210 |
| 5G Core | Open5GS, UPF |
| Edge Orchestration | MicroK8s (Kubernetes) |
| Cloud Infrastructure | OpenStack DevStack |
| ML Scheduler | PyTorch DQN, scikit-learn |
| MILP Solver | Pyomo + HiGHS / CBC |
| Inference Server | FastAPI, Uvicorn |
| Cloud Storage | Node.js, Express, Multer |
| Containerisation | Docker |
| Model | CIFAR-10 CNN (TensorFlow / Keras) |


---


## Images

<!-- Add project images and screenshots here -->

| | |
|---|---|
| System Architecture | *(add image)* |
| Horizon Dashboard | *(add image)* |
| MicroK8s Pod Status | *(add image)* |
| MILP Decision Output | *(add image)* |
| Throughput Results | *(add image)* |

---
## Authors

**Amogh Talwar**
B.E. Computer Science and Engineering
KLE Technological University, Hubballi

> If you use, reference, or build upon this work, you **must** credit the original authors and link back to this repository. See the [License](#license) section below.

---


## License

This project is licensed under the **Creative Commons Attribution-NonCommercial 4.0 International (CC BY-NC 4.0)** license.

**You are free to:**
- Share — copy and redistribute the material in any medium or format
- Adapt — remix, transform, and build upon the material

**Under the following terms:**
- **Attribution** — You must give appropriate credit, provide a link to this repository, and indicate if changes were made. Credit must appear visibly in your README, report, or codebase.



[![License: CC BY-NC 4.0](https://img.shields.io/badge/License-CC%20BY--NC%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by-nc/4.0/)



<div align="center">
<sub>© 2025 Amogh Talwar · KLE Technological University · All rights reserved under CC BY-NC 4.0</sub>
</div>
