# Chapter 088 — Scaling and Queue Admission Control

**Part 11 — Capacity, FinOps and Reliability | AI Infrastructure Engineering & SRE**

**Publication status:** Published documentation only. Lab execution, GPU staging tests, independent security/technical review and production acceptance remain **PENDING**.

## 1. Mission

Engineer safe GPU autoscaling and workload admission for Kubernetes AI platforms. Combine online inference SLO protection, distributed training queue fairness, cloud GPU provisioning, quota governance, topology-aware placement and cost limits. Ensure scaling actions never silently degrade model quality, training recoverability, tenant isolation or customer latency.

**Prerequisites:** Chapters 083–087; Kubernetes scheduling, HPA/KEDA, Cluster Autoscaler or Karpenter, Kueue/Volcano, GPU Operator, DCGM/Prometheus, inference serving, training checkpointing and incident management.

### Learning outcomes

1. Distinguish **serving replica scaling**, **GPU node provisioning**, **queue admission** and **application concurrency limits**.
2. Model burst arrival, pending demand, capacity lead time, cold starts and scaling hysteresis.
3. Implement priority-aware admission with protected inference headroom.
4. Prevent GPU SKU/MIG/fabric fragmentation and unsafe distributed training preemption.
5. Define overload behavior, budgets, alerting, rollback and measurable acceptance gates.

## 2. Control-loop architecture

\`\`\`text
Customer requests → ingress/rate limits → serving queue → inference pods
                        │                   │
                        │              SLO metrics
                        │                   ↓
                        └────────→ HPA/KEDA/custom scaler
                                            ↓
Training job submissions → Kueue/Volcano admission → pending pods
                             │                      │
                       quota/fair share              ↓
                             └────────────→ node autoscaler/provisioner
                                                   ↓
                                     cloud GPU quota + physical supply
                                                   ↓
                                     compatible nodes/SKUs/topology
                                                   ↓
                                         scheduled GPU workloads
\`\`\`

**Four independent loops:** application admission controls incoming work; batch admission decides which training jobs can run; pod autoscaling changes replicas; node provisioning adds compatible physical capacity. An admitted workload can remain Pending if no suitable nodes or cloud supply exist.

## 3. Workload classes and service contracts

| Class | Scaling objective | Hard guardrail |
|---|---|---|
| P0 customer inference | preserve p99, error rate and TTFT/TPOT | reserved compatible capacity and HA |
| P1 deadline training | meet deadline and checkpoint RPO | gang placement and recovery |
| P2 routine training | throughput and fair queue wait | quotas and approved borrowing |
| P3 interruptible research | opportunistic utilization | checkpoint-aware reclaim |
| Batch inference | meet completion window | bounded queue and data protection |

A GPU utilization target is **not** a service SLO. Measure SLO-compliant throughput under realistic model context lengths, concurrency, cold starts and zone failures.

## 4. Scaling signal selection

### Online inference

Prefer demand and service metrics such as:
- pending requests, queue age and arrival rate;
- generated tokens/sec and prompt tokens/sec;
- p95/p99 end-to-end latency, TTFT and TPOT;
- per-replica **measured** sustainable throughput at the latency SLO;
- HBM/KV-cache headroom, active sequences and model load time;
- ready replicas by zone and GPU SKU.

**Avoid:** scaling solely on GPU SM utilization or memory occupancy; these are diagnostic signals, not sufficient capacity controls.

### Training and batch

Use queue backlog **in GPU-hours**, earliest deadlines, requested GPU topology, priority and historical runtimes. Count admitted-but-Pending jobs separately from unadmitted jobs to avoid double-counting scale demand.

## 5. Capacity math

### 5.1 Inference replica requirement

\[
R_{\mathrm{base}}=\left\lceil\frac{\lambda_{\mathrm{peak}}}{\mu_{\mathrm{SLO}}}\right\rceil
\]

where \`lambda_peak\` is peak eligible requests/sec and \`mu_SLO\` is **measured sustainable per-replica throughput at the required latency SLO**. Add redundancy and rollout/cold-start buffers separately. Throughput is not constant across prompt/output lengths.

### 5.2 Synthetic example

Peak = **900 requests/sec**. A staging benchmark validates **150 SLO-compliant requests/sec per GPU-backed replica**.

\[
R_{\mathrm{base}}=\lceil900/150\rceil=6
\]

If one node hosts two replicas and the requirement is to survive loss of one such node at peak, at least **8 ready compatible replicas** are needed in this simplified model (six survivors). Zone-loss planning may require more, depending on placement and traffic. This is a hypothetical benchmark, not an observed result.

### 5.3 Training GPU-hour backlog

\[
H_{\mathrm{backlog}}=\sum_j G_jT_j
\]

Backlog GPU-hours do not establish gang schedulability. Model peak simultaneous GPU count, whole-node bundles, GPU memory, network topology and deadline constraints.

### 5.4 Scale-up lead time

\[
T_{\mathrm{ready}}=T_{\mathrm{quota}}+T_{\mathrm{provision}}+T_{\mathrm{bootstrap}}+T_{\mathrm{image/model\,load}}+T_{\mathrm{warmup}}
\]

This additive approximation may overestimate if stages overlap; instrument actual end-to-end readiness. Capacity must be reserved or prewarmed when bursts arrive faster than scale-up.

## 6. Autoscaling controls

### 6.1 HPA/KEDA

HPA can scale Deployments using resource/custom/external metrics. KEDA can supply external metrics based on event-driven signals. GPU metrics require compatible adapters/integration; do not assume DCGM metrics automatically work with HPA.

**Illustrative HPA using a custom per-pod metric:**

\`\`\`yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: inference-queue-hpa
  namespace: ai-serving
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: inference
  minReplicas: 4
  maxReplicas: 16
  metrics:
    - type: Pods
      pods:
        metric:
          name: pending_requests_per_pod
        target:
          type: AverageValue
          averageValue: "20"
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 600
    scaleUp:
      stabilizationWindowSeconds: 0
\`\`\`

**Not directly deployable without** a real Deployment, registered custom metrics adapter, metric definition, benchmarked target, GPU capacity and staging validation. The \`20\` threshold is illustrative, not a recommended universal value.

### 6.2 GPU node provisioning

Configure GPU-specific node pools/NodePools and compatible instance types, driver/operator readiness, cloud quota and regional supply. Karpenter and Cluster Autoscaler behavior is version/provider-specific. A Pending pod can trigger node scale-up only when the provisioner can create a compatible node.

### 6.3 Hysteresis and cooldown

Use stabilization windows, scale-down delays, minimum warm replicas and per-interval scale limits to avoid oscillation. Coordinate pod scaling with node consolidation so expensive model loads do not repeat unnecessarily.

## 7. Kueue and queue admission

Use ClusterQueues, LocalQueues, ResourceFlavors and cohorts where supported to express GPU SKU quotas, tenant floors, borrowing and priority. Validate installed CRDs and versioned APIs before writing manifests.

**Conceptual policy:**

\`\`\`text
protected-serving: reserved node pool, no research borrowing
deadline-training: guaranteed 32 GPUs, controlled borrowing
research: guaranteed 8 GPUs, burst to 16 if spare
global admission: block when no compatible GPU bundle exists
\`\`\`

Protect serving capacity with **separate scheduling and governance controls**; Kueue admission alone does not enforce inference SLOs. Multi-node training needs gang-aware placement and checkpoint-safe recovery. Avoid automatic eviction of stateful jobs without an approved preemption policy.

## 8. Backpressure and overload

When demand exceeds feasible capacity:
1. Preserve P0 serving minimums and protect error budget.
2. Apply bounded request queues, rate limits and documented rejection behavior (e.g., HTTP 429 with retry guidance where appropriate).
3. Prioritize eligible requests by contract; never let untrusted callers self-select high priority.
4. Reduce batch admission, not customer reliability headroom.
5. Use bounded exponential backoff with jitter to avoid retry storms.
6. Escalate capacity quota/physical supply shortages and communicate ETA only when grounded.
7. Preserve training checkpoints and audit admission/preemption decisions.

An infinite queue is not a resilience strategy: it increases tail latency, memory pressure and timeout amplification.

## 9. Scale-in safety

Before reducing replicas/nodes:
- confirm real p99/throughput headroom and multi-zone survivability;
- drain inference connections and stop routing before termination;
- respect startup/readiness probes, graceful shutdown and PDB limitations;
- checkpoint training and verify durable restore point;
- validate queue fairness and no deadline misses;
- protect GPU device-plugin/driver readiness and model caches;
- enforce minimum node/replica floors and rollback triggers.

Scale-in is a **reliability-sensitive change**, even if it lowers GPU-hours.

## 10. Security and FinOps controls

- Restrict who can change priority classes, queue quotas, node provisioning policies and custom metrics.
- Authenticate metric sources and defend against adversarial traffic-induced scaling.
- Cap cost by tenant/budget without cutting protected inference capacity.
- Distinguish reserved GPU-hours, allocated GPU-hours, useful work and actual billed savings.
- Record decisions with workload UID, owner, quota, priority, scaling cause and approval.
- Keep customer identifiers out of high-cardinality metrics and public dashboards.

## 11. Six production runbooks

**A — Inference overload:** verify queue age, p99 and model readiness; protect minimum replicas, rate-limit safely, scale within compatible capacity, rollback on SLO regression.

**B — Pending GPU pods despite autoscaling:** inspect scheduler events, SKU/MIG requests, taints, CPU/RAM, cloud quota and regional stock; fix the actual blocker.

**C — Training queue starvation:** inspect queue priority, cohort borrowing, gang requirements and long-running jobs; correct fairness without unsafe preemption.

**D — Scale oscillation:** compare arrival bursts, cooldown, custom metric lag, model warmup and node consolidation; tune stabilization under staging load.

**E — GPU quota/supply exhaustion:** separate cloud quota, reservations and physical supply; protect P0, reject/queue lower-priority work and escalate.

**F — Unsafe scale-in:** freeze consolidation, restore replicas/nodes, verify checkpoint integrity, monitor SLO and file change corrective action.

## 12. Sixteen hands-on exercises

1. Inventory GPU node pools and workload priorities.
2. Build demand metrics for inference and training queues.
3. Calculate SLO-based serving replica floor.
4. Calculate N-1 replica headroom.
5. Calculate GPU-hour backlog and gang concurrency.
6. Measure synthetic node provisioning/cold-start lead time.
7. Inspect HPA/KEDA metric availability.
8. Validate illustrative HPA manifest in offline schema review.
9. Inspect GPU provisioner and cloud quota constraints.
10. Model Kueue ResourceFlavors and cohort borrowing.
11. Simulate quota exhaustion and Pending diagnosis.
12. Simulate inference overload and backpressure.
13. Simulate queue starvation and safe prioritization.
14. Evaluate scale oscillation and cooldown.
15. Design staging scale-in rollback and cost controls.
16. Produce acceptance evidence and independent review.

## 13. Twelve troubleshooting scenarios

| Symptom | Investigation |
|---|---|
| HPA increases replicas, pods Pending | GPU supply and scheduling |
| Node autoscaler idle | incompatible pod requirements |
| p99 rises while GPU util low | queue, CPU, KV cache, model load |
| Serving replicas oscillate | metric lag and cooldown |
| Training queue grows despite idle GPUs | gang topology/MIG |
| Cloud quota approved, nodes unavailable | physical regional stock |
| Kueue admits, Kubernetes cannot place | admission vs scheduling |
| Batch work starves serving | missing protected pool |
| Scale-in interrupts checkpoint | unsafe termination |
| Large retry storm | backpressure/jitter |
| More GPUs but same throughput | upstream bottleneck |
| Invoice unchanged after scale-down | commitments and billing |

## 14. Acceptance gate (100 points)

| Gate | Points |
|---|---:|
| Workload demand and metric quality | 20 |
| SLO-based inference scaling and HA | 20 |
| GPU provisioning and scheduling feasibility | 20 |
| Queue fairness, admission and overload | 20 |
| Safe scale-in, FinOps and rollback | 10 |
| Runbooks, evidence and independent review | 10 |
| **Total** | **100** |

**Pass:** ≥85/100 with no critical SLO violation, unauthorized preemption, quota bypass, model quality regression, checkpoint loss or missing rollback. Published documentation does **not** count as executed scaling tests.

## 15. References

- [Kubernetes HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [Kubernetes Scheduling](https://kubernetes.io/docs/concepts/scheduling-eviction/)
- [Kueue](https://kueue.sigs.k8s.io/)
- [KEDA](https://keda.sh/)
- [Karpenter](https://karpenter.sh/)
- [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/)
- [Google SRE — Handling Overload](https://sre.google/sre-book/handling-overload/)

**Next:** Chapter 089 — GPU Platform DR and Continuity.
