# AI Infrastructure Engineering & SRE

A **100-chapter, 12-part** production-focused curriculum for engineers supporting GPU infrastructure, Kubernetes, cloud compute, CUDA/NCCL, high-performance networking, distributed training, storage, reliability and incident response.

**Published:** 19/100. **Lab execution:** pending. **Technical/security review:** pending. Published documentation is not equivalent to tested infrastructure.

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

**Next:** Chapter 020 — MIG Partitioning and Isolation.
