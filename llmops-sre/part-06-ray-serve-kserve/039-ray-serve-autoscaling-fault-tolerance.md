# Chapter 039 — Ray Serve Autoscaling and Fault Tolerance

**Part 06 — Ray Serve and KServe | Chapter 039 of 100**

> **Status:** Written tutorial published. Live labs, technical/security review and operational acceptance remain pending. Ray Serve autoscaling APIs, KubeRay CRDs, recovery semantics and metrics are version-specific; pin and validate before use.

## Learning objectives

Design Ray Serve replica autoscaling, KubeRay worker-node capacity, GPU resource reservations, admission control, cold-start budgets, availability, actor recovery, request draining, deployment rollouts, failure injection and rollback. Connect reliability SLOs to GPU FinOps.

## 1. Three distinct scaling layers

```text
Traffic / client demand
       │
Gateway quotas + backpressure
       │
Ray Serve deployment autoscaler
       │ desired replicas and concurrency targets
       ▼
Ray Core scheduler
       │ pending actor resource reservations
       ▼
KubeRay worker-group autoscaling
       │ worker Pods and Kubernetes resources
       ▼
Kubernetes node autoscaler / GPU node provisioning
       │
Actual GPU capacity + model startup + readiness
```

These layers operate on different time scales. Increasing Serve replicas does not instantly add GPUs, and adding GPU nodes does not guarantee a warmed model.

## 2. Autoscaling signals

Depending on Ray version, Serve can scale based on request load and target ongoing requests per replica. Record the actual metric, sampling interval, aggregation and smoothing semantics from the pinned release.

| Signal | Useful for | Limitation |
|---|---|---|
| ongoing requests/replica | replica load | length distribution matters |
| queued requests | overload detection | backlog can grow faster than capacity |
| TTFT p95/p99 | user experience | delayed tail signal |
| valid completion rate | quality/SLO | not a direct replica target |
| GPU VRAM | admission safety | memory not captured by logical GPU |
| GPU utilization | compute saturation | not equivalent to goodput |
| cold-start duration | scaling lead time | can vary by artifact size |
| GPU-hours per valid completion | FinOps | delayed accounting |

## 3. Core autoscaling parameters

Common concepts include minimum/maximum replicas, target ongoing requests, upscale/downscale delays, lookback windows, cooldown, startup timeout and queue limits. **Exact Ray Serve configuration names and defaults vary by release.** Validate via installed schema and documented examples; never copy an old manifest unchanged.

```yaml
# DESIGN WORKSHEET — NOT A DEPLOYABLE SERVE CONFIG
autoscaling_policy:
  min_replicas: APPROVED_WARM_CAPACITY
  max_replicas: APPROVED_GPU_QUOTA
  target_ongoing_requests_per_replica: APPROVED_LOAD_TARGET
  upscale_delay: VERSION_VALIDATED
  downscale_delay: VERSION_VALIDATED
  queue_admission_limit: APPROVED_BOUND
  gpu_hour_budget: APPROVED_CEILING
```

## 4. Warm capacity and cold starts

Model replicas may spend time pulling images, downloading weights, initializing CUDA, allocating KV, compiling kernels and warming up. Cold-start latency can exceed interactive SLOs. Maintain sufficient warm replicas for critical cohorts, and treat scale-to-zero as an explicit tested design decision rather than a default.

## 5. GPU capacity equation

```text
maximum_safe_replicas =
 min(
   Kubernetes GPU allocation / GPUs per replica,
   Ray logical GPU reservation / GPUs per replica,
   VRAM-safe replicas,
   topology constraints,
   namespace quota,
   approved cost ceiling
 )
```

Do not rely on the logical GPU count alone. Account for weights, activations, CUDA graphs, KV cache, workspace, communication buffers, fragmentation and safety headroom per GPU.

## 6. Queueing and backpressure

Autoscaling is a delayed control loop. Bounded queues, per-tenant quotas, request deadlines, retry budgets and explicit overload rejection are required even when scaling is enabled. Unbounded retries during a GPU shortage can cause positive feedback and cascading failure.

## 7. Scale-up sequence

```text
Demand increases
 → Serve desired replicas rises
 → Ray actors request resources
 → worker-group capacity expands (if enabled)
 → Kubernetes GPU Pods schedule
 → GPU node provisioned if needed
 → model weights loaded and warmed
 → replica ready
 → queue drains
 → SLO and GPU-hours validated
```

Instrument each stage and compare total time against allowed queue/deadline budgets.

## 8. Scale-down and draining

Scale-down can interrupt long-running generation if in-flight requests are not drained correctly. Verify version-specific graceful shutdown and replica draining behavior, client cancellation, terminal streaming completion and model resource cleanup. Test both short and long output cohorts.

## 9. Fault-tolerance boundaries

| Failure | Expected responsibility |
|---|---|
| single replica crash | Serve/Ray restart/replacement policy |
| Ray worker loss | Ray rescheduling plus Kubernetes |
| Ray head failure | KubeRay/Ray cluster recovery strategy |
| GPU node loss | node autoscaler, worker replacement, serving failover |
| model artifact unavailable | artifact storage and startup retry |
| gateway failure | ingress/LB HA |
| external state loss | durable backing service |
| region outage | multi-region routing/DR design |

Ray actor restart is not the same as state restoration. In-memory KV cache, streaming state and in-flight requests may be lost. Document retry/idempotency contracts and avoid claiming exactly-once inference.

## 10. RayService and rolling updates

KubeRay RayService can manage Ray cluster and Serve application lifecycle, but rollout and traffic-switching behavior depend on operator version and configuration. Pin Ray/KubeRay images and CRDs; stage changes; compare current and pending cluster/app status; validate synthetic golden requests; retain a complete immutable rollback bundle.

## 11. GPU topology and distributed inference

Tensor/pipeline parallel replicas require rank-consistent placement and enough GPU capacity. Scaling a multi-GPU replica can fail even when individual GPUs appear available because of topology, strict placement groups, NCCL/network constraints or insufficient per-node GPU counts.

## 12. Failure budgets and SLO

Define valid terminal completion, TTFT, TPOT, E2E, availability and fairness per workload cohort. Alert on both latency burn and capacity unavailability (Pending GPU workers, pending actors, failed replica startup). Record detection, mitigation and restoration times during drills.

## 13. Autoscaling experiments

| Experiment | Stimulus | Evidence |
|---|---|---|
| baseline | steady synthetic traffic | warm capacity, SLO |
| controlled ramp | increasing offered load | desired/ready replicas |
| burst | short demand spike | queue/rejection, TTFT |
| long-context | larger prompt/output | VRAM, decode |
| quota ceiling | no spare GPU capacity | bounded backlog |
| scale-down | load decreases | drain/terminal completion |
| replica crash | isolated staging only | restart time |
| worker loss | tabletop by default | recovery design |

Never test unbounded load against production.

## 14. Safe-read Kubernetes triage

```bash
date -u
kubectl config current-context
kubectl get rayservices,rayclusters -A
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -100
```

For approved namespace:

```bash
kubectl -n RAY_NAMESPACE describe rayservice APPROVED_RAYSERVICE
kubectl -n RAY_NAMESPACE get pods -o wide
```

If CRDs are absent, stop and record; do not install KubeRay without approval.

## 15. Observability

Collect Serve desired/ready replicas, request backlog, ongoing requests, actor pending/scheduling delay, Ray worker capacity, GPU Pod Pending, model cold-start duration, per-GPU VRAM, TTFT/TPOT/E2E, terminal completion, autoscaler decisions and GPU-hours. Validate actual metric names, units and labels for the installed version.

## 16. Security and FinOps

Restrict dashboard, Jobs API, model registry, KubeRay operator and autoscaler permissions. Quotas and cost ceilings protect against accidental runaway GPU provisioning. Track provisioned-but-idle GPU-hours and scale-down savings only after accounting for SLO and warm-capacity requirements.

## 17. Troubleshooting decision tree

```text
Autoscaling or availability degraded?
 ├─ desired replicas unchanged → policy/signal/config
 ├─ replicas desired but pending → Ray resources/placement
 ├─ workers pending → Kubernetes quota/taint/GPU
 ├─ nodes absent → node autoscaler/cloud quota
 ├─ replicas slow to ready → image/model/CUDA warmup
 ├─ scaling oscillates → delays/smoothing/traffic
 ├─ scale-down drops streams → drain/cancellation
 ├─ worker crash → actor restart/state loss
 ├─ Ray head unavailable → control-plane recovery
 ├─ gateway retries spike → overload policy
 ├─ metrics stale → scrape/exporter
 └─ GPU cost rises → warm pool/idle capacity
```

## 18. Twenty-four production failure scenarios

| ID | Failure | Evidence |
|---|---|---|
| F01 | autoscaling config unsupported | schema |
| F02 | min replicas below warm SLO | TTFT |
| F03 | max replicas exceeds quota | capacity |
| F04 | desired replica signal stale | metrics |
| F05 | scaling decision delayed | controller |
| F06 | Ray actors Pending | scheduler |
| F07 | placement group infeasible | bundles |
| F08 | GPU worker Pod Pending | events |
| F09 | GPU node cloud quota exhausted | node events |
| F10 | device plugin unavailable | allocatable |
| F11 | model image pull slow | pull timing |
| F12 | weights download fails | artifact log |
| F13 | CUDA warmup exceeds deadline | startup |
| F14 | replica startup OOM | VRAM |
| F15 | queue grows unbounded | backlog |
| F16 | retries amplify overload | gateway |
| F17 | scaling oscillation | desired/ready |
| F18 | scale-down interrupts streaming | terminal validator |
| F19 | replica restart loses state | request traces |
| F20 | worker loss stalls distributed ranks | rank logs |
| F21 | Ray head outage | control plane |
| F22 | metrics missing | scrape freshness |
| F23 | rollback fails to restore model | manifest |
| F24 | idle GPU-hours exceed budget | FinOps |

## 19. Sixteen operational runbooks

1. Validate pinned autoscaling schema and policy.
2. Diagnose missing or stale scaling signals.
3. Diagnose desired versus ready replicas.
4. Recover pending actors and placement groups.
5. Recover GPU worker Pod scheduling.
6. Diagnose node autoscaler and cloud quota.
7. Restore GPU device plugin resources.
8. Recover image/model artifact cold start.
9. Recover CUDA initialization and startup OOM.
10. Bound queue and retry amplification.
11. Tune scaling oscillation and cooldown.
12. Restore graceful drain and terminal streaming.
13. Recover replica/worker actor state loss.
14. Recover Ray head and distributed rank availability.
15. Restore telemetry, SLO and GPU FinOps.
16. Execute RayService full-stack rollback and RCA.

## 20. Eight tabletop game days

1. Traffic bursts faster than GPU provisioning.
2. Serve desires replicas but Ray placement groups cannot fit.
3. Cloud GPU quota blocks new worker nodes.
4. Model download doubles cold-start duration.
5. Scale-down interrupts long streaming responses.
6. GPU worker dies during distributed inference.
7. Ray head fails during a RayService rollout.
8. Scaling oscillation doubles GPU-hours while SLO worsens.

Default tabletop. Live fault injection requires separately approved isolated staging.

## 21. Twenty acceptance gates

G01 approved scope/versions; G02 autoscaling schema; G03 min/max capacity; G04 demand signals; G05 desired/ready reconciliation; G06 Ray actor scheduling; G07 GPU worker scaling; G08 node/cloud quota; G09 model cold start; G10 VRAM headroom; G11 bounded queue; G12 retry/deadline; G13 scale-up latency; G14 scale-down draining; G15 replica/worker recovery; G16 Ray head/TP health; G17 security/privacy; G18 telemetry/SLO; G19 rollback/FinOps; G20 independent review.

See [companion lab](../labs/039-ray-serve-autoscaling-fault-tolerance-check.md). All gates remain pending until independently evidenced.

**Next: Chapter 040 — Ray Serve Observability and Recovery.**
