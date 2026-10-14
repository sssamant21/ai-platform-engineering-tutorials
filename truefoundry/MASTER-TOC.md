# TrueFoundry — Master Table of Contents

## Part 1 — Foundations & Architecture
- 1.1 — What Is TrueFoundry?
- 1.2 — TrueFoundry Architecture: Control Plane, Compute Plane & `tfy-agent`
- 1.3 — TrueFoundry Platform Components & Terminology
- 1.4 — TrueFoundry and Kubernetes: How They Work Together
- 1.5 — Workspaces, Environments & Deployment Concepts
- 1.6 — SRE Ownership, Failure Domains & Support Model

## Part 2 — Kubernetes Integration
2.1 Cluster Connection; 2.2 `tfy-agent`; 2.3 Namespaces & Placement; 2.4 RBAC; 2.5 ConfigMaps/Secrets; 2.6 Services/Ingress/DNS/TLS; 2.7 Storage.

## Part 3 — Compute & Scheduling
CPU/memory, requests/limits/QoS, affinity, taints/tolerations, dedicated node pools, scheduling troubleshooting.

## Part 4 — GPU Infrastructure
- [4.1 — GPU Architecture and NVIDIA GPU Fundamentals](part-04-gpu-infrastructure/01-gpu-architecture-nvidia-fundamentals/01-gpu-architecture-nvidia-fundamentals.md) — [Hands-on Lab](part-04-gpu-infrastructure/labs/01-gpu-architecture-nvidia-fundamentals-check.md) (published)
- [4.2 — GPU Nodes and Kubernetes GPU Scheduling](part-04-gpu-infrastructure/02-gpu-nodes-kubernetes-scheduling/02-gpu-nodes-kubernetes-scheduling.md) — [Hands-on Lab](part-04-gpu-infrastructure/labs/02-gpu-nodes-kubernetes-scheduling-check.md) (published)
- [4.3 — NVIDIA Drivers, Device Plugin and GPU Operator](part-04-gpu-infrastructure/03-nvidia-drivers-device-plugin-gpu-operator/03-nvidia-drivers-device-plugin-gpu-operator.md) — [Hands-on Lab](part-04-gpu-infrastructure/labs/03-nvidia-drivers-device-plugin-gpu-operator-check.md) (published)
- [4.4 — CUDA, Container Runtime and Compatibility](part-04-gpu-infrastructure/04-cuda-container-runtime-compatibility/04-cuda-container-runtime-compatibility.md) — [Hands-on Lab](part-04-gpu-infrastructure/labs/04-cuda-container-runtime-compatibility-check.md) (published)
- 4.5 — Dedicated GPU Node Pools and Resource Isolation (planned)
- 4.6 — GPU Validation, Monitoring and Troubleshooting (planned)
- 4.7 — Production GPU Infrastructure Recovery Lab (planned)
- [Part 4 master layout](part-04-gpu-infrastructure/PART-04-LAYOUT.md)

## Part 5 — Application Deployment
First service, images/registries, configuration/secrets, resources, probes, endpoints, validation, rollout troubleshooting.

## Part 6 — Model Serving
Model-serving fundamentals, architecture, CPU/GPU serving, storage/loading, endpoints, health/readiness, troubleshooting.

## Part 7 — vLLM & LLM Serving
vLLM fundamentals/architecture, TrueFoundry deployment, GPU/CUDA, loading, tensor parallelism, KV cache, tuning, device detection, CUDA/PyTorch troubleshooting.

## Part 8 — Scaling & Performance
Scaling, HPA, GPU autoscaling, concurrency, latency/throughput, GPU utilization, performance troubleshooting.

## Part 9 — Networking
Architecture, Services, Ingress, DNS, TLS, internal/external endpoints, troubleshooting.

## Part 10 — Security
Architecture, authentication/authorization, RBAC, ServiceAccounts, secrets, cloud IAM, isolation, production checklist.

## Part 11 — Observability
Architecture, logs, Kubernetes events, platform/GPU/model metrics, latency/throughput/errors, dashboards/alerts.

## Part 12 — Production Operations
Readiness, capacity, GPU capacity, HA, upgrades, FinOps, operational runbooks.

## Part 13 — Troubleshooting
Methodology, `tfy-agent`, deployment, Pending, CrashLoopBackOff, ImagePullBackOff, RBAC, networking, GPU, NVIDIA/CUDA, vLLM, inference performance, GPU OOM, incidents.

## Part 14 — SRE Runbooks
Platform health, `tfy-agent`, GPU health, model deployment, vLLM health, incidents, post-incident validation.

## Part 15 — End-to-End Production Project
Architecture, Kubernetes, GPU, TrueFoundry, vLLM deployment, networking/security, observability, load testing, failure/recovery, readiness, incident simulation, acceptance lab.
