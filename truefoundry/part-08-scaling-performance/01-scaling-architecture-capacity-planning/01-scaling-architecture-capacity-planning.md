# 8.1 — Scaling Architecture and GPU-Aware Capacity Planning

**Part:** 08 — Scaling & Performance · **Chapter:** 1 of 7  
**Audience:** TrueFoundry platform engineers, GPU SREs, Kubernetes administrators, inference reliability engineers  
**Safety:** **[SAFE-READ]** inventory and observation; **[LAB-MUTATE]** isolated, approved nonproduction only; **[CHANGE-CONTROL]** production scaling or quota changes.

## 1. Objectives and prerequisites

By the end, you will distinguish request concurrency, serving replica scaling and GPU node scaling; calculate workload-driven GPU capacity; budget cold starts and failure headroom; choose scaling signals and safeguards; and diagnose scaling failures using the **Lowest Proven Healthy Layer (LPHL)** and **First Failed Transition (FFT)** method.

Prerequisites: Parts 06–07, a documented TrueFoundry deployment workflow for your installed release, read-only Kubernetes access, GPU quota visibility, vLLM model and tokenizer revisions, measured inference baselines and a named SLO owner.

> Scaling is a multi-layer control problem. Adding serving replicas cannot help when there is no schedulable GPU capacity; adding nodes cannot help if a model cannot load or an API gateway is rejecting traffic.

## 2. End-to-end scaling architecture

```text
Synthetic or production client requests
              │
       TLS / auth / quotas
              ▼
         API gateway
              │       request rate, 429/5xx, client latency
              ▼
      Kubernetes Service
              │       Ready EndpointSlices / load distribution
      ┌───────┴──────────┐
      ▼                  ▼
  vLLM Pod A         vLLM Pod B       ← serving replicas (horizontal scale)
  queue/KV/GPU       queue/KV/GPU
      │                  │
      └────────┬─────────┘
               ▼
    Kubernetes scheduler            ← resource requests, affinity, quota
               ▼
    GPU node pool / provisioner      ← autoscaler / Karpenter or equivalent
               ▼
    cloud GPU capacity + storage     ← instance quota, image/model download
```

**TrueFoundry** manages the authoritative application deployment and supported scaling configuration; **Kubernetes** places Pods; **node provisioners** add eligible capacity; **vLLM** schedules tokens within each replica. These loops have different delays and failure modes.

## 3. Four independent scaling dimensions

| Dimension | Control | Primary benefit | Constraint |
|---|---|---|---|
| In-replica batching/concurrency | Engine request and token budgets | Better GPU utilization | KV cache, tail latency, memory |
| Horizontal serving replicas | Deployment replica count / supported autoscaler | More independent inference capacity | GPU nodes, cold starts, cost |
| Vertical GPU sizing | GPU SKU/VRAM, CPU/RAM, tensor parallelism | Larger models and contexts | Instance availability, compatibility |
| GPU node capacity | Node groups / node provisioner | More schedulable GPUs | Cloud quota, launch time, topology |

Do not assume vLLM's continuous batching means a Kubernetes HPA is unnecessary. Likewise, do not assume CPU utilization predicts GPU-serving demand.

## 4. Define workload and service objectives first

Measure with approved instrumentation, keeping customer content out of metrics:

- Request arrival rate and burst factor, with peak-duration windows.
- Input and output token distributions (P50/P95/P99) and streaming share.
- Time to first token (TTFT), inter-token latency (ITL), end-to-end latency and errors.
- Prompt/output tokens per second at the **accepted SLO**, not maximum benchmark throughput.
- Running/waiting requests, queue age, KV-cache occupancy and preemption.
- GPU VRAM, utilization, device health, model load time and readiness transition.
- Availability target, number of fault domains and required failover reserve.

A realistic model can be **prefill-bound**, **decode-bound**, **KV-bound**, **network-bound** or **GPU-capacity-bound**. Choose the limiting metric based on measured workload behavior.

## 5. Capacity planning model

A first-order, **illustrative** estimate for required replicas is:

```text
steady_replicas = ceil(peak_output_tokens_per_second
                       / sustainable_output_tokens_per_second_per_replica)

failure_headroom_replicas = ceil(steady_replicas / (1 - failure_fraction))
```

The second expression is an **approximation** for homogeneous capacity; for a concrete N-1 node failure, explicitly subtract the GPUs lost in the largest failure domain. Model output-token throughput alone is insufficient when long prompts, mixed contexts or TTFT SLOs dominate.

### Worked example — hypothetical, not a deployment recommendation

| Input | Assumed value |
|---|---:|
| Peak accepted demand | 900 output tokens/s |
| Sustainable per-replica throughput at approved SLO | 250 output tokens/s |
| Steady-state replica requirement | ceil(900 / 250) = **4** |
| Reserve for one replica failure | **5 total** |
| GPU allocation per replica | 1 GPU |
| GPU capacity needed for five replicas | 5 allocatable GPUs |
| Additional rolling-update surge | Must be evaluated separately |

If the five GPUs are all on one node, this is **not node-level high availability**. Spread replicas across failure domains, evaluate largest-node loss and preserve capacity for rolling updates. Inference replicas may need multiple GPUs for tensor parallelism; account for **GPUs per Pod**, not Pods alone.

## 6. VRAM, KV cache and GPU fragmentation

Estimate memory separately for each GPU:

```text
GPU budget ≈ weights_per_rank + runtime_overhead
           + KV_cache + activation_peaks + safety_headroom
```

For a decoder-only transformer, a rough logical KV estimate is:

```text
KV bytes ≈ 2 × layers × KV_heads × head_dim
         × live_tokens × bytes_per_KV_element
```

Actual usage changes with tensor parallelism, quantization, prefix caching, block rounding and vLLM version. GPUs are scheduled using advertised resources, not free VRAM, unless an explicitly supported sharing/MIG system is in place. One idle-looking GPU may still be unable to admit a second GPU-requesting Pod.

## 7. Readiness and cold-start budget

A new replica does not contribute capacity until it becomes **Ready**:

```text
demand spike
  → metrics scrape/window
  → scaling decision
  → GPU node provision (if needed)
  → Pod schedule / image pull
  → model artifact download / CUDA init
  → weights and KV allocation / warmup
  → model-aware readiness / Service endpoint
  → first successful authorized inference
```

Record **time to usable capacity**, not only Pod creation time. If model cold starts take minutes, reactive autoscaling cannot absorb second-scale bursts alone. Combine a tested minimum warm capacity, admission controls, bounded queueing and predictive or scheduled scale where supported.

## 8. Choosing autoscaling signals

| Signal | Good for | Caveat |
|---|---|---|
| CPU utilization | CPU-bound tokenizer or gateway | Weak proxy for GPU decode |
| Requests per replica | Stable request costs | Long prompts vary substantially |
| Waiting requests / queue age | Saturation | Scale may lag cold starts |
| Token throughput and arrival rate | Token-oriented demand | Requires validated metric units |
| TTFT/ITL SLO breach | User-impact feedback | Lagging signal, noisy alone |
| KV-cache pressure | Memory/concurrency constraints | High use not always failure |
| GPU utilization | Capacity correlation | High utilization can be healthy |
| GPU VRAM headroom | OOM risk | Not a direct demand measure |

Use metrics validated for the installed vLLM/TrueFoundry release. For HPA/KEDA, ensure metric adapters, permissions, scrape targets and aggregation are functioning. Avoid unbounded scaling based on high-cardinality or stale metrics.

## 9. Scale-up, scale-down and disruption protection

**Scale-up:** verify cloud GPU quota, node SKU supply, scheduling affinity, taints/tolerations, GPU device plugin, storage bandwidth and warmup capacity. Configure a maximum replica ceiling to bound cost and blast radius.

**Scale-down:** protect in-flight streaming requests, respect graceful termination and gateway drain time, avoid terminating the only ready model replica, and validate any PodDisruptionBudget and topology-spread constraints. A PDB protects certain voluntary disruptions but does not guarantee capacity or prevent every autoscaler termination.

**Rollout:** consider maxSurge, maxUnavailable, spare GPU capacity and cold-start time. An otherwise valid rolling update can stall because no extra GPU is available.

## 10. Kubernetes read-only investigation [SAFE-READ]

```bash
kubectl config current-context
kubectl -n <namespace> get deployments,pods,hpa -o wide
kubectl -n <namespace> get endpointslices -o wide
kubectl -n <namespace> get pdb,resourcequota,limitrange
kubectl -n <namespace> get events --sort-by=.lastTimestamp
kubectl -n <namespace> describe deployment <deployment>
kubectl -n <namespace> describe pod <pod>
kubectl get nodes -o wide
kubectl describe node <gpu-node>
```

If KEDA is installed, inspect its CRDs and objects using the cluster's supported commands; do not assume it is present. If the platform is TrueFoundry-managed, read the **authoritative** autoscaling configuration before considering a Kubernetes patch.

### Verify GPU node placement

Record `nvidia.com/gpu` allocatable, Pod GPU requests, node taints, tolerations, selectors, affinity, topology spread and quota. A Pending Pod with `Insufficient nvidia.com/gpu` means the request cannot be placed; increasing HPA replicas will create more Pending Pods, not more serving capacity.

## 11. Cost and resilience guardrails

- Define GPU-hours budget and maximum scale ceiling per environment.
- Record effective **cost per million output tokens at the required SLO**, including idle warm replicas, storage, network and observability.
- Preserve at least the approved failure-domain reserve, not merely average traffic headroom.
- Ensure cloud quotas and node pool max sizes support the intended scale.
- Evaluate spot/preemptible GPU nodes only with appropriate disruption tolerance and fallbacks.
- Use immutable model/image revisions to avoid capacity differences caused by untracked runtime changes.

Cost optimization that removes failover reserve or violates TTFT SLO is not a successful optimization.

## 12. Troubleshooting matrix

| ID | Symptom | First failed transition | Evidence |
|---|---|---|---|
| S1 | HPA shows unknown metrics | Metrics API → HPA | HPA conditions/adapter |
| S2 | Scale signal stale | Scrape → adapter | Timestamps/series |
| S3 | Desired replicas increase, Pods Pending | Deployment → scheduler | Events and quota |
| S4 | No allocatable GPU | Device plugin → scheduler | Node capacity |
| S5 | GPU cloud quota exceeded | Provisioner → cloud | Quota errors |
| S6 | GPU SKU unavailable | Cloud supply → node | Provisioning events |
| S7 | New node lacks driver/plugin | Node → GPU allocatable | Driver/plugin health |
| S8 | Model cold start too slow | Pod → Ready | Image/artifact timings |
| S9 | 503 during scale-up | Service → ready endpoint | EndpointSlices |
| S10 | TTFT worsens at peak | Queue → prefill | Waiting and TTFT |
| S11 | ITL worsens with concurrency | Decode → GPU | KV/VRAM |
| S12 | KV OOM after increasing batch | Scheduler → cache | Active tokens |
| S13 | Replica imbalance | Gateway/Service → Pod | Per-replica requests |
| S14 | Scaling oscillation | Metrics → control loop | Stabilization history |
| S15 | Rollout stalls on GPU shortage | Surge → scheduler | GPU headroom |
| S16 | Scale-down breaks streams | Termination → client | Drain and completion |
| S17 | PDB blocks maintenance | Eviction → availability | PDB status |
| S18 | N-1 failure overwhelms remaining GPUs | Failure → reserve | Domain capacity |
| S19 | Cost rises without throughput gain | New replicas → useful work | Token throughput |
| S20 | Wrong model after scaling | Image/artifact → runtime | Immutable revision |

Do not diagnose a scaling problem from replica count alone. Record LPHL and FFT for each case.

## 13. Production scaling incident runbook

1. **Declare:** endpoint/model, affected tenants, traffic, TTFT/ITL, 429/5xx, stream completion and current replica count.
2. **Correlate:** scaling decisions, HPA/KEDA metrics, node provisioner events, GPU quota, rollout, image/model revision and cold-start time.
3. **Localize:** metric source → autoscaler → Deployment → scheduler → GPU node → image/artifact → vLLM readiness → gateway.
4. **Contain:** freeze unsafe promotion, protect known-good replicas, enforce approved admission limits and client retry backoff with jitter.
5. **Recover:** fix confirmed layer; if needed restore immutable known-good scaling/release configuration through TrueFoundry/GitOps.
6. **Validate:** correct model, Ready endpoints, GPU allocatable, queue drain, TTFT/ITL, errors, stream completion and cost ceiling.
7. **Document:** LPHL, FFT, peak demand, capacity gap, recovery time, follow-up owners and new acceptance tests.

## 14. Acceptance gates

| Gate | Required evidence |
|---|---|
| A1 | Approved scope, owner and workload/SLO definition |
| A2 | Four scaling dimensions and responsibility boundaries understood |
| A3 | Throughput-derived capacity and failure-domain reserve calculated |
| A4 | GPU/VRAM/KV and quota constraints verified |
| A5 | Cold-start-to-Ready budget measured or marked Not run |
| A6 | Scaling signals and metrics validation documented |
| A7 | S1–S20 failures classified with LPHL/FFT |
| A8 | Authoritative rollback, drain and post-recovery checks documented |

## 15. Evidence template

```text
Date/time / owner / approval:
TrueFoundry workspace, cluster and namespace:
Model/image digest, vLLM version, GPU SKU:
Traffic and prompt/output token distributions:
SLO targets and sustainable tokens/s per replica:
Steady replicas / N-1 reserve / surge headroom:
GPU allocatable, quotas, taints and placement:
VRAM, KV capacity, context and concurrency:
Metric source, HPA/KEDA settings and scaling events:
Node provision time, model load and time-to-Ready:
TTFT/ITL, waiting requests, 429/5xx, stream completion:
Lowest Proven Healthy Layer / First Failed Transition:
Rollback revision, recovery and cost ceiling:
```

**Hands-on lab:** [8.1 — Scaling Architecture and Capacity Planning Check](../labs/01-scaling-architecture-capacity-planning-check.md)  
**Next:** Chapter 8.2 — Horizontal Pod Autoscaling for Inference Workloads.

## References

- [TrueFoundry Documentation](https://docs.truefoundry.com/)
- [Kubernetes Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [Kubernetes GPU Scheduling](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/)
- [Kubernetes Pod Disruption Budgets](https://kubernetes.io/docs/tasks/run-application/configure-pdb/)
- [KEDA Documentation](https://keda.sh/docs/)
- [vLLM Documentation](https://docs.vllm.ai/)
