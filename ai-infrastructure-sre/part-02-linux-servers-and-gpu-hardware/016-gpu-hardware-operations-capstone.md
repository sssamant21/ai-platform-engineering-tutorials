# Chapter 016 — GPU Hardware Operations Capstone

**Part 02 — Linux, Servers and GPU Hardware · Final capstone · Chapter 016 of 100**

**Publication:** Complete. **Lab execution, staging game day, independent security review and production acceptance:** Pending.

## Mission and expected outcomes

Act as the on-call GPU Infrastructure SRE responsible for a mixed Kubernetes GPU fleet during a hardware degradation and scheduled node refresh. Prove that you can inventory hardware, validate firmware/driver compatibility, detect Xid/ECC anomalies, protect training checkpoints and inference capacity, replace a GPU node, and restore a service within its approved RTO/RPO and SLO. This capstone integrates Chapters 009–015 rather than repeating isolated command demonstrations.

The acceptance process must be **evidence-driven**: a green Kubernetes node is not enough. The node must enumerate the expected physical GPUs, expose usable devices to approved workloads, meet the application SLO, and produce current security and health telemetry.

## 1. Reference topology

```text
                         Incident command / change control
                                      |
              +-----------------------+-----------------------+
              |                                               |
        Kubernetes cluster                             Asset / CMDB
              |                                               |
       GPU node pool (AZ-A / AZ-B)                  GPU UUID + serial mapping
       +-----------------------+
       | Node 1: inference     |---> GPU driver / NVML / device plugin
       | Node 2: inference     |---> DCGM exporter / Xid / ECC
       | Node 3: training      |---> NIC / RDMA / NVMe / checkpoint
       | Node 4: spare/canary  |---> image/firmware compatibility gate
       +-----------------------+
              |
   Workload SLO + checkpoint/object storage + alert routing
```

The four nodes above are a **conceptual example**, not a claim about actual fleet capacity. Design for your real instance SKUs, zones, GPUs per node, MIG layout, quota, scheduling constraints, and failure domains.

## 2. End-to-end operating contract

| Control | Required evidence | Hard-stop condition |
|---|---|---|
| Fleet ownership | node, pool, cloud/bare-metal, owner, ticket | no accountable owner |
| Hardware identity | GPU UUID, PCI bus, SKU, expected count | mismatch or unapproved replacement |
| Firmware baseline | BIOS/BMC/GPU/NIC/NVMe, vendor compatibility | unsupported combination |
| Driver baseline | OS/kernel, NVIDIA driver, runtime/toolkit, operator owner | unknown or conflicting owner |
| Hardware health | Xid/ECC, PCIe/NVLink, thermals, power | critical fault not contained |
| Capacity | allocatable GPUs, zone spread, N-1, cloud quota | insufficient safe headroom |
| Workload safety | PDB, checkpoint, local PV/NVMe, rollback | possible unapproved data loss |
| Security | signed image, Secure Boot, IAM/workload identity | unverified trust |
| Observability | DCGM/NVML freshness, kubelet and app SLO | blind rollout |
| Recovery | approved previous image or replacement pool, RTO/RPO proof | untested recovery |

**Rule:** no single successful command overrides a failed gate. Unsupported metrics must be recorded as unavailable, not coerced to zero.

## 3. Preflight: approved read-only inventory

```bash
date -u
uname -r
cat /etc/os-release
lspci -nn | grep -Ei 'nvidia|3d controller|ethernet|infiniband'
nvidia-smi -L
nvidia-smi --query-gpu=uuid,name,driver_version,pci.bus_id,memory.total --format=csv
nvidia-smi topo -m
nvidia-smi -q
journalctl -k -b --no-pager | grep -Ei 'NVRM|Xid|AER|pcie|nvidia' | tail -n 150
```

Use approved privileged read access for BMC/BIOS inventory where necessary. Redact GPU serials, host identities, processes and security-sensitive logs before external sharing.

```bash
kubectl config current-context
kubectl get nodes -o wide -L topology.kubernetes.io/zone
kubectl get pods -A -o wide
kubectl get pdb -A
kubectl get pv
kubectl get pvc -A
kubectl get daemonsets -A
kubectl get events -A --sort-by=.lastTimestamp
kubectl describe node APPROVED_GPU_NODE
```

Compare GPU UUIDs with approved asset inventory, `nvidia.com/gpu` or MIG-specific allocatable resources, device-plugin state, node readiness and expected GPU count. **Do not** run destructive Kubernetes commands as part of this read-only preflight.

## 4. Capstone lab sequence

### Lab 1 — Hardware topology and NUMA
Inventory PCIe hierarchy, NUMA nodes, GPU/NIC affinity and interconnect. Explain how topology changes can affect CUDA copies, RDMA and NCCL. Record the baseline as a versioned artifact.

### Lab 2 — Host and driver readiness
Verify immutable image provenance, kernel/driver compatibility, module load, Secure Boot, GPU enumeration, NVIDIA Container Toolkit and exactly one driver installation owner. Prove a CUDA smoke workload can initialize each expected GPU on an **approved staging node**.

### Lab 3 — GPU fault telemetry
Capture ECC support and counters, Xid kernel events, PCIe AER, temperature, power and DCGM exporter freshness. Classify event severity using device- and vendor-specific guidance; do not assume every Xid is a failed card.

### Lab 4 — Firmware change review
Prepare a signed compatibility matrix for GPU SKU, BIOS/BMC, GPU firmware, NIC, kernel, driver and image. Review a proposed upgrade and document recovery constraints. Do **not** flash hardware as part of the offline lab.

### Lab 5 — Capacity and checkpoint protection
Compute whether removing one GPU node leaves sufficient usable capacity for inference replicas and distributed training. Check cloud quota, zone spread, PDBs, local PVs and durable checkpoint restore evidence.

### Lab 6 — Synthetic failure injection
Feed the companion validator scenarios for a missing GPU, uncorrectable ECC, stale telemetry, incompatible driver, unsigned image, failed checkpoint and insufficient N-1 capacity. Every scenario must fail closed.

### Lab 7 — Approved staging replacement
Only with separate authorization, rehearse workload-safe eviction, provider-supported EKS/AKS or bare-metal replacement, image bootstrap and asset/UUID reconciliation. Prefer managed node-group/pool lifecycle rather than ad-hoc VM deletion.

### Lab 8 — Post-replacement acceptance
Verify device-plugin GPU allocation, container CUDA smoke, NCCL where relevant, checkpoint restore, application SLO, DCGM metrics, Xid/ECC status, identity and rollback readiness.

### Lab 9 — Incident review
Publish incident timeline, blast radius, fault hypothesis, evidence links, vendor escalation, corrective actions, change approvals and remaining risks.

## 5. Production incident game-day cards

| Scenario | Injected observation | Expected SRE response |
|---|---|---|
| 1. GPU disappears | expected 8, observed 7 | halt admission, check PCIe/driver, isolate |
| 2. Uncorrectable ECC | new uncorrectable error | preserve evidence, checkpoint-safe containment |
| 3. Xid storm | repeated vendor-classified critical Xid | stop rollout, correlate driver and hardware |
| 4. Driver mismatch | host NVML fails after kernel change | recover pinned image; do not reinstall blindly |
| 5. Firmware topology drift | GPU/NIC NUMA affinity changes | compare baselines, abort canary |
| 6. Node Ready, zero GPU allocatable | plugin failure | investigate device plugin/operator/runtime |
| 7. Training checkpoint missing | local-only state | block node replacement, escalate data risk |
| 8. Inference capacity shortfall | one-node removal breaches SLO | stop drain, secure capacity |
| 9. Telemetry outage | DCGM scrape stale | stop unsafe rollout; restore monitoring |
| 10. Replacement identity drift | unexpected IAM/asset mapping | quarantine, correct and audit |

**Safety:** simulate faults using fixtures, alert streams and staging workloads. Do not inject real ECC faults, reset active GPUs, corrupt firmware, or delete production nodes.

## 6. Operational runbooks

**RB-016-A — Declare and triage:** create incident ticket, preserve UTC timestamps, affected GPU UUID/node/pool, workload impact and current change history.

**RB-016-B — Capture hardware evidence:** collect sanitized PCIe/NUMA, firmware, driver, Xid/ECC, NVLink and GPU topology. Distinguish unsupported fields from healthy measurements.

**RB-016-C — Protect workloads:** verify PDBs, training checkpoint restore, persistent/local storage, replica health, quota and N-1 capacity.

**RB-016-D — Contain a failed GPU:** isolate only with owner approval, coordinate job recovery and vendor diagnosis; avoid repeated resets.

**RB-016-E — Validate a driver/firmware change:** require vendor compatibility evidence, signed artifacts, canary, telemetry and a tested recovery path.

**RB-016-F — Replace a node:** use supported cloud/bare-metal lifecycle, reconcile GPU UUID, cloud identity, image, device plugin and application placement.

**RB-016-G — Rollback and failback:** halt admission, restore approved image/pool, confirm checkpoint, GPU health, workload SLO and security.

**RB-016-H — Close the incident:** document RTO/RPO, change evidence, operational metrics, RCA/COA, asset inventory and independent signoff.

## 7. Troubleshooting decision matrix

| Symptom | First evidence | Likely boundary | Escalation |
|---|---|---|---|
| missing PCIe GPU | lspci/BMC | hardware/firmware | vendor/cloud |
| GPU present, NVML fails | NVRM/module | driver/kernel | Linux GPU SRE |
| host healthy, pod cannot use GPU | plugin/toolkit | Kubernetes runtime | platform |
| ECC/Xid increase | NVML/DCGM/logs | GPU/driver | hardware/vendor |
| low training throughput | PCIe/NUMA/NCCL | fabric/data | HPC/network |
| checkpoint restore fails | storage logs/object version | data integrity | ML/storage |
| replacement lacks capacity | quota/topology/PDB | scheduling | cloud/platform |
| stale GPU metrics | exporter/scrape | observability | telemetry SRE |

## 8. SLO and change-acceptance evidence

Measure GPU-ready time, replacement RTO, checkpoint RPO, available GPU count, canary failure rate, workload latency/throughput, Xid/ECC recurrence, missing-telemetry duration and rollback completion. Use real service targets rather than invented fleet-wide numbers.

### Assessment: 100 points
| Domain | Points |
|---|---:|
| Hardware, topology and driver architecture | 20 |
| Health, Xid/ECC and observability | 20 |
| Workload safety, capacity and recovery | 20 |
| Incident response and runbooks | 20 |
| Executable lab, tests and audit evidence | 20 |
| **Total** | **100** |

**Pass:** >=90, all critical gates clear, successful approved staging recovery, independent signoff. A passing synthetic test suite alone does not satisfy production acceptance.

## 9. Companion executable lab
[Chapter 016 — GPU Hardware Operations Capstone Lab](../labs/016-gpu-hardware-operations-capstone-check.md) provides a fail-closed, standard-library Python 3.11+ acceptance evaluator and a pytest suite. Its Boolean inputs are **operator attestations**, not independent hardware checks.

## References
- [NVIDIA Xid Error Documentation](https://docs.nvidia.com/deploy/xid-errors/)
- [NVIDIA System Management Interface](https://docs.nvidia.com/deploy/nvidia-smi/)
- [NVIDIA NVML](https://docs.nvidia.com/deploy/nvml-api/)
- [NVIDIA DCGM](https://docs.nvidia.com/datacenter/dcgm/latest/)
- [Kubernetes Device Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)
- [Kubernetes Safely Drain a Node](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/)
- [Amazon EKS Managed Node Groups](https://docs.aws.amazon.com/eks/latest/userguide/managed-node-groups.html)
- [Azure AKS GPU Workloads](https://learn.microsoft.com/en-us/azure/aks/gpu-cluster)

**Part 02 complete by publication:** Chapters 009–016 (8/8). **Next:** Chapter 017 — CUDA Runtime and Driver Compatibility (Part 03).
