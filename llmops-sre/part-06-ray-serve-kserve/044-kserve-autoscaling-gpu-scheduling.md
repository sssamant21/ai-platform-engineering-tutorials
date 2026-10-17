# Chapter 044 — KServe Autoscaling and GPU Scheduling

**Part 06 — Ray Serve and KServe | Chapter 044 of 100**

> **Status:** Written tutorial published; live lab execution, independent technical/security review and operational acceptance remain pending. KServe autoscaling mechanisms depend on serving mode, version and installed controllers. Verify live CRD schemas and deployment topology.

## Objectives

Design KServe replica autoscaling and GPU node provisioning; distinguish Standard and Knative-based serving; calculate safe GPU capacity, cold-start and queue budgets; troubleshoot Pending GPU workloads, placement and device plugins; protect SLOs, security and GPU FinOps; validate rollback.

## 1. Four-layer scaling architecture

```text
Client demand → Gateway admission, token quotas, deadlines
       │
KServe serving-mode-specific autoscaler
       │ desired predictor replicas
       ▼
Kubernetes scheduler / device plugin
       │ GPU requests, limits, taints, affinity
       ▼
Node provisioning: Cluster Autoscaler / Karpenter / cloud service
       │ GPU node availability, cloud quota
       ▼
Model image/artifact fetch → CUDA init → model warmup
       │
Ready predictor → synthetic terminal completion → SLO
```

Scaling replicas is not the same as obtaining GPU nodes, and a ready Pod does not prove that a model produces valid output.

## 2. Standard versus Knative-based autoscaling

| Concern | Standard mode | Knative-based mode |
|---|---|---|
| scaling controller | version/configuration dependent Kubernetes HPA or other supported integration | Knative Pod Autoscaler/KPA or supported HPA integration |
| request concurrency | runtime/ingress dependent | Knative concurrency settings when applicable |
| scale to zero | not assumed | possible if enabled and supported |
| cold-start path | Pod/image/model warmup | includes activation/routing path |
| traffic routing | Kubernetes ingress/gateway | Knative revision/routing capabilities |

Do not assume Knative CRDs or specific KServe annotation names are available. Confirm installed KServe, Knative, metrics adapter, HPA and gateway versions before applying any policy.

## 3. Safe-read cluster inventory

```bash
date -u
kubectl config current-context
kubectl get inferenceservices -A
kubectl get hpa -A
kubectl get pods -A -o wide
kubectl get nodes -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -100
kubectl get crd | grep -Ei 'kserve|knative'
```

For authorized namespace:

```bash
kubectl -n APPROVED_NS describe inferenceservice APPROVED_MODEL
kubectl -n APPROVED_NS get deploy,hpa,pods -o wide
kubectl -n APPROVED_NS describe pod APPROVED_POD
```

Missing CRDs or HPAs are observations, not instructions to install controllers.

## 4. GPU requests and limits

GPU resources such as `nvidia.com/gpu` are extended resources exposed by device plugins. On typical Kubernetes GPU scheduling, request and limit semantics are constrained by the device plugin and Kubernetes resource rules. Verify allocatable GPU devices, runtime class, driver compatibility and actual VRAM independently.

```bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,GPU:.status.allocatable.nvidia\\.com/gpu
kubectl describe node APPROVED_GPU_NODE
```

If the custom-column JSONPath is unsupported in a given kubectl version, inspect `kubectl describe node` under Capacity/Allocatable instead.

## 5. Placement and topology

Node selectors, taints/tolerations, affinity, topology spread, GPU SKU labels, MIG profiles and multi-GPU tensor parallelism affect feasibility. A cluster with four free GPUs across four nodes may not schedule a four-GPU single-Pod inference replica. Match model parallelism and network topology to actual device placement.

## 6. VRAM sizing and capacity planning

```text
VRAM per replica ≈ weights + KV cache + activations
                  + runtime workspace + CUDA graphs
                  + communication buffers + safety headroom
```

Quantify prompt length, output length, active token budget, batch/concurrency and GPU memory fragmentation. Logical GPU allocation does not reserve sufficient VRAM automatically.

## 7. Cold-start and warm capacity

Model cold start includes image pull, artifact download, storage/KMS access, tokenizer load, CUDA initialization, weights loading and warmup. If cold-start time exceeds the request deadline, scaling after a burst cannot save those requests. Maintain approved warm capacity or use bounded queue/rejection policies.

## 8. Autoscaling signals

| Signal | Strength | Risk |
|---|---|---|
| CPU utilization | useful for CPU preprocessing | weak proxy for GPU decode |
| request concurrency | reflects active load | long/short requests differ |
| request queue depth | overload indicator | queue can explode before nodes ready |
| TTFT/TPOT | direct experience | lagging signal |
| GPU utilization | saturation clue | not equal to goodput |
| GPU VRAM | safety guard | not a direct throughput target |
| valid completion rate | SLO quality | may lag |
| GPU-hours/valid completion | FinOps | delayed cost signal |

Use the metrics and scaling adapter supported by the installed serving mode. Never invent metric names or assume native autoscaling on GPU utilization.

## 9. Backpressure and queue budgets

Bound pending requests and per-tenant concurrency at the gateway/runtime. Set token caps, timeouts, cancellation, retries with jitter and explicit overload responses. Model request deadlines against cold-start lead time; unbounded retries can make an autoscaling shortage worse.

## 10. Scale-up control loop

```text
Load ↑ → autoscaler decides → desired replicas ↑
 → scheduler checks GPU resources
 → node autoscaler provisions GPU node (if needed)
 → driver/device plugin readiness
 → image/model fetch → CUDA warmup
 → predictor readiness → golden inference
 → backlog drains → SLO and cost validated
```

Measure each stage. Do not interpret `Pending` as one failure class; check `FailedScheduling`, quota, taints, topology, cloud capacity and image/storage separately.

## 11. Scale-down, streaming and graceful termination

Long-running LLM streams can be interrupted by downscaling. Validate termination grace, preStop/drain where supported, endpoint removal, gateway timeouts and terminal streaming completion. Scale-down savings must be weighed against warm-capacity requirements.

## 12. Autoscaling oscillation

Aggressive scale-up/down can repeatedly reload large models and waste GPU-hours. Tune stabilization/cooldown based on observed traffic, cold-start time and SLO; use a minimum warm pool for latency-critical inference. Validate the actual controller's supported parameters.

## 13. Cloud GPU capacity

For EKS/AKS/GKE, document instance families, GPU quota, node group limits, AZ/zone availability, reservations, storage bandwidth, driver images and provisioning time. A desired GPU Pod count does not guarantee cloud GPU supply.

## 14. Security and multi-tenancy

Limit autoscaler, controller and node-provisioner RBAC; enforce namespace ResourceQuota, Pod Security, signed images, workload identity, NetworkPolicies and tenant request/token quotas. Prevent untrusted users from requesting unlimited GPU replicas or expensive model downloads.

## 15. SLO and FinOps

Define terminal-validity availability, TTFT p95/p99, TPOT, E2E, queue delay, fairness, warmup time, ready/desired replicas, Pending GPU Pods, per-device VRAM and GPU-hours per valid completion. Track idle GPU nodes after scale-down and orphaned staging workloads.

## 16. Read-only triage workflow

```text
SLO regression?
 ├─ requests rejected → admission/quotas
 ├─ desired replicas unchanged → scaling signal/policy
 ├─ desired > ready → scheduler/Pod startup
 ├─ GPU Pod Pending → quota/taints/topology
 ├─ no GPU nodes → cloud quota/autoscaler
 ├─ model load slow → artifact/storage/CUDA
 ├─ CUDA OOM → VRAM/token budget
 ├─ stream truncated → drain/gateway
 ├─ scaling flaps → stabilization/cold starts
 └─ cost rises → idle GPUs/repeated warmup
```

## 17. Twenty-four production failure scenarios

| ID | Failure | Evidence |
|---|---|---|
| F01 | wrong serving-mode scaling assumption | CRD inventory |
| F02 | autoscaler controller absent | controller discovery |
| F03 | scaling metric missing | metrics API |
| F04 | stale scaling signal | scrape freshness |
| F05 | min replicas violate warm SLO | TTFT |
| F06 | max replicas exceed quota | ResourceQuota |
| F07 | desired replicas stuck | controller status |
| F08 | GPU Pod Pending | scheduler events |
| F09 | GPU node taint mismatch | Pod spec |
| F10 | topology/affinity impossible | node labels |
| F11 | device plugin missing | allocatable |
| F12 | GPU driver incompatible | startup logs |
| F13 | cloud GPU quota exhausted | node events |
| F14 | image pull slow/fails | Pod events |
| F15 | model artifact download timeout | startup logs |
| F16 | KMS/workload identity denied | IAM audit |
| F17 | CUDA VRAM OOM | GPU telemetry |
| F18 | long-context KV exhaustion | allocator |
| F19 | unbounded queue/retries | gateway |
| F20 | cold start breaches request deadline | timing |
| F21 | downscale truncates streams | terminal validator |
| F22 | scaling oscillates | replica timeline |
| F23 | rollback leaves wrong HPA/route | manifest diff |
| F24 | idle GPU-hours exceed ceiling | FinOps |

## 18. Sixteen operational runbooks

1. Confirm serving mode and scaling controller.
2. Recover missing metrics adapter or scaling signal.
3. Diagnose desired versus ready replicas.
4. Validate warm capacity and min/max bounds.
5. Recover Pending GPU Pods and scheduler failures.
6. Resolve taints, affinity, topology and MIG mismatches.
7. Restore GPU device plugin and driver readiness.
8. Recover cloud GPU node quota/provisioning.
9. Diagnose image pull and model artifact cold start.
10. Recover storage IAM/KMS and download errors.
11. Recover CUDA/KV memory pressure.
12. Enforce bounded queues and retry budgets.
13. Restore streaming drain and termination.
14. Tune autoscaling oscillation and stabilization.
15. Restore SLO telemetry, tenant security and GPU FinOps.
16. Execute complete autoscaler/InferenceService rollback and RCA.

## 19. Eight tabletop game days

1. HPA/KPA controller missing after upgrade.
2. GPU Pods remain Pending due to taint/topology.
3. Cloud GPU quota blocks burst scale-out.
4. Device plugin disappears from one node.
5. Model download doubles cold-start time.
6. Long-context traffic triggers KV OOM.
7. Scale-down interrupts active streams.
8. Oscillation doubles GPU-hours while p99 worsens.

Default tabletop; live fault injection requires approved isolated staging.

## 20. Twenty acceptance gates

G01 scope/version/mode; G02 autoscaler controller; G03 metric freshness; G04 min/max/warm pool; G05 desired/ready reconciliation; G06 GPU scheduling; G07 taints/topology; G08 device plugin/driver; G09 cloud quota; G10 image/model startup; G11 workload identity/KMS; G12 VRAM/KV headroom; G13 bounded queue/retry; G14 cold-start SLO; G15 streaming drain; G16 tenant/security; G17 telemetry/alerting; G18 full rollback; G19 GPU FinOps; G20 independent review.

See [companion lab](../labs/044-kserve-autoscaling-gpu-scheduling-check.md). All gates remain pending until independently evidenced.

**Next: Chapter 045 — Ray Serve and KServe Operations Project.**
