# AI Infrastructure Engineering & SRE

A **100-chapter, 12-part** production-focused curriculum for engineers supporting GPU infrastructure, Kubernetes, cloud compute, CUDA/NCCL, high-performance networking, distributed training, storage, reliability and incident response.

**Published:** 40/100. **Lab execution:** pending. **Technical/security review:** pending. Published documentation is not equivalent to tested infrastructure.

[Master TOC](MASTER-TOC.md) · [Status Tracker](STATUS-TRACKER.md)

## Operating principles
- Deep theory, architecture, executable labs, pytest, troubleshooting, SRE/DBRE scenarios and runbooks.
- Safe read-only discovery first; destructive changes only in approved disposable staging.
- Separate observed measurements from synthetic simulation; never claim tests passed unless executed.
- Use AWS/EKS and Azure/AKS examples; include vendor-neutral Linux and Kubernetes paths.
- Update indexes and publish each chapter independently.

## Start here
- [Chapter 001 — AI Infrastructure Architecture and Enterprise SRE Responsibilities](part-01-foundations-and-architecture/001-ai-infrastructure-architecture-enterprise-sre-responsibilities.md)
- [Chapter 001 Lab](labs/001-ai-infrastructure-architecture-check.md)

- [Chapter 002 — AI Workload Taxonomy: Training, Fine-Tuning and Inference](part-01-foundations-and-architecture/002-ai-workload-taxonomy-training-fine-tuning-inference.md) — [Lab](labs/002-ai-workload-taxonomy-check.md)

- [Chapter 003 — GPU Infrastructure Reference Architectures](part-01-foundations-and-architecture/003-gpu-infrastructure-reference-architectures.md) — [Lab](labs/003-gpu-infrastructure-reference-architectures-check.md)

- [Chapter 004 — AI Infrastructure Service Ownership and Support Tiers](part-01-foundations-and-architecture/004-ai-infrastructure-service-ownership-support-tiers.md) — [Lab](labs/004-ai-infrastructure-service-ownership-check.md)

- [Chapter 005 — Linux, Kubernetes and Cloud Dependency Mapping](part-01-foundations-and-architecture/005-linux-kubernetes-cloud-dependency-mapping.md) — [Lab](labs/005-linux-kubernetes-cloud-dependency-mapping-check.md)

- [Chapter 006 — AI Infrastructure Reliability, SLOs and Error Budgets](part-01-foundations-and-architecture/006-ai-infrastructure-reliability-slos-error-budgets.md) — [Lab](labs/006-ai-infrastructure-reliability-slos-error-budgets-check.md)

- [Chapter 007 — Infrastructure Security and Trust Boundaries](part-01-foundations-and-architecture/007-infrastructure-security-trust-boundaries.md) — [Lab](labs/007-infrastructure-security-trust-boundaries-check.md)

- [Chapter 008 — Foundations Architecture Review Lab](part-01-foundations-and-architecture/008-foundations-architecture-review-lab.md) — [Lab](labs/008-foundations-architecture-review-check.md)

**Part 01 — Foundations and Architecture: 8/8 published (content complete; runtime validation pending).**

- [Chapter 009 — GPU Server Hardware, PCIe and NUMA](part-02-linux-servers-and-gpu-hardware/009-gpu-server-hardware-pcie-numa.md) — [Lab](labs/009-gpu-server-hardware-pcie-numa-check.md)

- [Chapter 010 — Linux GPU Host Provisioning and Baselines](part-02-linux-servers-and-gpu-hardware/010-linux-gpu-host-provisioning-baselines.md) — [Lab](labs/010-linux-gpu-host-provisioning-baselines-check.md)

- [Chapter 011 — NVIDIA Driver Lifecycle and Compatibility](part-02-linux-servers-and-gpu-hardware/011-nvidia-driver-lifecycle-compatibility.md) — [Lab](labs/011-nvidia-driver-lifecycle-compatibility-check.md)

- [Chapter 012 — GPU Discovery with nvidia-smi and NVML](part-02-linux-servers-and-gpu-hardware/012-gpu-discovery-nvidia-smi-nvml.md) — [Lab](labs/012-gpu-discovery-nvidia-smi-nvml-check.md)

- [Chapter 013 — GPU Hardware Health, ECC and Xid Errors](part-02-linux-servers-and-gpu-hardware/013-gpu-hardware-health-ecc-xid-errors.md) — [Lab](labs/013-gpu-hardware-health-ecc-xid-check.md)

- [Chapter 014 — GPU Firmware, BIOS and Host Maintenance](part-02-linux-servers-and-gpu-hardware/014-gpu-firmware-bios-host-maintenance.md) — [Lab](labs/014-gpu-firmware-bios-host-maintenance-check.md)

- [Chapter 015 — GPU Node Replacement and Recovery](part-02-linux-servers-and-gpu-hardware/015-gpu-node-replacement-recovery.md) — [Lab](labs/015-gpu-node-replacement-recovery-check.md)

- [Chapter 016 — GPU Hardware Operations Capstone](part-02-linux-servers-and-gpu-hardware/016-gpu-hardware-operations-capstone.md) — [Lab](labs/016-gpu-hardware-operations-capstone-check.md)

**Part 02:** All 8 chapters published; runtime validation pending.

- [Chapter 017 — CUDA Runtime and Driver Compatibility](part-03-cuda-gpu-memory-and-communication/017-cuda-runtime-driver-compatibility.md) — [Lab](labs/017-cuda-runtime-driver-compatibility-check.md)

**Part 03:** 1/9 chapters published.

- [Chapter 018 — GPU Memory Architecture and OOM](part-03-cuda-gpu-memory-and-communication/018-gpu-memory-architecture-oom.md) — [Lab](labs/018-gpu-memory-architecture-oom-check.md)

**Part 03:** 2/9 chapters published.

- [Chapter 019 — CUDA Processes, Streams and Contexts](part-03-cuda-gpu-memory-and-communication/019-cuda-processes-streams-contexts.md) — [Lab](labs/019-cuda-processes-streams-contexts-check.md)

**Part 03:** 3/9 chapters published.

- [Chapter 020 — MIG Partitioning and Isolation](part-03-cuda-gpu-memory-and-communication/020-mig-partitioning-isolation.md) — [Lab](labs/020-mig-partitioning-isolation-check.md)

**Part 03:** 4/9 chapters published.

- [Chapter 021 — NVLink and NVSwitch Topologies](part-03-cuda-gpu-memory-and-communication/021-nvlink-nvswitch-topologies.md) — [Lab](labs/021-nvlink-nvswitch-topologies-check.md)

**Part 03:** 5/9 chapters published.

- [Chapter 022 — NCCL Fundamentals and Collective Operations](part-03-cuda-gpu-memory-and-communication/022-nccl-fundamentals-collectives.md) — [Lab](labs/022-nccl-fundamentals-collectives-check.md)

**Part 03:** 6/9 chapters published.

- [Chapter 023 — NCCL Diagnostics and Hang Recovery](part-03-cuda-gpu-memory-and-communication/023-nccl-diagnostics-hang-recovery.md) — [Lab](labs/023-nccl-diagnostics-hang-recovery-check.md)

**Part 03:** 7/9 chapters published.

- [Chapter 024 — GPU Profiling and Bottleneck Analysis](part-03-cuda-gpu-memory-and-communication/024-gpu-profiling-bottleneck-analysis.md) — [Lab](labs/024-gpu-profiling-bottleneck-check.md)

**Part 03:** 8/9 chapters published.

- [Chapter 025 — CUDA and NCCL Troubleshooting Capstone](part-03-cuda-gpu-memory-and-communication/025-cuda-nccl-troubleshooting-capstone.md) — [Lab](labs/025-cuda-nccl-troubleshooting-capstone-check.md)

**Part 03 complete:** 9/9 chapters published (live validation pending).

- [Chapter 026 — GPU Device Plugin and Resource Discovery](part-04-kubernetes-gpu-administration/026-gpu-device-plugin-resource-discovery.md) — [Lab](labs/026-gpu-device-plugin-resource-discovery-check.md)

**Part 04:** 1/9 chapters published.

- [Chapter 027 — GPU Operator Architecture and Lifecycle](part-04-kubernetes-gpu-administration/027-gpu-operator-architecture-lifecycle.md) — [Lab](labs/027-gpu-operator-lifecycle-check.md)

**Part 04:** 2/9 chapters published.

- [Chapter 028 — GPU Scheduling, Taints and Affinity](part-04-kubernetes-gpu-administration/028-gpu-scheduling-taints-affinity.md) — [Lab](labs/028-gpu-scheduling-taints-affinity-check.md)

**Part 04:** 3/9 chapters published.

- [Chapter 029 — GPU Resource Quotas and Multitenancy](part-04-kubernetes-gpu-administration/029-gpu-resource-quotas-multitenancy.md) — [Lab](labs/029-gpu-resource-quotas-multitenancy-check.md)

**Part 04:** 4/9 chapters published.

- [Chapter 030 — GPU Pod Security and Runtime Integration](part-04-kubernetes-gpu-administration/030-gpu-pod-security-runtime-integration.md) — [Lab](labs/030-gpu-pod-security-runtime-integration-check.md)

**Part 04:** 5/9 chapters published.

- [Chapter 031 — GPU Node Autoscaling and Karpenter](part-04-kubernetes-gpu-administration/031-gpu-node-autoscaling-karpenter.md) — [Lab](labs/031-gpu-node-autoscaling-karpenter-check.md)

**Part 04:** 6/9 chapters published.

- [Chapter 032 — GPU Workload Rollouts and Disruptions](part-04-kubernetes-gpu-administration/032-gpu-workload-rollouts-disruptions.md) — [Lab](labs/032-gpu-workload-rollouts-disruptions-check.md)

**Part 04:** 7/9 chapters published.

- [Chapter 033 — GPU Operator and Device Plugin Troubleshooting](part-04-kubernetes-gpu-administration/033-gpu-operator-device-plugin-troubleshooting.md) — [Lab](labs/033-gpu-operator-device-plugin-troubleshooting-check.md)

**Part 04:** 8/9 chapters published.

- [Chapter 034 — Kubernetes GPU Platform Capstone](part-04-kubernetes-gpu-administration/034-kubernetes-gpu-platform-capstone.md) — [Lab](labs/034-kubernetes-gpu-platform-capstone-check.md)

**Part 04:** 9/9 chapters published — publication complete; runtime and staging validation pending.

- [Chapter 035 — AWS GPU EC2 Architecture and Quotas](part-05-cloud-gpu-infrastructure/035-aws-gpu-ec2-architecture-quotas.md) — [Lab](labs/035-aws-gpu-ec2-architecture-quotas-check.md)

**Part 05:** 1/8 chapters published.

- [Chapter 036 — EKS GPU Managed Node Groups](part-05-cloud-gpu-infrastructure/036-eks-gpu-managed-node-groups.md) — [Lab](labs/036-eks-gpu-managed-node-groups-check.md)

**Part 05:** 2/8 chapters published.

- [Chapter 037 — AWS EFA and GPU Networking](part-05-cloud-gpu-infrastructure/037-aws-efa-gpu-networking.md) — [Lab](labs/037-aws-efa-gpu-networking-check.md)

**Part 05:** 3/8 chapters published.

- [Chapter 038 — Azure GPU VM Architecture and Quotas](part-05-cloud-gpu-infrastructure/038-azure-gpu-vm-architecture-quotas.md) — [Lab](labs/038-azure-gpu-vm-architecture-quotas-check.md)

**Part 05:** 4/8 chapters published.

- [Chapter 039 — AKS GPU Node Pools and Drivers](part-05-cloud-gpu-infrastructure/039-aks-gpu-node-pools-drivers.md) — [Lab](labs/039-aks-gpu-node-pools-drivers-check.md)

**Part 05:** 5/8 chapters published.

- [Chapter 040 — Cloud GPU Identity, Storage and Security](part-05-cloud-gpu-infrastructure/040-cloud-gpu-identity-storage-security.md) — [Lab](labs/040-cloud-gpu-identity-storage-security-check.md)

**Part 05:** 6/8 chapters published.

**Next:** Chapter 041 — Multi-Cloud GPU Capacity and Cost.
