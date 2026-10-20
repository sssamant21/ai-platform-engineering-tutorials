# Chapter 080 — GPU Failure Triage and Escalation

**Part 10 — Incident Response and Troubleshooting | AI Infrastructure Engineering & SRE**

**Status:** Published documentation. Offline exercises, live GPU staging validation, independent technical review and production acceptance remain pending.

## 1. Mission and learning objectives

Identify GPU hardware and driver failure signals; distinguish transient workload errors from persistent device faults; determine incident severity and blast radius; preserve vendor-ready evidence; and safely escalate to infrastructure, hardware, cloud and application owners. This chapter emphasizes **NVIDIA Xid errors, ECC memory events, PCIe/AER failures, GPU fallen-off-bus, thermal/power limits, NVLink errors and GPU row-remapping/page-retirement signals**.

After completion you can construct a UTC evidence timeline, decide when a GPU node should be quarantined, protect inference availability and training checkpoints, open a complete vendor case, and validate return-to-service criteria.

**Prerequisites:** Chapters 075–079, Kubernetes node operations, DCGM/NVML basics, GPU Operator, NCCL and workload checkpoint recovery. Read-only production diagnosis only; resets, reboots, firmware updates and fault injection require approved maintenance or isolated staging.

## 2. GPU failure domains

\`\`\`text
Application / CUDA / NCCL error
    ↓
CUDA driver / NVML / device plugin
    ↓
NVIDIA kernel driver / NVRM Xid
    ↓
GPU silicon, HBM/ECC, NVLink, PCIe, power, cooling
    ↓
Host motherboard, riser, PSU, BIOS, kernel, fabric
    ↓
Cloud GPU instance / physical hardware vendor
\`\`\`

A CUDA error can be caused by an application bug, a driver incompatibility, an underlying hardware fault or an earlier asynchronous failure. A GPU can appear in \`nvidia-smi\` while failing specific kernels or memory tests. Conversely, a single Xid does not automatically prove irreparable hardware.

## 3. Incident severity and triage matrix

| Signal | Interpretation | Immediate posture |
|---|---|---|
| \`NVRM: Xid\` event | GPU driver detected an error; meaning is code-specific | correlate code, timestamp, GPU UUID, workloads |
| Xid 79 / GPU fallen off bus | serious loss of device accessibility, often PCIe/device related | protect jobs, escalate hardware/platform |
| Xid 48 or uncorrectable ECC | memory fault indicator; action depends on GPU generation and context | preserve ECC/remap evidence, vendor guidance |
| Repeated correctable ECC events | may indicate memory reliability degradation | trend and compare vendor thresholds |
| Row remapping / page retirement | GPU memory fault mitigation/retirement state | assess pending remap and reboot requirements |
| PCIe AER / link down | host-to-device path fault | inspect host/riser/switch, engage hardware owner |
| NVLink error/replay | GPU-to-GPU interconnect issue | compare fabric counters and NCCL symptoms |
| Thermal/power throttling | possible performance limit, not necessarily hardware failure | inspect clocks, temperatures, power and cooling |
| Device disappears from plugin allocatable | driver, plugin, MIG or hardware | correlate host GPU visibility and kubelet |
| DCGM diagnostic failure | health check indicates an issue | preserve exact test and vendor documentation |

**Do not generalize an Xid code across GPU generations or driver releases.** Consult NVIDIA's current Xid documentation and vendor-specific service policies. The decision to RMA/replace hardware is vendor-driven.

## 4. First 10 minutes: impact and containment

1. Confirm incident commander, severity, affected GPU UUID/node/cluster and customer workloads.
2. Check inference replicas, training checkpoint freshness and distributed rank membership.
3. Correlate first CUDA/NCCL error with earliest kernel Xid, PCIe/AER and cloud hardware events.
4. Determine if failure is isolated to one GPU, MIG instance, host, rack/rail or node pool.
5. Preserve logs and metrics before pod eviction or host repair.
6. Cordon only under approved operational policy; **cordon prevents new scheduling but does not evict existing pods**.
7. If drain is considered, obtain workload owner approval and verify checkpoint/PDB/stateful behavior. Never force-drain a running distributed job as a reflex.

### 4.1 Kubernetes evidence

\`\`\`bash
kubectl config current-context
kubectl get nodes -o wide
kubectl describe node <gpu-node>
kubectl get node <gpu-node> -o yaml
kubectl get pods -A -o wide --field-selector spec.nodeName=<gpu-node>
kubectl get events -A --sort-by=.lastTimestamp | tail -100
kubectl get pods -A | grep -Ei 'gpu-operator|nvidia|device-plugin|dcgm'
\`\`\`

Record node conditions, GPU allocatable resources, taints, pod UIDs, image digests, job/replica ownership and recent changes. Node Ready is not a complete GPU health signal.

## 5. GPU and kernel diagnostics

Approved host access only:

\`\`\`bash
nvidia-smi -L
nvidia-smi --query-gpu=uuid,name,driver_version,temperature.gpu,power.draw --format=csv
nvidia-smi -q -d ECC,PCI,PERFORMANCE,POWER,TEMPERATURE
nvidia-smi -q -d PAGE_RETIREMENT,ROW_REMAPPER
journalctl -k --since '-4 hours' --output=short-iso-precise | grep -Ei 'NVRM|Xid|AER|PCIe|nvidia'
lspci -nn | grep -Ei 'NVIDIA|3D controller|VGA'
\`\`\`

Some \`nvidia-smi -q -d\` categories or fields are unsupported on particular drivers/GPUs. Capture unsupported output as evidence, not a healthy result. \`nvidia-smi\` can fail when the device is inaccessible; do not repeatedly reset it to obtain metrics.

### 5.1 Xid timeline

For each Xid event, record:
- UTC timestamp, GPU UUID or PCI bus ID, Xid code, driver/kernel version.
- Earliest application CUDA/NCCL error and first impacted job.
- Host AER, GPU ECC, NVLink, power/thermal and cloud maintenance events.
- Repetition across workloads, pods, drivers or nodes.
- Whether the issue persists after a vendor-approved repair or replacement.

Xid 79 is commonly associated with a GPU becoming inaccessible over PCIe; do not treat it as a guaranteed diagnosis of a failed GPU chip versus host, riser or power path.

### 5.2 ECC and memory reliability

Separate **correctable** and **uncorrectable** ECC, volatile and aggregate counters, row-remapping/page-retirement state and model-specific behavior. A nonzero lifetime counter does not necessarily mean a current incident. Prefer change in counters, correlated Xid and workload failure. Avoid clearing counters before evidence preservation; do not disable ECC as a workaround.

### 5.3 PCIe, NVLink, power and thermal

Inspect AER events, link width/speed and errors, NVLink health where supported, GPU clocks, throttling reasons and thermal/power behavior. Power limits or thermal slowdown can degrade performance without producing a fatal Xid. Engage datacenter or cloud provider for cooling, PSU, riser, cable or motherboard investigations.

## 6. DCGM and fleet health signals

DCGM exporter may expose ECC, Xid, thermal, framebuffer and NVLink metrics depending on installed configuration and GPU generation. **Discover actual metric names, units and labels**; avoid assuming every DCGM counter is enabled or available. Correlate device health with node allocatable, plugin registration and user-facing job failure. A scrape success is not proof of GPU health.

For health alerting, distinguish:
- critical: GPU lost, repeated uncorrectable memory events, widespread job failures;
- warning: new correctable ECC trends, rising thermal throttling, isolated degraded link;
- informational: expected throttling under configured power caps or old aggregate counters.

Set thresholds with vendor guidance and fleet baselines; never hardcode one threshold for all GPU models.

## 7. Escalation decision tree

\`\`\`text
GPU workload failed
 ├─ no host driver/Xid signal → application/runtime/allocator first
 └─ driver or hardware signal present
      ├─ device inaccessible / Xid 79 → host/PCIe/vendor escalation
      ├─ ECC/row-remap indicators → memory reliability/vendor guidance
      ├─ NVLink/RDMA symptoms → interconnect/fabric owner
      ├─ power/thermal symptoms → datacenter/cloud capacity owner
      └─ unclear → preserve evidence, compare healthy peer and changes
\`\`\`

Escalate to vendor when hardware fault evidence is credible, recurrence crosses workloads, the device disappears, ECC/row-remap state requires action, or cloud instance hardware reports degradation. Do not wait for a second production outage when a high-severity hardware condition is already documented.

## 8. Vendor-ready support bundle

Include (redacted and access-controlled):
- GPU SKU, GPU UUID/PCI bus ID, server/instance type and cloud region/AZ if relevant.
- OS/kernel, NVIDIA driver, CUDA, GPU Operator, device plugin, firmware/BIOS versions.
- Incident UTC timeline, Xid/AER/ECC/NVLink events, driver logs and exact errors.
- \`nvidia-smi -q\` relevant excerpts and DCGM diagnostic result if safely obtained.
- Reproduction conditions, number of affected devices, blast radius and mitigation.
- Known-good comparison and recent hardware/software changes.
- Support contract/case ID, owner, priority, requested action and next update.

\`nvidia-bug-report.sh\` and full logs may include sensitive topology or tenant data. Run only when approved, redact and store securely. Never attach credentials, private keys, model weights, patient/customer data or unrestricted kernel logs to a public issue.

## 9. Recovery and return-to-service

A node may return only after:
1. Vendor/platform confirms repair or supported recovery action.
2. Driver/NVML enumerate expected physical GPUs and MIG instances.
3. No unresolved critical Xid/ECC/PCIe/NVLink condition remains.
4. Kubernetes plugin advertises expected allocatable resources.
5. Approved CUDA kernel and NCCL smoke tests pass in staging/maintenance scope.
6. Training checkpoint resume or inference replica probes meet SLO.
7. Incident owner signs off; alerting and monitoring restored.
8. Cordon/taint state is reverted only after acceptance.

GPU reset, module reload, firmware update or reboot can disrupt other workloads and may be unsupported on active MIG/NVLink configurations. Require explicit change approval and rollback. Replacement hardware should be tracked by UUID; do not assume stable GPU indices.

## 10. Sixteen hands-on exercises

1. Inventory GPU UUID, SKU, node and driver versions.
2. Map pod/job to GPU and host failure domain.
3. Collect read-only Xid/kernel event timeline.
4. Interpret synthetic Xid 79 case and escalation.
5. Interpret synthetic ECC correctable/uncorrectable trends.
6. Interpret synthetic row-remapping/page-retirement state.
7. Interpret synthetic PCIe AER and link failures.
8. Interpret synthetic NVLink error and NCCL impact.
9. Interpret synthetic power/thermal throttling.
10. Compare affected GPU with healthy peer baseline.
11. Assess DCGM metric availability and alert semantics.
12. Build a severity and customer-impact matrix.
13. Create a redacted vendor-ready support bundle.
14. Tabletop GPU quarantine and checkpoint-safe recovery.
15. Validate staging return-to-service checklist and alert routing.
16. Produce RCA/COA evidence, reviewer sign-off and 100-point score.

## 11. Troubleshooting matrix

| Symptom | Investigate | Avoid |
|---|---|---|
| Xid 79 and GPU missing | PCIe, host, vendor hardware | auto reset/reboot |
| ECC counters increasing | generation-specific reliability guidance | disabling ECC |
| GPU healthy, CUDA fails | runtime/binary/permissions | RMA-first |
| Node Ready, no allocatable GPU | plugin/driver/MIG | replacing hardware blindly |
| NVLink errors and NCCL slowdown | interconnect, rank topology | network-only diagnosis |
| High temperature and low clocks | cooling/power policy | assuming fatal hardware fault |
| No DCGM metrics | exporter/config/labels | declaring GPU healthy |
| Recurring fault on same GPU UUID | hardware evidence across jobs | suppressing alerts |

## 12. Production runbooks

**A — Xid 79 / GPU fallen off bus:** preserve NVRM/AER and job timeline, protect checkpoints/replicas, coordinate node isolation and vendor diagnostics, restore only after acceptance.

**B — ECC / row-remap escalation:** compare volatile/aggregate counters and remapping state, apply model-specific vendor guidance, track replacement or reboot requirement under change control.

**C — PCIe/NVLink fault:** correlate GPU topology, AER and interconnect errors with NCCL impact, engage host/fabric owner, avoid disruptive tests on shared workloads.

**D — Thermal/power incident:** inspect temperature, throttle reason, power cap, cooling and node placement; remediate infrastructure conditions; validate stable clocks and workload SLO.

**E — Vendor support and return to service:** assemble redacted evidence, assign owner/SLA, document repair action, run CUDA/NCCL checks, confirm allocatable and customer health.

## 13. Acceptance gate (100 points)

| Gate | Points |
|---|---:|
| GPU failure-domain classification | 20 |
| Xid/ECC/PCIe/NVLink diagnostic evidence | 25 |
| Severity, containment and checkpoint safety | 15 |
| Vendor escalation and support bundle | 15 |
| Return-to-service staging validation | 15 |
| Runbooks, security and independent review | 10 |
| **Total** | **100** |

**Pass:** ≥85/100 with no critical data-integrity, tenant privacy, unsafe reset or unapproved production impact. Written documentation alone does not satisfy lab or review gates.

## 14. References

- [NVIDIA Xid Errors](https://docs.nvidia.com/deploy/xid-errors/)
- [NVIDIA System Management Interface](https://docs.nvidia.com/deploy/nvidia-smi/)
- [NVIDIA DCGM](https://docs.nvidia.com/datacenter/dcgm/latest/)
- [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/)
- [Kubernetes Node Status](https://kubernetes.io/docs/reference/node/node-status/)
- [NVIDIA MIG User Guide](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/)

**Next:** Chapter 081 — Recovery Runbooks and RCA COA.
