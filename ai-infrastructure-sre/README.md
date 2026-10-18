# AI Infrastructure Engineering & SRE

A **100-chapter, 12-part** production-focused curriculum for engineers supporting GPU infrastructure, Kubernetes, cloud compute, CUDA/NCCL, high-performance networking, distributed training, storage, reliability and incident response.

**Published:** 13/100. **Lab execution:** pending. **Technical/security review:** pending. Published documentation is not equivalent to tested infrastructure.

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

**Next:** Chapter 014 — GPU Firmware, BIOS and Host Maintenance.
