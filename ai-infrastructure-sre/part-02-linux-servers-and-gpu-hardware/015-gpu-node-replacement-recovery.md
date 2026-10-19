# Chapter 015 — GPU Node Replacement and Recovery

**Part 02 — Linux, Servers and GPU Hardware · Chapter 015 of 100**  
**Status:** Published documentation. Lab tests, staging replacement exercise, independent review and production acceptance remain pending.

## Learning outcomes
Recognize when a GPU node should be repaired, quarantined, reimaged or replaced; preserve tenant workloads and training checkpoints; recover EKS, AKS and bare-metal nodes; verify GPU identity and driver/runtime readiness; apply change-control, capacity, data-integrity and security gates; and prove post-replacement reliability.

## 1. Replacement architecture and failure domains
```text
Incident / change ticket + fleet inventory
                 |
         Fault classification
       /          |           \
 hardware      software     cloud host
       \          |           /
         Quarantine decision
                 |
     N-1 capacity + checkpoint gate
                 |
     Drain / stop admission (approved)
                 |
    Replace or rebuild approved node
                 |
  Image provenance + identity bootstrap
                 |
 Driver -> NVML -> toolkit -> device plugin
                 |
 GPU smoke + data restore + workload SLO
                 |
       Reinstate + evidence + RCA
```
A Kubernetes `Ready` node can have zero allocatable GPUs or an unusable NVIDIA runtime. Successful replacement requires **hardware inventory, driver initialization, GPU scheduling, application smoke, storage/network access and telemetry**, not simply a successful VM boot.

### Repair versus replacement
| Observation | Candidate action | Caveat |
|---|---|---|
| Single workload CUDA failure, host healthy | troubleshoot workload first | avoid needless fleet churn |
| Driver regression after image rollout | rollback known-good image | confirm kernel/driver matrix |
| Repeated critical Xid/uncorrectable ECC | isolate and vendor hardware triage | classify Xid using official guide |
| PCIe GPU disappears after reboot | platform and firmware triage | check BMC/PCIe/driver evidence |
| Node filesystem corruption | reimage or replace | preserve logs and recover persistent data |
| Managed VM host fault | provider-supported replacement | underlying host details may be hidden |
| Node stale or unreachable | verify control plane/network | do not assume physical GPU failure |

## 2. Data safety, state and disruption
GPU nodes should be treated as **replaceable compute** only when workload state and persistent data are managed independently. Training checkpoints, object-store writes, mounted datasets, local NVMe caches, ephemeral model weights, RDMA/NCCL ranks and inference routing require different recovery strategies.

- **Training:** verify checkpoint age, durability, consistency and a successful restore test; coordinate distributed ranks and admission before disruption.
- **Inference:** check warm-up latency, load-balancer deregistration, minimum replicas, PDBs, queue depth and user-facing error budgets.
- **Stateful storage:** identify hostPath/local PV and local NVMe dependencies; node replacement can make data inaccessible or irrecoverable.
- **Multitenancy:** prevent tenant workload migration to unapproved pools and preserve GPU isolation and identity controls.
- **Capacity:** verify real quota and N-1 headroom across zones, instance families and topology constraints; nominal GPU count is insufficient.

`kubectl drain` respects eviction behavior but can be blocked by PDBs, unmanaged pods or local data; never bypass these protections automatically. DaemonSet pods are handled separately. A drain is not a checkpoint mechanism.

## 3. Replacement decision gates
| Gate | Evidence | Hard stop |
|---|---|---|
| authorization | ticket, owner, on-call approval | missing |
| fault evidence | GPU UUID, Xid/ECC, NVRM, timestamps | incomplete for irreversible action |
| capacity | available allocatable GPUs, quota, N-1, zone | insufficient |
| workload safety | checkpoints/PDB, replicas, drain plan | unverified |
| data safety | object-store checkpoint, local PV inventory | potential loss |
| replacement image | signed digest, supported driver/kernel | unapproved |
| identity | scoped cloud role, workload identity | broad/static credentials |
| telemetry | DCGM, kubelet, app SLO, pager | blind rollout |
| recovery | known-good image, rollback or alternate pool | untested |

## 4. Read-only triage commands
Run only in an authorized environment. Redact identifiers and process data.
```bash
date -u
uname -r
lspci -nn | grep -Ei 'nvidia|3d controller'
nvidia-smi -L
nvidia-smi --query-gpu=uuid,name,driver_version,pci.bus_id --format=csv
nvidia-smi -q -d ECC,PERFORMANCE,PCI
journalctl -k -b --no-pager | grep -Ei 'NVRM|Xid|AER|pcie' | tail -n 100
lsblk -o NAME,TYPE,SIZE,MOUNTPOINTS
df -hT
```
```bash
kubectl config current-context
kubectl get nodes -L topology.kubernetes.io/zone
kubectl describe node APPROVED_GPU_NODE
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp
kubectl get pdb -A
kubectl get pv,pvc -A
kubectl get daemonsets -A
```
`kubectl get pv,pvc -A` behavior varies by resource scope and CLI version; query `kubectl get pv` and `kubectl get pvc -A` separately if needed. Inspect PDBs and local volumes before any disruptive action.

## 5. Replacement workflows by platform
### AWS EKS
Confirm GPU instance-family availability, service quotas, node group/Karpenter ownership, GPU-optimized image support, bootstrap and driver ownership, IAM role/IRSA or Pod Identity, EBS/local NVMe dependencies and EFA requirements. Prefer the supported managed-node-group or Karpenter lifecycle over manual deletion of an unmanaged EC2 instance. Validate node labels, taints, topology, NVIDIA driver and `nvidia.com/gpu` allocatable after replacement.

### Azure AKS
Confirm GPU VM SKU availability and zone capacity, managed node pool/VMSS lifecycle, supported AKS image/driver strategy, managed identity, Azure Disk versus ephemeral disk dependencies, and network/RDMA support. Use the supported AKS node pool replacement/upgrade workflow, not ad-hoc VMSS mutations. Verify node GPU allocatable and staging workload readiness.

### Bare metal / self-managed
Record BMC/Redfish, serial/asset, rack, power, BIOS, firmware, GPU UUID, NIC/RDMA fabric and local NVMe. Coordinate physical replacement and re-enrollment; validate Secure Boot, image signatures, out-of-band recovery and firmware compatibility. A new GPU UUID requires asset and telemetry reconciliation.

## 6. Approved staged recovery workflow
1. Open incident/change and preserve UTC evidence, host/GPU UUID, driver, hardware logs and workload symptoms.
2. Confirm affected node and blast radius, replicas, training ranks, checkpoint restore, data location, quota and N-1 capacity.
3. Halt new workload placement using approved procedures. Coordinate checkpoint, traffic draining and PDB-aware eviction.
4. Remove/rebuild/replace using the platform-supported lifecycle. Never use a destructive command copied from a tutorial without a scoped change.
5. Bootstrap a signed, pinned image with one NVIDIA driver owner; validate cloud identity, network, storage and telemetry.
6. Verify `nvidia-smi`, GPU count, Xid/ECC status, topology, driver/runtime, device plugin and Kubernetes allocatable.
7. Run approved pinned CUDA and application smoke jobs; for training, validate NCCL and checkpoint recovery.
8. Restore traffic gradually, compare TTFT/TPOT or training throughput, monitor error budget, record final signoff.
9. Reconcile CMDB, GPU UUID, old-node access/credentials, monitoring labels, cloud cost and incident RCA.

## 7. SLOs and acceptance signals
Track mean time to detect, mean time to replace, boot-to-GPU-ready, recovery RTO, checkpoint RPO, available GPU capacity, replacement success rate, GPU inventory drift, Xid/ECC recurrence, node and workload errors, queue depth, and spend from double-running capacity. Targets must come from the actual service SLO and supported fleet design.

## 8. Eight incident exercises
1. **Uncorrectable ECC:** preserve evidence, stop unsafe placement, recover workload on healthy GPU and escalate vendor.
2. **Driver broken after kernel update:** compare module signing/version, restore approved image and smoke test.
3. **Node Ready but zero GPUs:** investigate plugin/operator, driver and runtime; reject false acceptance.
4. **GPU host disappears mid-training:** validate durable checkpoint and recover distributed ranks safely.
5. **Replacement GPU in wrong zone:** enforce topology, quota and resilience before resuming admission.
6. **Local NVMe checkpoint only:** detect data-loss risk; stop replacement until recovery decision is approved.
7. **New node has stale IAM identity:** quarantine and fix identity, audit access and rotate affected credentials.
8. **Replacement has lower throughput:** compare PCIe/NUMA, NIC/RDMA, GPU SKU, power and application benchmarks.

## 9. Troubleshooting matrix
| Symptom | Investigate | Owner |
|---|---|---|
| new node never joins | bootstrap, kubelet, DNS, identity | cloud/Kubernetes |
| node joins, GPU missing | PCIe, driver, operator/plugin | GPU SRE |
| GPU count mismatches | SKU, MIG, plugin, asset | GPU platform |
| CUDA fails in pod | container toolkit, image/driver | runtime |
| NCCL fails | RDMA, GPU/NIC topology, rank state | HPC/network |
| checkpoint restore fails | storage IAM, integrity, RPO | data/ML |
| PDB blocks eviction | replicas and disruption budget | workload owner |
| inference latency regresses | warm-up, routing, clocks, capacity | inference SRE |
| telemetry missing | exporter, scrape, UUID labels | observability |
| duplicate node cost | ASG/VMSS/Karpenter lifecycle | cloud FinOps |

## 10. Operational runbooks
**RB-015-A — Fault triage:** classify GPU hardware, driver, runtime, workload or control-plane failure; preserve evidence.

**RB-015-B — Capacity and data gate:** confirm N-1 headroom, quota, checkpoint, local PV and workload owner.

**RB-015-C — Safe eviction:** coordinate training ranks, inference routing, PDBs, admission and approved drain.

**RB-015-D — Cloud replacement:** use supported EKS/AKS node lifecycle, image, IAM/identity and GPU operator ownership.

**RB-015-E — Bare-metal replacement:** coordinate BMC/asset, firmware, Secure Boot, NVMe and fabric enrollment.

**RB-015-F — GPU readiness:** validate driver, UUID/count, ECC/Xid, container toolkit, device plugin, CUDA and NCCL.

**RB-015-G — Rollback and failback:** halt admission, restore previous approved node/pool, revalidate workload and SLO.

**RB-015-H — Closure/RCA:** update asset inventory, evidence, root cause, recurring failure mitigation and cost.

## 11. Executable lab
[Chapter 015 — GPU Node Replacement and Recovery Acceptance Lab](../labs/015-gpu-node-replacement-recovery-check.md) implements a synthetic, fail-closed replacement evaluator with Python 3.11+ and pytest. The lab **does not** delete nodes, call cloud APIs or prove real checkpoint recovery.

## 12. Final 100-point assessment
| Area | Points |
|---|---:|
| Replacement architecture and failure classification | 20 |
| Workload/data protection and capacity | 20 |
| EKS/AKS/bare-metal operations | 20 |
| Incidents and runbooks | 20 |
| Executable lab and recovery evidence | 20 |
| **Total** | **100** |

Pass >=90 with **no** data-loss, security, capacity, telemetry or recovery hard stop.

## References
- [Kubernetes Safely Drain a Node](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/)
- [Kubernetes Disruptions](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)
- [Kubernetes Device Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)
- [Amazon EKS managed node groups](https://docs.aws.amazon.com/eks/latest/userguide/managed-node-groups.html)
- [AKS GPU workloads](https://learn.microsoft.com/en-us/azure/aks/gpu-cluster)
- [NVIDIA Xid Errors](https://docs.nvidia.com/deploy/xid-errors/)

**Next:** Chapter 016 — GPU Hardware Operations Capstone.
