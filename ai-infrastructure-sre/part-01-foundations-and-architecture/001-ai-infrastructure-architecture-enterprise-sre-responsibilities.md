# Chapter 001 — AI Infrastructure Architecture and Enterprise SRE Responsibilities

**Part 01 — Foundations and Architecture · Chapter 001 of 100**  
**Publication:** Complete. **Lab execution, real infrastructure validation and independent review:** Pending.

## Learning outcomes
Explain how AI compute infrastructure differs from general application infrastructure; map GPU hardware, CUDA, Kubernetes, model-serving/training workloads, networking and storage; define support ownership and SLOs; perform safe discovery; build an actionable incident response and capacity plan.

## 1. What AI infrastructure SRE owns
AI infrastructure SRE maintains the **platform underneath** AI workloads, not merely the application model. Typical responsibilities include GPU node readiness, driver/device-plugin compatibility, CUDA and NCCL failures, Kubernetes scheduling, training queues, checkpoint durability, RDMA/network performance, GPU quotas, cloud capacity, identity, security, cost, observability and disaster recovery.

| Layer | Primary components | SRE failure mode |
|---|---|---|
| Physical/virtual compute | NVIDIA GPU, CPU, RAM, PCIe, NUMA | Xid/ECC, PCIe fault, insufficient VRAM |
| Host | Linux, drivers, firmware, container runtime | kernel/driver mismatch, device absent |
| Accelerator runtime | CUDA, NVML, NCCL, GPU operator | CUDA init failure, collective hang |
| Orchestration | Kubernetes, Slurm, Ray | Pending pods, failed worker placement |
| Network | Ethernet, EFA, InfiniBand, RDMA | congestion, low all-reduce throughput |
| Storage | object, block, Lustre, checkpoints | dataset stalls, slow restore |
| Workloads | inference, fine-tuning, training | OOM, slow TTFT, training restarts |
| Platform services | IAM, registry, monitoring, GitOps | unauthorized access, drift, blind spots |

## 2. Reference architecture
```text
                  AI users / teams / CI pipelines
                                |
                 identity + quotas + policy
                                |
                  workload submission API
                         /             \
                training jobs       inference services
                    |                       |
                 Slurm/Ray              KServe/Ray
                         \             /
                       Kubernetes GPU clusters
                      /         |          \
                GPU node A   GPU node B   GPU node C
                driver + CUDA + device plugin
                      \         |          /
                    NCCL / EFA / RDMA fabric
                                |
                parallel filesystem / object storage
                                |
                Prometheus + DCGM + OTel + audit
                                |
                     incident + capacity SRE
```
Distinguish **control plane** (schedulers/operators/CI) from **data plane** (GPU execution, storage I/O, networking). A control-plane outage may not stop existing jobs immediately, but may prevent replacement or recovery.

## 3. Workload profiles and capacity
**Inference** emphasizes latency, token mix, KV cache, availability and streaming completion. **Fine-tuning** emphasizes VRAM, checkpoint durability and batch throughput. **Distributed training** emphasizes GPU interconnect, NCCL synchronization, dataset throughput and recovery time.

Capacity must use workload-specific safe throughput, not theoretical GPU TFLOPS. Measure memory headroom, GPU utilization, power/thermal throttling, interconnect bandwidth, storage throughput, queue age and failure-domain placement. For N-1 zone resilience:
```text
safe_N_minus_1 = sum(zone_safe_capacity) - max(zone_safe_capacity)
headroom = safe_N_minus_1 - forecast_peak_demand
```
All quantities must share the same unit and workload profile.

## 4. Reliability and operational ownership
Define SLOs for **GPU allocation success**, workload scheduling delay, device health, training job success, checkpoint restore success, inference completion and platform recovery. Map escalation across hardware/cloud provider, platform SRE, ML platform, model owners, network/storage, security and DBRE. Use a RACI matrix and an on-call service catalog. Require versioned compatibility matrices for GPU architecture, driver, CUDA, NCCL, kernel, container runtime and GPU operator.

## 5. Hands-on Lab A — Read-only infrastructure inventory
Use a dedicated test workstation or approved staging context. These commands inspect state and do not modify workloads:
```bash
uname -a
lscpu
free -h
df -h
command -v nvidia-smi && nvidia-smi -L || true
command -v nvidia-smi && nvidia-smi --query-gpu=index,name,memory.total,driver_version,temperature.gpu --format=csv || true
kubectl config current-context
kubectl get nodes -o wide
kubectl get nodes -L topology.kubernetes.io/zone
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp
```
`nvidia-smi` may be absent on CPU-only machines. `kubectl` requires an authorized kubeconfig. GPU metrics require NVIDIA driver and device visibility. Do not paste credentials, internal IPs or sensitive pod metadata into public issues.

## 6. Hands-on Lab B — Kubernetes GPU discovery
```bash
kubectl describe node YOUR_APPROVED_GPU_NODE
kubectl get daemonsets -A
kubectl get resourcequota -A
kubectl get priorityclass
```
Verify allocatable `nvidia.com/gpu` (or vendor-specific resource), GPU node labels, taints, device-plugin DaemonSet, topology, node readiness, requested/allocated GPU and quota. A GPU installed physically but absent from allocatable resources suggests driver, device-plugin, runtime or operator configuration problems. Never repair by blindly restarting a production node.

## 7. Production incident simulation
**Scenario:** A new GPU node joins the cluster but advertises zero allocatable GPUs. In staging, compare node capacity, driver status, device-plugin logs, operator state, container runtime and host kernel. Identify failure domain, collect a UTC timeline, establish a safe remediation plan and verify a synthetic GPU pod schedules without bypassing Pod Security or IAM.

**Expected escalation:** infrastructure/platform SRE first; hardware/cloud support for GPU/PCIe faults; ML platform owner for model runtime incompatibility; security for access-policy violations.

## 8. Troubleshooting matrix
| Symptom | Likely issue | Evidence |
|---|---|---|
| GPU missing in Kubernetes | device plugin/driver | allocatable resources, DaemonSet |
| CUDA initialization error | driver/runtime compatibility | driver version, container CUDA |
| GPU OOM | VRAM/KV cache/batch size | process memory, context, logs |
| NCCL timeout | networking or worker failure | NCCL diagnostics, network metrics |
| Low GPU utilization | input pipeline, CPU, storage | GPU, I/O and queue telemetry |
| Pending GPU pod | quota, taints, topology or scarcity | scheduler events, node allocatable |
| Checkpoint stalls | storage latency/throughput | filesystem and object metrics |
| Rising GPU costs | low utilization or idle reservation | allocation, GPU-hours, spend |

## 9. Runbooks
**RB-001-A — GPU not detected:** confirm context/host, node health, device inventory, driver, device plugin, operator and compatible runtime; isolate faulty node only with approved change.

**RB-001-B — Capacity incident:** identify queued workloads, per-zone GPU availability, quotas, fragmentation and safe admission; escalate shortage without uncontrolled retry.

**RB-001-C — Training slowdown:** correlate GPU, NCCL, network, dataset reads and checkpoint writes; preserve checkpoints and evidence.

**RB-001-D — Change/rollback:** verify tested driver/runtime matrix, canary staging node, GPU health, workload success, maintenance window and recovery plan.

## 10. Architecture review and 100-point assessment
| Area | Points |
|---|---:|
| Infrastructure component and dependency map | 20 |
| GPU/Kubernetes/Linux inventory | 20 |
| SLO, capacity and failure-domain analysis | 20 |
| Incident triage and safe recovery | 20 |
| Runbooks, ownership and evidence | 20 |
| **Total** | **100** |

Educational pass >=90/100 plus no critical identity, safety or data-integrity violation. Read-only discovery and offline simulations do not certify production readiness.

## 11. Executable offline lab
[Chapter 001 — AI Infrastructure Architecture Acceptance Lab](../labs/001-ai-infrastructure-architecture-check.md) contains a Python 3.11+ N-1 capacity and security gate evaluator with pytest.

## References
- [NVIDIA Data Center GPU Documentation](https://docs.nvidia.com/datacenter/)
- [Kubernetes Scheduling](https://kubernetes.io/docs/concepts/scheduling-eviction/)
- [Kubernetes Device Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)
- [Google SRE Workbook](https://sre.google/workbook/)

**Next:** Chapter 002 — AI Workload Taxonomy: Training, Fine-Tuning and Inference.
