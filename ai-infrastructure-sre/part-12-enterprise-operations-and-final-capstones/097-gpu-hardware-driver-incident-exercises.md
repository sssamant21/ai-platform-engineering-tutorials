# Chapter 097 — Run GPU Hardware and Driver Incident Exercises

**Part 12 — Enterprise Operations and Final Capstones | AI Infrastructure Engineering & SRE**

**Publication:** PUBLISHED. Staging incident exercises, hardware fault validation, recovery timings, security review and independent production acceptance **PENDING**.

## 1. Mission

Develop and rehearse an enterprise GPU incident-response system for NVIDIA GPU Xid/ECC events, driver/kernel failures, GPU Operator/device-plugin outages, PCIe/NVLink faults, thermal/power throttling and GPU node replacement. Protect inference SLOs and checkpointed training state while collecting forensic evidence and coordinating safe recovery.

**Prerequisites:** Chapters 091–096; approved GPU staging cluster; tested CUDA/NCCL baselines; GPU fleet inventory; monitoring and on-call routes; rollback and checkpoint runbooks.

### Outcomes

1. Classify GPU hardware, driver, runtime, operator and workload failures.
2. Detect GPU incidents using DCGM, Kubernetes and host logs.
3. Distinguish workload failure from node/GPU failure and network/storage dependencies.
4. Contain faults without causing unnecessary production impact.
5. Recover inference and training using verified rollback, node replacement and checkpoint state.
6. Measure detection, mitigation, recovery, data loss and customer impact.
7. Produce RCA/COA and independently reviewed incident evidence.

## 2. Failure-domain map

\`\`\`text
Inference p99 / training progress degradation
                  |
        workload or infrastructure?
                  |
     ┌────────────┼───────────────┐
     |            |               |
 GPU device    Driver/runtime   Kubernetes/node
 Xid/ECC       CUDA mismatch    operator/plugin
 HBM/power     kernel module    scheduling/taints
     |            |               |
     └────────────┼───────────────┘
                  |
        PCIe / NVLink / RDMA / storage
                  |
   isolate → mitigate → restore → verify → RCA
\`\`\`

A single GPU Xid event does not by itself establish root cause or mandate node replacement. Interpret exact Xid code, driver version, workload, recurrence and vendor guidance. Double-bit ECC and other uncorrectable errors require prompt hardware-specific triage.

## 3. Severity and incident ownership

| Severity | Example | Immediate objective |
|---|---|---|
| SEV1 | multiple serving GPUs fail; protected inference SLO breach | restore customer service, page incident commander |
| SEV2 | training pool loses capacity; checkpoint recovery at risk | contain and restore safe throughput |
| SEV3 | isolated GPU fault with available redundancy | quarantine, investigate and replace |
| SEV4 | non-impacting warning or planned maintenance anomaly | ticket, monitor and follow up |

Severity is **illustrative**; adopt organization-specific incident policy. Assign incident commander, GPU platform lead, workload owner, security contact, cloud/provider support and communications owner. Establish a timeline with UTC timestamps.

## 4. Detection and initial triage

### Read-only Kubernetes commands

\`\`\`bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -100
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.allocatable.nvidia\.com/gpu}{"\n"}{end}'
kubectl get pods -A | grep -Ei 'gpu-operator|device-plugin|dcgm' || true
\`\`\`

### Approved GPU-node inspection

\`\`\`bash
nvidia-smi -L
nvidia-smi -q -d ECC,ERROR,PERFORMANCE,TEMPERATURE,POWER
nvidia-smi --query-gpu=uuid,name,driver_version,temperature.gpu,memory.used --format=csv
journalctl -k --no-pager | grep -Ei 'NVRM|Xid|nvidia|pcie' | tail -100
uname -r
\`\`\`

These require node access and supported driver queries. Capture exact GPU UUID, host, timestamp, driver/kernel, model, MIG configuration, Xid code, ECC history, workload Pod UID and customer impact. Avoid dumping secrets, prompts or model weights into evidence.

## 5. Xid/ECC investigation

Xid messages are driver-reported GPU error events with code-specific meanings. Some indicate application misuse; others point to hardware, memory, driver or system issues. Do **not** use a blanket rule that every Xid means a defective GPU.

**Decision workflow:**
1. Verify event authenticity and timestamp; correlate with DCGM and host logs.
2. Identify affected GPU UUID, node, MIG partition and Pod/job.
3. Check whether the event repeats across independent workloads.
4. Compare driver/kernel/operator versions against known-good nodes.
5. Inspect ECC (correctable versus uncorrectable), retired pages and vendor health guidance.
6. Check thermal, power, PCIe and NVLink health.
7. Decide workload restart, GPU reset (only if supported/approved), node replacement or vendor escalation.
8. Confirm CUDA/NCCL and workload SLO recovery before returning capacity.

GPU reset can disrupt other processes, may be unsupported for some configurations and is **not** a safe default remediation.

## 6. Driver/kernel failure modes

| Symptom | Likely area | Evidence |
|---|---|---|
| \`nvidia-smi\` cannot communicate | driver module/device | kernel logs and module state |
| GPU disappears after reboot | PCIe/firmware/driver | inventory comparison |
| Driver pod CrashLoop | GPU Operator/node image | pod logs and compatibility matrix |
| CUDA initialization fails | driver/runtime/toolkit | framework CUDA test |
| NCCL hangs after driver update | compatibility/topology | rank logs and baseline |
| MIG devices change unexpectedly | partition policy/operator | profile and allocatable |
| Xid storm on one node | hardware/driver/workload | GPU UUID correlation |
| GPU throttling | temperature/power/cooling | clock/power telemetry |

Do not mix preinstalled host drivers and operator-managed drivers without an approved design. Kernel and driver compatibility must be checked before node image upgrades.

## 7. GPU Operator and device-plugin outage

A healthy Kubernetes node can report zero allocatable GPUs when device plugin registration fails. Distinguish:
- GPU hardware absent or unhealthy;
- driver loaded but device plugin not registered;
- GPU Operator reconciliation/CRD or permissions failure;
- runtime injection broken for containers;
- MIG resource profile mismatch.

Review namespace, DaemonSet/Pod health, events, logs and allocatable resources. Avoid restarting all GPU Operator components cluster-wide during a live incident; use scoped, approved remediation.

## 8. Thermal, power and interconnect faults

Track GPU temperature, clock throttling reasons, power draw, PCIe link status, NVLink/NVSwitch health and NIC errors. A performance regression may result from:
- thermal or power throttling;
- degraded PCIe/NVLink path;
- GPU-to-NIC topology change;
- fabric congestion or RDMA fallback;
- node contention and HBM pressure.

Correlate with Chapter 093 CUDA/NCCL and Chapter 095 storage/network baselines. A passing \`nvidia-smi\` query does not prove full performance.

## 9. Containment and safe node evacuation

**Safety gate:** decide whether to protect serving traffic, checkpoint training and retain forensic evidence before moving workloads.

For approved staging drills only:

\`\`\`bash
# Example mutation: do not run against production without change approval.
kubectl cordon <GPU_NODE_NAME>
kubectl get pods -A --field-selector spec.nodeName=<GPU_NODE_NAME>
\`\`\`

Cordon prevents new scheduling but does not evict running Pods. \`kubectl drain\` can evict serving replicas or interrupt distributed training and may be blocked by PodDisruptionBudgets; plan application-specific graceful termination and checkpoint completion. Never force-delete stateful training Pods as a generic remediation.

## 10. Inference service recovery

1. Confirm user-visible p99, TTFT/TPOT, error rate and warm replica count.
2. Route traffic away from unhealthy replicas only after healthy compatible capacity exists.
3. Check node/GPU allocation, model artifact integrity and readiness.
4. Replace node or roll back driver/image using approved blue-green plan.
5. Rewarm models and verify HBM/latency under load.
6. Confirm SLO and error-budget stabilization before incident closure.

A \`Ready\` Pod can still serve incorrect or slow responses; end-to-end load/quality gates are mandatory.

## 11. Distributed training recovery

1. Record failed rank, world size, last successful step and checkpoint marker.
2. Preserve logs and identify GPU versus fabric or application failure.
3. Stop/reconcile remaining ranks safely to avoid divergent state.
4. Restore the last **complete verified checkpoint**, including optimizer, RNG and data sampler state.
5. Restart on a compatible GPU topology and verify NCCL collectives.
6. Validate resumed step, model quality and lost-work budget.

Checkpoint replication without tested restore integrity is insufficient. Avoid reusing partial checkpoints after node failure.

## 12. Incident metrics and evidence

| Metric | Definition |
|---|---|
| MTTD | first qualifying fault to detection |
| MTTA | detection to human acknowledgement |
| MTTR | incident start to verified service recovery |
| Customer impact | failed/slow requests and affected tenants |
| Training impact | lost successful steps and restart duration |
| GPU capacity loss | unavailable compatible GPU-hours |
| SLO burn | consumed error budget |
| Recovery correctness | CUDA/NCCL, model and checkpoint checks |
| Cost impact | idle/replaced GPUs, egress and remediation |

Capture incident timeline, exact commands, approvals, dashboards, sanitized logs, owner decisions, rollback and corrective actions.

## 13. Safe incident exercise design

Use **simulation first**: synthetic DCGM/alert payloads, recorded Xid logs, fake node conditions and tabletop timelines. For live staging fault injection, obtain change approval, ensure spare compatible GPUs, isolate the cluster and define automatic abort triggers. Do not deliberately overheat GPUs, induce ECC corruption, kill kernel drivers or tamper with firmware as training exercises.

### Example drill types

- Inject synthetic Xid alert and verify routing/triage.
- Remove one disposable GPU worker from staging capacity using provider-supported procedures.
- Simulate driver incompatibility through a known-bad **test image** rather than corrupting the host.
- Simulate device-plugin outage in isolated staging.
- Reproduce CUDA OOM in a bounded test Pod.
- Trigger synthetic inference SLO breach and rollback.
- Simulate training rank loss and checkpoint restore.

## 14. Six production runbooks

**R1 — Xid/ECC alert:** identify code/GPU UUID, correlate hardware/driver/workload, protect service, escalate vendor and verify recovery.

**R2 — Driver unavailable:** inspect kernel module, node image, Operator status and known-good versions; replace/canary node rather than unsafe global driver changes.

**R3 — GPU Operator outage:** inspect reconciliation, DaemonSets, RBAC, device plugin registration and MIG; restore scoped component with rollback.

**R4 — Inference GPU incident:** fail traffic to healthy warm replicas, monitor p99/TTFT, restore model and validate user SLO.

**R5 — Training GPU failure:** preserve checkpoint and logs, reconcile ranks, restart compatible topology and verify optimizer state.

**R6 — Thermal/interconnect degradation:** inspect clocks, power, cooling, PCIe/NVLink/NIC counters and NCCL baseline; quarantine degraded node.

## 15. Sixteen hands-on exercises

1. Approve staging scope, incident roles and abort criteria.
2. Inventory GPU hardware, drivers, firmware and topology.
3. Review DCGM Xid/ECC alert definitions.
4. Build a GPU incident evidence checklist.
5. Triage a recorded Xid event.
6. Distinguish correctable versus uncorrectable ECC.
7. Diagnose simulated driver/kernel incompatibility.
8. Diagnose device-plugin/Operator registration failure.
9. Review thermal/power and interconnect telemetry.
10. Run synthetic GPU health alert and escalation.
11. Simulate safe node cordon and capacity failover.
12. Recover inference with measured SLO.
13. Recover training from a verified checkpoint.
14. Compare pre/post CUDA/NCCL performance.
15. Write RCA, corrective actions and rollback review.
16. Conduct independent incident acceptance assessment.

## 16. Twelve troubleshooting scenarios

| Scenario | Root-cause focus |
|---|---|
| Xid occurs once, no recurrence | application/driver/hardware context |
| Repeated Xid on one GPU | hardware and vendor escalation |
| ECC uncorrectable | device quarantine/replacement |
| Node Ready, GPU allocatable zero | device plugin/driver |
| GPU Operator pods fail after upgrade | version/RBAC/kernel |
| \`nvidia-smi\` fails on one node | driver/device |
| GPU lost after MIG reconfiguration | partition/device resources |
| Serving p99 spikes during node evacuation | warm headroom |
| Training ranks hang after one GPU fault | collective restart |
| NCCL bandwidth regresses | fabric/topology/throttle |
| GPU thermal throttling | cooling/power |
| Alert fires but incident not paged | routing/on-call gap |

## 17. Acceptance rubric (100 points)

| Domain | Points |
|---|---:|
| Detection, telemetry and incident classification | 20 |
| Xid/ECC and driver/Operator diagnosis | 20 |
| Safe containment and workload protection | 20 |
| Inference and training recovery validation | 20 |
| RCA, COAs and incident communications | 10 |
| Evidence and independent review | 10 |
| **Total** | **100** |

**Pass:** ≥85/100 with no critical unsafe fault injection, missing customer SLO recovery, corrupted checkpoint, unresolved hardware safety issue, untested rollback or absent independent review. **Current status: exercises documented, not executed.**

## 18. References

- [NVIDIA Xid Errors](https://docs.nvidia.com/deploy/xid-errors/)
- [NVIDIA DCGM](https://docs.nvidia.com/datacenter/dcgm/latest/)
- [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/)
- [Kubernetes Safely Drain a Node](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/)
- [NVIDIA NCCL User Guide](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/)
- [Google SRE Workbook — Incident Response](https://sre.google/workbook/incident-response/)

**Next:** Chapter 098 — Run Distributed Training Failure Recovery.
