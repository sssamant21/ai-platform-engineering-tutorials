# Part 4 — GPU Infrastructure — Master Layout

## Objective
Teach production GPU architecture, Kubernetes GPU placement, NVIDIA driver/device-plugin integration, CUDA compatibility, node-pool isolation, observability and recovery for TrueFoundry-managed workloads.

## Chapters
| # | Chapter | Canonical | Lab |
|---|---|---|---|
| 4.1 | GPU Architecture and NVIDIA GPU Fundamentals | [Tutorial](01-gpu-architecture-nvidia-fundamentals/01-gpu-architecture-nvidia-fundamentals.md) | [Lab](labs/01-gpu-architecture-nvidia-fundamentals-check.md) |
| 4.2 | GPU Nodes and Kubernetes GPU Scheduling | [Tutorial](02-gpu-nodes-kubernetes-scheduling/02-gpu-nodes-kubernetes-scheduling.md) | [Lab](labs/02-gpu-nodes-kubernetes-scheduling-check.md) |
| 4.3 | NVIDIA Drivers, Device Plugin and GPU Operator | [Tutorial](03-nvidia-drivers-device-plugin-gpu-operator/03-nvidia-drivers-device-plugin-gpu-operator.md) | [Lab](labs/03-nvidia-drivers-device-plugin-gpu-operator-check.md) |
| 4.4 | CUDA, Container Runtime and Compatibility | [Tutorial](04-cuda-container-runtime-compatibility/04-cuda-container-runtime-compatibility.md) | [Lab](labs/04-cuda-container-runtime-compatibility-check.md) |
| 4.5 | Dedicated GPU Node Pools and Resource Isolation | [Tutorial](05-dedicated-gpu-node-pools-isolation/05-dedicated-gpu-node-pools-isolation.md) | [Lab](labs/05-dedicated-gpu-node-pools-isolation-check.md) |
| 4.6 | GPU Validation, Monitoring and Troubleshooting | Planned | Planned |
| 4.7 | Production GPU Infrastructure Recovery Lab | Planned | Planned |

## Standard
Canonical path: `NN-topic/NN-topic.md`. Separate lab: `labs/NN-topic-check.md`. Production checks default to **[SAFE-READ]**. Mutating experiments require disposable resources and cleanup. Record source-of-truth ownership, failure domain, first failed transition, blast radius, rollback and acceptance.

## Scope boundary
Part 4 establishes GPU infrastructure. Model deployment is Part 6; vLLM-specific serving and tuning are Part 7; broad scaling/performance is Part 8; platform-wide observability is Part 11.

## Review gate
Draft → Technical/Vendor Validation → Production/SRE Review → Revised Final → Canonical → Hands-On Lab → Repository Validation. Publication and link verification are tracked separately from vendor validation; a published file is not evidence of completed external review.
