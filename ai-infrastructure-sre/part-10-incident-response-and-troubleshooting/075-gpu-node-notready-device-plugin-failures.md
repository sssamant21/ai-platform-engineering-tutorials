# Chapter 075 — GPU Node NotReady and Device Plugin Failures

**Part 10 — Incident Response and Troubleshooting | AI Infrastructure Engineering & SRE**

**Publication:** Documentation published; offline execution, live staging validation, security review and independent production acceptance are pending.

## 1. Mission and learning objectives

Diagnose Kubernetes GPU nodes that become \`NotReady\`, report incorrect GPU allocatable capacity, or fail NVIDIA device-plugin registration. Restore healthy GPU scheduling safely while preserving training checkpoints, inference availability, evidence, and tenant isolation.

After this chapter you can: distinguish node readiness from GPU device health; trace GPU resource advertisement through kubelet and device plugin; investigate NVIDIA GPU Operator, drivers, container runtime and Node Feature Discovery; detect failures involving MIG, taints, scheduling and GPU Xid; choose non-disruptive recovery; and produce an auditable incident handoff.

**Prerequisites:** Chapters 011–018 (GPU node fundamentals), 025–030 (Kubernetes GPU scheduling), 067–074 (GPU observability). Required: read-only Kubernetes access, approved GPU host access for \`nvidia-smi\`/journals, staging cluster for any change.

## 2. The GPU node dependency chain

\`\`\`text
Cloud/physical host → Linux kernel → NVIDIA driver/NVML
                                → NVIDIA container toolkit/runtime
                                → NVIDIA device plugin ↔ kubelet
                                → Kubernetes extended GPU resources
                                → scheduler / admission / GPU workloads
                    └─ GPU Operator / NFD / DCGM Exporter / telemetry
\`\`\`

**Separate these states:**

| State | Interpretation | What it does NOT prove |
|---|---|---|
| Node \`Ready=True\` | kubelet and required node checks pass | GPU hardware, driver and plugin are healthy |
| Node \`Ready=False\` | kubelet reports node not ready | GPU failure is the root cause |
| Node \`Ready=Unknown\` | control plane has lost timely node heartbeats | host is powered off or driver is broken |
| \`nvidia.com/gpu\` capacity present | device plugin advertised a resource | workloads can successfully initialize CUDA |
| \`nvidia.com/gpu\` allocatable = 0/absent | no schedulable GPU resource advertised | physical GPU is absent |
| DCGM exporter \`up=1\` | exporter scrape succeeds | every GPU is healthy and available |
| GPU Operator components Running | operator pods are running | driver/runtime/plugin registration succeeded |

Kubernetes taints such as \`node.kubernetes.io/not-ready\` and \`node.kubernetes.io/unreachable\` can affect scheduling and eviction. Actual behavior depends on Kubernetes version, tolerations, pod policies and controller timing; do not assume an immediate drain.

## 3. Incident classification

| Symptom | Likely failure domains | Priority |
|---|---|---|
| Node NotReady and no kubelet heartbeat | host, network, kubelet, disk/memory pressure | assess impact and failure domain |
| Node Ready but no GPU capacity | device plugin, driver, GPU Operator, MIG strategy | GPU scheduling incident |
| GPU capacity present but CUDA init fails | driver/runtime/toolkit, device permissions, workload image | workload impact |
| GPUs disappear after upgrade | driver/kernel compatibility, plugin registration, Operator reconciliation | rollback candidate |
| MIG slices missing | MIG configuration/profile, strategy and restart behavior | allocation mismatch |
| Device plugin CrashLoopBackOff | NVML failure, privileges, socket mount, incompatible version | platform incident |
| Pending GPU pods | quota, taints, affinity, topology, no allocatable GPU | may be scheduling rather than node failure |
| Xid 79 or GPU absent from \`nvidia-smi\` | PCIe/host/GPU failure | urgent hardware escalation |

Severity is driven by workload impact and recovery objectives, not merely by an individual status flag.

## 4. Read-only triage: first 10 minutes

\`\`\`bash
# Confirm context and blast radius
kubectl config current-context
kubectl get nodes -o wide
kubectl get node <gpu-node> -o yaml
kubectl describe node <gpu-node>
kubectl get events -A --sort-by=.lastTimestamp | tail -100

# GPU workloads, including placement and current phase
kubectl get pods -A -o wide --field-selector spec.nodeName=<gpu-node>
kubectl get pods -A --field-selector=status.phase=Pending
kubectl get resourcequota -A
kubectl get priorityclass

# Discover operator/plugin installation rather than assuming namespace
kubectl get pods -A -o wide | grep -Ei 'nvidia|gpu-operator|device-plugin|nfd|dcgm'
kubectl get daemonsets -A | grep -Ei 'nvidia|gpu|nfd'
kubectl get clusterpolicies.nvidia.com -A 2>/dev/null || true
\`\`\`

For the selected node, record:
- \`status.conditions\` including Ready, MemoryPressure, DiskPressure and PIDPressure.
- \`status.capacity\` versus \`status.allocatable\`, including extended GPU resource names.
- Node lease and last heartbeat, taints, unschedulable flag and Kubernetes events.
- Pod UIDs, GPU resource requests/limits, restart history and owning controllers.
- Current driver/kernel/GPU Operator/device plugin versions and last rollout/change.

**Avoid conflating** the kubelet heartbeat \`Lease\` with complete GPU functional health.

## 5. Host and driver evidence

With approved host access:

\`\`\`bash
uname -r
nvidia-smi -L
nvidia-smi --query-gpu=uuid,name,driver_version,memory.total --format=csv
nvidia-smi -q -d ECC,PCI,PERFORMANCE
systemctl status kubelet --no-pager
journalctl -u kubelet --since '-2 hours' --no-pager
journalctl -k --since '-2 hours' --output=short-iso-precise | grep -Ei 'NVRM|Xid|nvidia|AER'
df -h
free -h
\`\`\`

If \`nvidia-smi\` fails, distinguish missing kernel modules, unsupported/failed device, NVML mismatch, driver initialization failure and inaccessible permissions. Check driver/kernel compatibility and recent node image upgrades. Preserve redacted journal and \`nvidia-bug-report.sh\` output only under approved data-handling policy; diagnostic bundles can contain sensitive host metadata.

### 5.1 GPU device plugin registration

The NVIDIA device plugin advertises extended resources to kubelet. Inspect the actual DaemonSet:

\`\`\`bash
kubectl -n <plugin-namespace> get daemonset <plugin-daemonset> -o yaml
kubectl -n <plugin-namespace> get pods -o wide -l <actual-label-selector>
kubectl -n <plugin-namespace> logs <plugin-pod> --timestamps=true --since=2h
kubectl -n <plugin-namespace> describe pod <plugin-pod>
\`\`\`

Look for NVML initialization errors, \`/var/lib/kubelet/device-plugins\` socket problems, device enumeration, driver initialization, node selectors, runtime configuration and version compatibility. Device plugin pod health is necessary but not sufficient: compare node allocatable to independently discovered physical/MIG devices.

### 5.2 GPU Operator and NFD

The NVIDIA GPU Operator may manage drivers, toolkit, device plugin, DCGM and related components, depending on deployment flags. Node Feature Discovery can drive scheduling labels. Confirm actual \`ClusterPolicy\` and installed component ownership; do **not** install another driver over an operator-managed one or restart all operator components to fix one node.

## 6. MIG, topology and resource accounting

MIG exposes profile-specific resources under configured device-plugin strategy, e.g. \`nvidia.com/mig-1g.10gb\` on supported GPU generations and configurations. **Names and profile sizes vary by GPU model and MIG strategy**. A \`mixed\` strategy can advertise profile-specific resources; a \`single\` strategy may advertise \`nvidia.com/gpu\` as slices. Confirm live node allocatable, actual MIG layout and operator configuration before interpreting counts.

A physical GPU is not interchangeable with a MIG instance; neither is an arbitrary set of free GPUs guaranteed to satisfy a topology-constrained training job.

## 7. Decision tree: symptom → proof → action

\`\`\`text
Node Ready?
 ├─ No → host reachable? kubelet running? lease current? pressure?
 │        → check cloud/network/OS first, then GPU driver evidence
 └─ Yes → GPU resource advertised?
          ├─ No → driver/NVML → plugin socket/logs → Operator/MIG config
          └─ Yes → workload CUDA init succeeds?
                   ├─ No → toolkit/runtime, image, device permissions, driver
                   └─ Yes → investigate scheduler placement, quotas,
                            topology, performance and workload health
\`\`\`

Do not automatically cordon/drain or reset a GPU based on a single metric or event. Before disruptive remediation, assess inference redundancy, training checkpoint freshness, distributed job gang behavior, storage integrity, node pool spare capacity, change window and rollback. Cordon prevents new scheduling but does **not** evict existing pods. Drain evicts eligible pods and can interrupt stateful GPU workloads.

## 8. Staging-only remediation patterns

### 8.1 Approved cordon/drain workflow

\`\`\`bash
# EXAMPLE ONLY — NEVER RUN WITHOUT APPROVED CHANGE AND WORKLOAD OWNER
kubectl cordon <gpu-node>
# Inspect all workloads and checkpoint/replica safety before considering drain.
# kubectl drain <gpu-node> --ignore-daemonsets --delete-emptydir-data
# Only after validated node repair and GPU acceptance:
# kubectl uncordon <gpu-node>
\`\`\`

The commented drain command is intentionally not an instruction to run: \`--delete-emptydir-data\` can destroy ephemeral job state. Respect PodDisruptionBudgets, local storage, job owners and Kubernetes eviction behavior. Do not force delete pods to bypass safety.

### 8.2 Component recovery

Prefer the smallest scoped approved change: fix node labels/taints only if policy incorrect; restore driver/runtime compatibility; roll a single affected plugin pod only after verifying blast radius; perform host reboot or GPU reset only with vendor guidance and checkpoint/workload approval. Recheck device registration, CUDA test and SLOs before returning node to service.

## 9. Production-grade alerts

Alert on independent layers:
1. \`kube_node_status_condition{condition="Ready",status="true"} == 0\` (kube-state-metrics dependent; \`Unknown\` also needs coverage).
2. Missing expected GPU resources relative to inventory, **not** merely exporter series count.
3. Device plugin unavailable/CrashLoopBackOff (actual labels and deployment names).
4. GPU workload admission/queue SLO degradation.
5. Xid/ECC/PCIe indicators and user-facing inference/training impact.

\`up\` cannot detect targets removed from discovery. GPU health must not be inferred from an exporter being scrapeable. Validate all alert rule labels and series against the installed stack; distinguish warning from critical based on customer impact.

## 10. Sixteen hands-on exercises

1. Inventory node GPU models, drivers, Operator and plugin versions.
2. Map node readiness conditions, leases, taints and resource capacity.
3. Reconcile physical GPUs, MIG profiles and Kubernetes allocatable.
4. Inspect plugin DaemonSet scheduling and logs.
5. Inspect GPU Operator ClusterPolicy and component health.
6. Capture kubelet logs and node pressure signals.
7. Capture driver/NVML and Xid evidence safely.
8. Diagnose a **synthetic** Ready node with zero GPU allocatable.
9. Diagnose a **synthetic** NotReady node with healthy GPU hardware.
10. Diagnose Pending pods caused by topology/taints/quotas.
11. Validate an approved staging-only device-plugin restart scenario.
12. Tabletop a driver/kernel mismatch after node image upgrade.
13. Tabletop Xid 79 during a distributed training checkpoint.
14. Validate missing-GPU and NodeNotReady alert routing to test receiver.
15. Produce a recovery and rollback plan with explicit go/no-go.
16. Submit evidence, review, score and post-incident improvements.

## 11. Troubleshooting matrix

| Symptom | Read-only evidence | Avoid |
|---|---|---|
| Node Ready, GPU missing | \`nvidia-smi\`, plugin logs, allocatable | assuming kubelet fault |
| Node NotReady, GPU healthy | kubelet, lease, network, pressure | GPU reset |
| Device plugin CrashLoop | NVML, socket, privileges, image | restarting entire fleet |
| MIG mismatch | profile strategy, live allocation | comparing slices to physical GPUs |
| CUDA init error | runtime/toolkit, driver, image | arbitrary driver downgrade |
| GPU pods Pending | quota, affinity, topology, taints | blind node scaling |
| Xid 79 | kernel, PCIe, node and workload evidence | automatic node reboot |
| Post-upgrade capacity loss | rollout, driver/kernel matrix, Operator | unapproved mass rollback |

## 12. Production runbooks

**A — Node NotReady:** determine host/network/kubelet failure versus GPU symptoms; capture lease, conditions and node events; protect workloads; restore node under change control; confirm heartbeat and scheduling.

**B — Device plugin registration:** verify driver/NVML and plugin logs/socket; inspect Operator and MIG config; repair smallest failed component; confirm allocatable and approved CUDA smoke test.

**C — Driver/kernel regression:** correlate upgrade revision, module and NVML errors; preserve logs; choose pinned known-good node image/driver under rollback policy; validate GPU and job recovery.

**D — GPU lost-on-bus:** treat Xid 79 as serious hardware/PCIe signal; preserve evidence; coordinate checkpoint/replica safety; engage vendor; return only after verified GPU health and workload probes.

**E — Scheduler starvation:** inspect compatible GPU pool, taints, topology, quota and pending events; adjust admission/placement under policy; verify queue SLO without disrupting active jobs.

## 13. Acceptance gate (100 points)

| Gate | Points |
|---|---:|
| Node condition, lease and GPU inventory analysis | 20 |
| Driver/plugin/Operator diagnostic correctness | 20 |
| MIG and scheduling isolation analysis | 15 |
| Alerting, incident severity and escalation | 15 |
| Staging recovery/tabletop and rollback safety | 20 |
| Runbooks, evidence and independent review | 10 |
| **Total** | **100** |

**Pass:** ≥85/100 with no critical security, data-integrity or unsafe-remediation failure. Offline, staging and review milestones remain pending until actually executed.

## 14. References

- [Kubernetes Node Status](https://kubernetes.io/docs/reference/node/node-status/)
- [Kubernetes Device Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)
- [Kubernetes Safely Drain a Node](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/)
- [NVIDIA Kubernetes Device Plugin](https://github.com/NVIDIA/k8s-device-plugin)
- [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/)
- [NVIDIA Xid Errors](https://docs.nvidia.com/deploy/xid-errors/)

**Next:** Chapter 076 — CUDA Initialization and Driver Errors.
