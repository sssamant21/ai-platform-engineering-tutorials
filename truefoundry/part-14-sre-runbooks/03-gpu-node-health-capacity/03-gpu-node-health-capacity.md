# Chapter 14.3 — GPU Node Health and Capacity Runbook

**Part 14 — TrueFoundry SRE Runbooks | Production operations**

## Mission and operational boundaries

Detect, classify and recover GPU node failures and inference capacity shortfalls on TrueFoundry-managed Kubernetes environments (EKS, AKS, GKE). Cover NVIDIA drivers, device plugin, GPU Operator where installed, MIG, DCGM telemetry, XID/ECC, scheduling, autoscaling, zonal GPU supply, reservation headroom, node drains and FinOps.

**[SAFE-READ]** approved node/Pod, Events and telemetry inspections. **[LAB-MUTATE]** isolated staging GPU pool with a fixed budget. **[CHANGE-CONTROL]** production node drains, GPU resets, driver upgrades, MIG reconfiguration, node pool scaling and quota changes. Never reboot or reset a GPU serving production inference without an approved maintenance plan.

## 1. GPU service dependency map

```text
Cloud GPU instance / GPU SKU / availability zone / quota
     ↓ kernel driver + GPU hardware + firmware
NVIDIA container toolkit / device plugin / GPU Operator (optional)
     ↓ nvidia.com/gpu or MIG resource advertisement
Kubernetes scheduler / taints / affinity / topology / autoscaler
     ↓ Pod device assignment and CUDA visibility
vLLM model load / KV cache / batching / TTFT / TPOT
     ↓ Gateway / synthetic streaming / customer SLO
DCGM exporter / Kubernetes metrics / alerts / on-call
```

Separate **GPU physical health**, **Kubernetes allocatable**, **requested capacity**, **actual workload utilization**, and **cloud provisionability**. These are not interchangeable.

## 2. First five minutes

1. Record UTC, cluster, region/AZ, affected node pool, GPU SKU, models, tenants and incident commander.
2. Validate synthetic inference availability, TTFT/TPOT and stream completion.
3. Inspect Pending GPU Pods, FailedScheduling Events, GPU allocatable and requested GPU resources.
4. Inspect node Ready condition, taints, pressure, device plugin and driver health.
5. Inspect DCGM GPU memory, utilization, XID/ECC, thermal/power and telemetry freshness.
6. Check autoscaler logs, pool limits, cloud quota and zone-specific GPU availability.
7. Determine if the failure is a single GPU/node, pool, zone, region or model configuration.
8. Choose an approved mitigation, capacity reserve and rollback; set next update time.

## 3. Read-only Kubernetes command pack

```bash
kubectl config current-context
kubectl get nodes -L topology.kubernetes.io/zone -o wide
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -80
kubectl describe node GPU_NODE
kubectl -n MODEL_NAMESPACE describe pod MODEL_POD
kubectl -n MODEL_NAMESPACE get deployment MODEL_DEPLOYMENT -o wide
kubectl get resourcequota -A
```

Inspect allocatable `nvidia.com/gpu` or installed MIG resource types and Pod requests/limits. The node's `Allocatable` is not remaining free GPU capacity; subtract scheduled requests for the relevant resource type. `kubectl top node` does not report GPU utilization.

## 4. NVIDIA diagnostics

Run only from an approved GPU-enabled host or diagnostic Pod where devices and utilities are present:

```bash
nvidia-smi
nvidia-smi --query-gpu=index,name,driver_version,memory.total,memory.used,utilization.gpu,temperature.gpu,power.draw --format=csv
```

Unavailable metrics or missing `nvidia-smi` in a minimal image do not independently prove a node hardware fault. Capture driver and device-plugin versions, CUDA runtime compatibility, NVIDIA XID, ECC and hardware events through vendor-supported diagnostics. Do not publish full GPU UUIDs if restricted.

## 5. GPU scheduling and allocatable capacity

Review taints/tolerations, affinity, topology spread, ResourceQuota, Pod requests, extended resources, MIG profile and node selectors. GPU extended resources are typically allocated in whole units for the advertised resource type; fractional sharing requires an explicitly configured mechanism. A GPU with free memory may still be unavailable for scheduling because its extended resource is allocated.

```bash
kubectl -n MODEL_NAMESPACE get pods -o wide
kubectl -n MODEL_NAMESPACE get events --sort-by=.lastTimestamp
kubectl describe node GPU_NODE
```

Diagnose `Insufficient nvidia.com/gpu` distinctly from unsupported GPU SKU, missing MIG resource, quota denial, taint mismatch or max node pool size.

## 6. Autoscaling and cloud capacity

Inspect the installed autoscaler (Cluster Autoscaler, Karpenter or cloud-specific), provisioning constraints, max size, IAM, instance type offerings, capacity reservations, subnet IPs, zone inventory and provider quota. A successful autoscaler scale-up decision does not guarantee an instance can be provisioned. Avoid broad instance-type fallback when model correctness or GPU architecture is constrained.

## 7. GPU health and memory pressure

| Symptom | First evidence | Next action |
|---|---|---|
| Node NotReady | kubelet/node Events | classify host/network/cloud |
| GPU allocatable zero | plugin, driver, node status | verify resource advertisement |
| XID/ECC errors | vendor diagnostics/DCGM | isolate device with change control |
| CUDA OOM | vLLM logs/GPU memory | model fit, KV cache, concurrency |
| Host OOMKilled | Pod last termination reason | container memory/loading peak |
| Thermal/power throttling | DCGM/driver metrics | hardware/cooling/instance investigation |
| GPU high utilization | TTFT/TPOT/queue and traffic mix | determine healthy load vs saturation |
| Low utilization, high latency | queue, CPU, network, model load | find non-GPU bottleneck |

Do not reset a device merely because utilization is high; utilization may be expected.

## 8. Capacity math and N-1 headroom

For homogeneous GPUs, define **schedulable GPU slots** as allocatable extended resources minus already requested resources. Define **usable inference replicas** based on GPU slots per model replica, node placement, GPU SKU and topology constraints.

Example: four nodes with one GPU each and four one-GPU replicas yield zero spare slots. Losing one node requires either three remaining replicas to sustain peak SLO or an approved warm spare / fast capacity provision. For a two-GPU-per-replica model, arithmetic must consider packing and node topology; aggregate free GPUs alone may be insufficient.

Set N-1/N-AZ objectives from measured throughput at p95 workload and error budget, not from GPU utilization percentages alone. Estimate cost per successfully served 1,000 output tokens and reserve GPU-hours; account for idle warm replicas, spot interruption and cold-start.

## 9. Safe mitigation hierarchy

1. Verify customer-path impact and freeze unrelated releases.
2. Identify unhealthy device/node without evicting healthy streams.
3. Route to approved healthy replicas while preserving tenancy/security.
4. Use platform-supported autoscaler/pool expansion within quota and budget.
5. Apply reversible workload concurrency/context adjustments only after staging verification.
6. Cordon/drain/replace node only with PDB, streaming grace, model warmup, capacity and change approval.
7. Roll back driver/plugin/GPU Operator changes via tested versioned plan.
8. Validate synthetic SLO, GPU allocatable, queue and alert recovery.

Never delete all GPU Pods, mass-drain nodes or use `nvidia-smi --gpu-reset` on shared production hardware.

## 10. Fourteen operational runbooks

1. GPU node inventory and hardware baseline.
2. Node NotReady and kubelet failure.
3. Driver mismatch or NVIDIA device missing.
4. Device plugin/GPU Operator health.
5. MIG profile or resource mismatch.
6. Pending GPU Pod and quota.
7. Autoscaler or cloud capacity provisioning.
8. XID/ECC and hardware escalation.
9. Thermal/power throttling.
10. CUDA OOM vs host OOM.
11. vLLM queue/TTFT/TPOT capacity.
12. N-1/AZ failover capacity.
13. Change-controlled node drain and rollback.
14. Post-recovery SLO and FinOps validation.

## 11. Forty failure scenarios

| # | Scenario | # | Scenario |
|---|---|---|---|
| 01 | Node NotReady | 21 | Host OOMKilled |
| 02 | Kubelet unhealthy | 22 | CUDA OOM |
| 03 | GPU missing on host | 23 | KV cache exhaustion |
| 04 | Driver incompatible | 24 | GPU memory fragmentation |
| 05 | Device plugin CrashLoop | 25 | High TTFT |
| 06 | Allocatable GPU zero | 26 | High TPOT |
| 07 | MIG profile mismatch | 27 | Queue saturation |
| 08 | Time-slicing policy mismatch | 28 | Cold model warmup |
| 09 | Taint mismatch | 29 | GPU high utilization but healthy |
| 10 | Node affinity mismatch | 30 | Low GPU utilization but slow |
| 11 | Topology spread blocked | 31 | PDB blocks drain |
| 12 | ResourceQuota exhausted | 32 | Stream interrupted on drain |
| 13 | Insufficient GPU slots | 33 | Node replacement slow |
| 14 | Cloud GPU quota reached | 34 | Zone GPU shortage |
| 15 | Autoscaler max reached | 35 | Spot interruption |
| 16 | Provisioning IAM denied | 36 | Subnet IP exhausted |
| 17 | XID error | 37 | N-1 capacity shortfall |
| 18 | ECC error | 38 | AZ loss |
| 19 | Thermal throttling | 39 | DCGM exporter stale |
| 20 | Power throttling | 40 | GPU cost/token spike |

## 12. Twelve acceptance gates

G1 approved scope/spend; G2 GPU/node inventory; G3 driver/plugin/device health; G4 schedulable GPU resources; G5 quotas/autoscaler/cloud capacity; G6 XID/ECC/thermal signals; G7 host vs CUDA OOM; G8 model SLO/TTFT/TPOT; G9 N-1 and topology; G10 safe drain/rollback; G11 synthetic recovery and cost; G12 reviewer sign-off.

See the [hands-on lab](../labs/03-gpu-node-health-capacity-check.md). Formal technical/security review and live execution pending.

**Next:** Chapter 14.4 — Model Deployment and Rollback Runbook.
