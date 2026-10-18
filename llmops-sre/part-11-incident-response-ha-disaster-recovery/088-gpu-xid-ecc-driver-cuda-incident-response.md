# Chapter 088 — GPU XID, ECC, Driver and CUDA Incident Response

**Part 11 — Incident Response, HA and Disaster Recovery · Chapter 088 of 100**  
**Status:** Chapter and executable offline lab published. Live GPU fault drills, vendor-specific diagnostic validation and independent operational review pending.

## Learning objectives
Diagnose GPU inference incidents involving NVIDIA Xid events, ECC errors, driver/runtime incompatibility, CUDA failures, GPU memory exhaustion and device-plugin scheduling faults. Safely cordon, drain, recover and validate accelerator nodes without violating availability or cost guardrails.

## 1. Reference architecture
```text
AI gateway -> inference router -> Kubernetes scheduler
                                    |
                           GPU worker node
                             /     |      \
                    driver/CUDA  device plugin  GPU model pod
                             \     |      /
                          DCGM / NVML / dmesg
                                    |
                    metrics + logs + alert manager
                                    |
                       incident command / ticket
                                    |
                isolate -> diagnose -> remediate -> validate
```
Collect Xid and ECC evidence from host kernel logs, DCGM/NVML and GPU operator diagnostics. GPU model, driver, firmware, MIG mode and Kubernetes runtime affect both interpretation and remediation.

## 2. Failure taxonomy
| Signal | Likely causes | Initial checks | Response |
|---|---|---|---|
| Xid event | GPU hardware, driver, PCIe, application fault | Xid code, UUID, timestamp, process, vendor guidance | isolate affected GPU and escalate based on code |
| ECC correctable increase | memory error correction activity | per-device trend, thresholds, retired pages | monitor and consult hardware policy |
| ECC uncorrectable | unrecoverable memory fault | DCGM/NVML, node health, error recurrence | remove from service, vendor escalation |
| CUDA out of memory | KV cache pressure, batch/concurrency, fragmentation | allocated/reserved memory, model revision, batch | reduce admission and restore capacity |
| CUDA unavailable | driver/runtime mismatch, device not exposed | `nvidia-smi`, CUDA driver compatibility, container runtime | correct stack and plugin |
| GPU missing in pod | scheduling, plugin, taint, resource request | node allocatable, plugin pods, events | restore device discovery |
| GPU reset/restart loop | persistent fault, stale process, operator churn | kernel logs, workload logs, node events | controlled node isolation |
| performance degradation | thermal/power throttling, memory bandwidth, PCIe | DCGM clocks, power, temperature, utilization | inspect hardware and load |

**Xid code interpretation is model/driver-specific.** Do not label all Xid events as fatal or prescribe automatic GPU reset solely from the numeric code. Use the NVIDIA Xid documentation and vendor support criteria.

## 3. First 10-minute triage
1. Confirm inference impact: 5xx, TTFT/TPOT, queue age, p95/p99, available GPU replicas.
2. Identify GPU node, UUID, workload, model revision and last change.
3. Inspect kernel events, Xid codes, ECC counters and GPU health; use UTC timestamps.
4. Compare node driver, CUDA runtime, GPU operator, container toolkit and device plugin versions.
5. Check GPU memory, MIG partitions, allocations, pod events and device requests.
6. Verify remaining healthy capacity before draining or evicting.
7. Coordinate with incident commander and on-call hardware/platform owner.
8. Contain only affected node/GPU and preserve sanitized diagnostics.

## 4. Production diagnostics
```bash
# Host/node diagnostics; run with approved access.
nvidia-smi -L
nvidia-smi --query-gpu=index,uuid,name,temperature.gpu,utilization.gpu,memory.used,memory.total --format=csv
nvidia-smi -q -d ECC
journalctl -k --since "30 minutes ago" | grep -Ei 'NVRM|Xid|ECC|PCIe'

# Kubernetes scheduling/device-plugin checks
kubectl get nodes -o wide
kubectl describe node <gpu-node>
kubectl get pods -A -o wide | grep -i nvidia
kubectl get events -A --sort-by=.lastTimestamp | tail -n 40
```
These commands are illustrative. Confirm vendor/driver support for `nvidia-smi` ECC queries and host log access; not all GPU models expose identical counters. Avoid copying full host logs containing sensitive data into public tickets.

## 5. Safe node containment and recovery
```bash
kubectl cordon <gpu-node>
kubectl get pods -A -o wide --field-selector spec.nodeName=<gpu-node>
# After checking disruption budgets, replacement GPU capacity and owner approval:
kubectl drain <gpu-node> --ignore-daemonsets --delete-emptydir-data
```
**Warning:** `drain` can evict stateful or long-running inference workloads and may delete ephemeral data. Confirm PodDisruptionBudgets, replica placement, pending GPU capacity, autoscaler behavior, model cold-start time and customer impact first. Do not use `--force` casually. GPU reset and node reboot require hardware/runbook authorization; a reset can terminate every workload using the device.

After remediation, validate device health, scheduling, GPU allocation, synthetic inference, latency, throughput, ECC/Xid recurrence and cost burn. Uncordon only after explicit readiness checks.

## 6. CUDA/driver compatibility
Check host driver version against the CUDA runtime/toolkit requirements of the application image. The CUDA runtime inside a container does not replace the host kernel driver. Validate NVIDIA Container Toolkit, Kubernetes device plugin, GPU operator and GPU partitioning. Avoid untested driver upgrades during a high-severity incident; use known-good image/node pools and staged rollout.

## 7. Alerting and SLOs
Monitor Xid occurrence by GPU UUID and code, ECC correctable/uncorrectable deltas, GPU availability, memory usage, OOM kills, GPU reset count, DCGM health, thermal/power throttling, inference queue age and p95/TTFT. Alert on sustained customer impact and hardware fault conditions. Treat high-cardinality GPU UUIDs carefully in shared telemetry.

## 8. Incident runbooks

### RB-088-A — Xid/ECC fault
Record device UUID, Xid/ECC evidence and workload impact. Cordon affected node if needed, preserve diagnostics, compare vendor fault guidance, escalate hardware support, and restore service on healthy capacity before repair.

### RB-088-B — CUDA OOM / KV-cache pressure
Inspect input/output token distributions, batch and concurrency, memory fragmentation and model revision. Reduce admission, restore known-good serving profile and validate p95 and completed requests.

### RB-088-C — Driver/runtime mismatch
Compare node image, driver, container toolkit, runtime and application CUDA requirements. Revert to known-good node pool or image, validate `nvidia-smi` and device discovery, then run synthetic inference.

### RB-088-D — GPU device plugin failure
Inspect DaemonSet health, allocatable `nvidia.com/gpu`, node taints/tolerations and pod events. Restore plugin/operator version under change control, then test scheduling.

### RB-088-E — Hardware replacement and return to service
Drain with capacity and PDB validation, replace or repair node under vendor guidance, verify burn-in/health, reintroduce via canary and close incident with RCA/COA.

## 9. Failure-injection game day
1. Synthetic Xid warning and repeated fault escalation.
2. Correctable ECC growth vs uncorrectable ECC alert.
3. CUDA OOM from long-context burst.
4. Missing `nvidia.com/gpu` allocatable resource.
5. Driver/CUDA mismatch after node image rollout.
6. GPU device-plugin crash.
7. Node drain with insufficient healthy GPU capacity.
8. Hardware replacement and N-1 recovery.
9. Thermal throttling causing TTFT regression.
10. Persistent fault after restart requiring vendor escalation.

## 10. 100-point assessment
| Area | Points |
|---|---:|
| GPU fault classification and evidence | 20 |
| Driver/CUDA/runtime and scheduling diagnostics | 20 |
| Safe isolation, drain and capacity planning | 20 |
| SLO recovery, monitoring and escalation | 20 |
| Automated tests, runbooks and game-day evidence | 20 |
| **Total** | **100** |

Critical blockers: unsafe GPU reset, unapproved destructive drain, cross-tenant impact, persistent uncorrectable ECC or return-to-service without health validation. Educational target >=90/100. Production readiness requires live hardware-specific validation.

## 11. Companion lab
[Chapter 088 — Executable GPU XID, ECC, Driver and CUDA Incident Lab](../labs/088-gpu-xid-ecc-driver-cuda-incident-response-check.md). This offline simulator evaluates synthetic signals and recovery gates. It does not query real GPUs or classify specific Xid codes.

## References
- [NVIDIA Xid Errors](https://docs.nvidia.com/deploy/xid-errors/index.html)
- [NVIDIA DCGM](https://docs.nvidia.com/datacenter/dcgm/latest/)
- [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/)
- [Kubernetes Device Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)
- [Kubernetes Node Drain](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/)

**Next:** Chapter 089 — Model Quality and Release Regression Incidents.
