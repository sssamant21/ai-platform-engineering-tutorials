# 8.7 — Production Scaling and Recovery Capstone

**Part 08 — Scaling & Performance | Chapter 7 of 7**  
**Audience:** TrueFoundry platform engineers, Kubernetes/GPU SREs, inference reliability teams  
**Safety:** **[SAFE-READ]** observation and tabletop; **[LAB-MUTATE]** approved isolated staging only; **[CHANGE-CONTROL]** production change.

## 1. Mission

Integrate Chapters 8.1–8.6 into a repeatable production-readiness assessment for a GPU-backed inference endpoint. Demonstrate that demand produces **SLO-compliant Ready serving capacity**, not merely HPA desired replicas or GPU nodes. Exercise controlled scaling, diagnose simulated failures, recover via an authoritative release, calculate cost and capture evidence for operational acceptance.

**Completion of this written chapter is not evidence that a live system passed.** Lab execution, vendor/version review, change approval and SRE signoff remain separate gates.

## 2. Learning outcomes

After completing the capstone, an engineer can:

1. Define an inference SLO, workload envelope and admission strategy.
2. Model GPU capacity with largest-failure-domain N-1 reserve and rollout surge.
3. Explain TrueFoundry, HPA, KEDA and node-autoscaler ownership boundaries.
4. Measure demand → metrics → scaling decision → GPU allocation → model Ready → client recovery.
5. Benchmark TTFT, ITL, E2E, throughput and SLO-constrained goodput.
6. Identify queue, KV, GPU, model startup, metrics and cloud capacity bottlenecks.
7. Calculate GPU-hours and cost per million useful output tokens.
8. Conduct bounded failure drills and restore the immutable known-good release.
9. Produce a reviewable go/no-go decision with evidence and unresolved risks.

## 3. Reference system and trust boundaries

```text
Authorized synthetic client
       │
       ▼
Gateway: auth / quota / admission / streaming drain
       │
       ▼
Service → Ready vLLM Pods (TrueFoundry deployment)
       │          ├─ prefill/decode, KV, TTFT/ITL, tokens/s
       │          └─ GPU device/runtime and model readiness
       │
       ├─ Prometheus / OpenTelemetry / Grafana
       │       └─ approved metrics adapters
       ├─ HPA or KEDA-managed HPA
       │       └─ desired Pod replicas
       └─ GPU scheduler → CA/Karpenter/AKS autoscaler
               └─ cloud GPU nodes → device plugin → Ready
       
GitOps/TrueFoundry: authoritative configuration and rollback
Cloud billing: GPU-hours, allocation, idle capacity, total cost
```

**Critical ownership rule:** Do not let TrueFoundry, GitOps, a manually created HPA and KEDA independently fight over the same replica target. KEDA often manages its own HPA. Node autoscalers provision infrastructure but do not guarantee that model endpoints become Ready. Confirm supported integrations for the installed releases before a mutation.

## 4. Capstone scenario

**Illustrative only:** A customer-facing streaming model endpoint must tolerate a bounded burst, a GPU node outage and a delayed GPU scale-up while meeting an approved TTFT/ITL and error SLO. The workload includes short/long prompts and short/long outputs. A limited synthetic load test runs in isolated staging with a preapproved GPU cost ceiling.

The capstone is **provider neutral**. EKS might use Karpenter or Cluster Autoscaler; AKS typically uses supported GPU node-pool autoscaling. Do not assume KEDA, GPU Operator, Prometheus or Karpenter is installed.

### Evidence baseline

| Property | Record before any change |
|---|---|
| TrueFoundry workspace / cluster / namespace | Exact names and owner |
| Model/tokenizer/image | Immutable revision/digest |
| vLLM/Kubernetes/KEDA/autoscaler versions | Actual installed versions |
| GPU SKU, driver, plugin and topology | Device allocatable and Pod requests |
| Admission and request limits | Quota, max queue age, retry policy |
| SLO | TTFT, ITL, E2E, 429/5xx, completion |
| HPA/KEDA | Metric, threshold, min/max, behavior, owner |
| Node autoscaler | Pool/NodePool limits, cloud quota, zones |
| Cost | Rate source, max GPU-hours, stop threshold |
| Recovery | Known-good release and rollback operator |

## 5. Capacity planning gate

Calculate required serving replicas from **measured** SLO-qualified throughput for the representative token mix:

```text
serving_replicas =
  ceil(peak_accepted_output_tokens_per_second
       / sustainable_output_tokens_per_second_per_replica_at_SLO)

steady_GPUs = serving_replicas × GPUs_per_Pod
planned_GPUs = steady_GPUs + largest_failure_domain_reserve + rollout_surge
```

These are planning approximations: prompt prefill, KV capacity, model topology and network behavior may make output tokens/s insufficient alone. Verify CPU/RAM, GPU per Pod, affinity, taints, topology spread, subnet IPs, namespace quota and cloud GPU instance supply.

**Worked hypothetical:** 1,100 output tokens/s peak ÷ 280 tokens/s per one-GPU replica = 4 serving replicas. If the largest GPU node holds two replicas, N-1 node resilience needs at least **two** spare replica slots, not one; a rolling surge can require more. Validate that reserve is actually schedulable in another failure domain.

## 6. Define the scale-up acceptance timeline

```text
T0  synthetic demand crosses agreed threshold
T1  event-source/adapter metric reflects demand
T2  HPA/KEDA records desired-replica decision
T3  Deployment creates Pods
T4  scheduler declares GPU demand / Pending
T5  autoscaler launches or selects GPU node
T6  node Ready and nvidia.com/gpu allocatable
T7  Pod scheduled and image/model loaded
T8  CUDA/vLLM initialized and warmup completed
T9  Ready EndpointSlice includes the Pod
T10 client TTFT/ITL/error SLO recovered
```

Measure **T10−T0**, not merely node launch duration. Record actual timestamps and gaps. A Pod Running without model readiness is not usable capacity.

## 7. Observability and evidence contract

| Layer | Required metrics |
|---|---|
| Client/gateway | TTFT/ITL/E2E percentiles, 429/5xx, timeouts, stream completion |
| vLLM | Waiting/running requests, queue age, KV usage/preemption, input/output tokens/s |
| HPA/KEDA | Trigger freshness, conditions, desired/current replicas, scaling events |
| Deployment | Desired/available/Ready replicas and EndpointSlices |
| Scheduler | Pending reasons, affinity/taints/topology, GPU resource fit |
| Node autoscaler | Provisioner decisions, node boot, quota/capacity errors |
| GPU | Allocatable, VRAM, utilization, driver health, ECC/XID |
| Cost | GPU-hours, idle warm capacity, useful tokens, cost per useful million |

Discover actual exporter names, labels and histogram buckets. **Do not average per-Pod P95 values**; aggregate histogram buckets before computing quantiles. Keep prompts, tokens and customer identifiers out of metric labels and incident attachments.

## 8. Benchmark matrix

| Test | Input tokens | Output tokens | Concurrency | Goal |
|---|---:|---:|---:|---|
| B1 | 128 | 64 | 1 | Warm baseline |
| B2 | 128 | 256 | 8 | Decode |
| B3 | 2,048 | 64 | 8 | Prefill |
| B4 | 2,048 | 512 | 16 | Mixed pressure |
| B5 | 8,192 | 128 | 4 | Long context |
| B6 | Representative synthetic P50/P95 mix | Mixed | Ramped | SLO goodput |
| B7 | Shared-prefix synthetic mix | Mixed | Ramped | Cache benefit |
| B8 | Interactive + batch | Mixed | Ramped | Fairness |

All values are **illustrative**. Reduce lengths/concurrency to the actual model context, SLO and approved budget. Capture repeat-run variance, sample counts, arrival rate, output-length distribution, streaming ratio and warm/cold conditions.

## 9. Execution phases

### Phase A — Read-only discovery [SAFE-READ]

```bash
kubectl config current-context
kubectl -n <namespace> get deployments,pods,hpa -o wide
kubectl -n <namespace> get endpointslices -o wide
kubectl -n <namespace> get resourcequota,pdb
kubectl -n <namespace> get events --sort-by=.lastTimestamp
kubectl get nodes -o wide
kubectl describe node <gpu-node>
kubectl -n <namespace> describe pod <inference-pod>
```

Optional where installed:

```bash
kubectl -n <namespace> get scaledobjects.keda.sh
kubectl get nodepools.karpenter.sh
kubectl get nodeclaims.karpenter.sh
```

If a CRD is absent, mark **Not installed**; do not install a cluster-wide component merely to satisfy a lab.

### Phase B — Ownership and safety review

Confirm TrueFoundry/GitOps authoritative settings, metric adapter and trigger ownership, max replicas, GPU node-pool limit, node disruption policy, model-aware readiness, streaming drain and known-good rollback. Ensure no two HPAs target the same Deployment.

### Phase C — Controlled scale-up [LAB-MUTATE]

With explicit staging approval, run bounded synthetic traffic and observe T0–T10. Stop on SLO breach, 429/5xx escalation, prolonged queue, GPU Pending, OOM, cost ceiling or unexpected disruption. Capture both successful and unsuccessful transitions.

### Phase D — Performance tuning

Classify prefill, decode, KV, admission, GPU or cold-start bottleneck. Change **one** supported vLLM/TrueFoundry setting at a time; repeat the same benchmark mix and compare SLO goodput, tail latency, errors, model quality and cost. Roll back any regression.

### Phase E — Failure and recovery drills

Perform tabletop by default; mutate only disposable lab components when separately approved. Use the scenario matrix in Section 10. Restore via the authoritative workflow and validate Ready endpoints, correct model revision, queue drain, SLO and cost ceiling.

### Phase F — Final readiness review

Compile results, outstanding risks, owners and evidence links. Mark unexecuted gates **Not run**, not Passed. Approve production promotion only with change control, platform/vendor review and SRE acceptance.

## 10. Failure-injection and recovery matrix

| ID | Scenario | First failed transition to inspect | Recovery validation |
|---|---|---|---|
| C1 | HPA metric missing | Adapter → HPA | Valid current metric |
| C2 | KEDA trigger stale/auth denied | Source → KEDA | Active/Ready condition |
| C3 | Conflicting replica controllers | Source of truth → Deployment | Stable desired replicas |
| C4 | GPU Pods Pending | Scheduler → GPU node | Schedulable replicas |
| C5 | Cloud GPU quota exhausted | Provisioner → cloud | Node launch possible |
| C6 | GPU instance unavailable | Cloud → node | Approved fallback |
| C7 | Device plugin missing | Node → GPU allocatable | GPU advertised |
| C8 | CUDA/vLLM initialization fails | Runtime → model | Healthy model startup |
| C9 | Model cold start exceeds SLO | Pod → Ready | Warm capacity/TTFT |
| C10 | Long prompt TTFT regression | Prefill → first token | TTFT by input bucket |
| C11 | Decode ITL regression | KV/decode → stream | ITL by output bucket |
| C12 | KV preemption/OOM | KV budget → runtime | Stable memory/errors |
| C13 | Gateway 429/retry storm | Admission → client | Controlled retries |
| C14 | Scale-down interrupts streams | Drain → client | Completed streams |
| C15 | GPU node loss | Failure domain → Ready replicas | N-1 reserve |
| C16 | Spot interruption | Cloud → replacement | No SLO loss or bounded impact |
| C17 | Rollout surge cannot schedule | Deployment → capacity | Ready rollout |
| C18 | Cost spike without goodput | GPU spend → useful work | Budget and goodput |
| C19 | Observability gap | Exporter → dashboard | Valid samples |
| C20 | False alert on healthy high GPU | Alert → operator | SLO-based policy |
| C21 | Wrong model revision after scale | Artifact → runtime | Digest verified |
| C22 | Wrong GPU affinity/taint | Scheduler → node | Eligible placement |
| C23 | PVC/artifact download bottleneck | Storage → warmup | Load time within budget |
| C24 | Rollback leaves orphan GPU nodes | Recovery → cleanup | Cost/resource baseline |

For each record **LPHL** (Lowest Proven Healthy Layer), **FFT** (First Failed Transition), blast radius, impact, safe mitigation, rollback, recovery time and evidence. Do not inject node failure or cloud quota changes into shared production environments.

## 11. Cost and FinOps acceptance

```text
GPU_cost = Σ(instance_GPU_hour_rate × billed_instance_hours)

total_serving_cost = GPU_cost + allocated_CPU_RAM_storage_network
                   + monitoring + platform_costs

cost_per_million_useful_output_tokens =
    total_serving_cost / (SLO_compliant_output_tokens / 1_000_000)
```

Label effective rate, discounts, allocation boundaries, warm-idle costs and whether GPU-hours reflect billed instances or requested GPU slices. Example only: 96 GPU-hours × $2 = $192 GPU-only; 16 million SLO-compliant output tokens → **$12 per million useful output tokens** before other costs. Compare model revisions only under equivalent workload/SLO/quality assumptions.

## 12. Production incident command sequence

1. **Declare** affected endpoint/model, onset, TTFT/ITL/E2E, 429/5xx, queue and stream completion.
2. **Build timeline** from demand through metrics, HPA/KEDA, scheduler, GPU nodes, model readiness and client recovery.
3. **Find FFT** and preserve events, metrics, release diff, node/provisioner decisions and model digest.
4. **Contain** using approved admission limits, warm replicas, jittered retries and rollout freeze.
5. **Recover** first failed transition or roll back immutable TrueFoundry/GitOps release; avoid ad hoc controller conflicts.
6. **Validate** Ready endpoints, model revision, SLO, queue drain, GPU N-1 reserve and cost.
7. **Close** with LPHL/FFT, timeline, RTO/RPO where applicable, owners, alert improvements and corrective actions.

## 13. Capstone acceptance scorecard

| Gate | Acceptance evidence |
|---|---|
| A1 | SLO and workload envelope explicitly defined |
| A2 | Immutable release, TrueFoundry/HPA/KEDA/node owner verified |
| A3 | GPU fit, quota, largest-domain N-1 and surge calculated |
| A4 | Metrics and T0–T10 usable-capacity timeline recorded |
| A5 | Representative benchmark, TTFT/ITL and SLO goodput evaluated |
| A6 | C1–C24 recovery drills classified with LPHL/FFT |
| A7 | FinOps cost, warm-idle budget and forecast validated |
| A8 | Graceful drain, rollback and correct model confirmed |
| A9 | Incident runbook, alerts and escalation reviewed |
| A10 | Platform/vendor/SRE review and go/no-go signoff recorded |

**Decision policy:** Any critical failed gate blocks promotion; Not run is **not** Passed. A documented waiver requires an accountable owner, risk and expiration date.

## 14. Final evidence package

```text
Capstone ID / date / reviewers / change approval:
Workspace / EKS or AKS / namespace / environment:
Model/tokenizer/image digest / vLLM / TrueFoundry version:
HPA/KEDA/CA/Karpenter ownership and versions:
GPU SKU/count, driver/plugin, quota, subnet, topology:
Workload shape, arrival rate, concurrency, SLO:
B1–B8 benchmark results and sample counts:
T0–T10 timestamps and time-to-usable-capacity:
TTFT/ITL/E2E, errors, queue, KV, tokens/s, goodput:
Largest-failure-domain N-1 reserve and surge:
C1–C24 LPHL/FFT, recovery times and evidence:
GPU-hours, cost per useful million, idle budget:
Known-good rollback, stream drain, model verification:
A1–A10 Pass/Fail/Not run, risks, owner and signoff:
```

**Hands-on capstone lab:** [8.7 — Production Scaling and Recovery Acceptance Lab](../labs/07-production-scaling-recovery-capstone-check.md)

**Part 08 publication milestone:** All seven canonical chapters and companion labs are published. Live execution and formal review are separate milestones.

**Next curriculum part:** Part 09 — Networking (confirm its existing master layout before authoring).

## References

- [TrueFoundry Documentation](https://docs.truefoundry.com/)
- [vLLM Documentation](https://docs.vllm.ai/)
- [Kubernetes Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [KEDA Documentation](https://keda.sh/docs/)
- [Karpenter Documentation](https://karpenter.sh/docs/)
- [Kubernetes GPU Scheduling](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/)
- [Prometheus Histogram Guidance](https://prometheus.io/docs/practices/histograms/)
- [NVIDIA DCGM Exporter](https://github.com/NVIDIA/dcgm-exporter)
