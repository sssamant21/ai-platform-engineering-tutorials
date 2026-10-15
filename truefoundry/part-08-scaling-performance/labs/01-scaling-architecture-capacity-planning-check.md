# Lab 8.1 — Scaling Architecture and GPU-Aware Capacity Planning Check

**Canonical:** [Chapter 8.1](../01-scaling-architecture-capacity-planning/01-scaling-architecture-capacity-planning.md)  
**Default:** [SAFE-READ]. Synthetic scaling tests require [LAB-MUTATE] and approval in an isolated nonproduction workspace.

## 1. Goal and prerequisites

Calculate GPU-serving capacity at a defined TTFT/ITL SLO, verify Kubernetes/TrueFoundry scheduling and autoscaling prerequisites, assess cold starts and failure reserve, and classify 20 scaling failure modes.

Record an approved TrueFoundry workspace, read-only Kubernetes context, vetted immutable model and image, GPU pool, monitoring source, SLO owner and known-good rollback release. No production load injection.

## 2. Read-only discovery

```bash
kubectl config current-context
kubectl -n <namespace> get deployments,pods,hpa -o wide
kubectl -n <namespace> get endpointslices -o wide
kubectl -n <namespace> get pdb,resourcequota,limitrange
kubectl -n <namespace> get events --sort-by=.lastTimestamp
kubectl get nodes -o wide
kubectl describe node <gpu-node>
kubectl -n <namespace> describe deployment <deployment>
```

Do not assume HPA, KEDA or GPU node autoscaling is installed. Record absent components as Not configured.

## 3. Capacity worksheet

```text
Steady replicas = ceil(peak accepted output tokens/s
                       ÷ sustainable output tokens/s per replica)
Required GPU allocation = replicas × GPUs per replica
```

| Input | Planned | Observed/source |
|---|---|---|
| Peak request rate and burst duration | | |
| Prompt/output token P50/P95 | | |
| TTFT and ITL SLO | | |
| Sustainable tokens/s per replica at SLO | | |
| Steady-state replicas | | |
| GPUs per replica / TP size | | |
| Largest node/failure-domain GPU loss | | |
| N-1 reserve and rolling surge | | |
| Cloud quota / node pool maximum | | |
| GPU-hours budget | | |

Explain why token throughput alone cannot determine TTFT for long prompts.

## 4. GPU/VRAM/KV budget

| Resource | Evidence | Result |
|---|---|---|
| GPU node SKU and allocatable | | |
| Pod GPU request/limit | | |
| Node taints and tolerations | | |
| CPU/RAM for tokenizer and loader | | |
| Weights and runtime VRAM | | |
| KV cache capacity and headroom | | |
| Max context/output/concurrency | | |
| Namespace and cloud GPU quota | | |

Record assumptions for logical KV formula and compare with engine-reported capacity when available.

## 5. Cold-start and time-to-capacity

| Stage | Measured duration | Evidence |
|---|---|---|
| Metrics detection and scaling decision | | |
| GPU node provisioning (if required) | | |
| Pod scheduling | | |
| Image pull | | |
| Model download/verification | | |
| CUDA/weights/KV initialization | | |
| Warmup and model-aware readiness | | |
| First successful synthetic inference | | |

If not executed, mark Not run rather than inventing timing.

## 6. Scaling signal worksheet

| Signal | Actual metric/type | Validated? | Risk |
|---|---|---|---|
| Requests or tokens/s | | | |
| Running/waiting requests | | | |
| TTFT/ITL | | | |
| KV occupancy/preemptions | | | |
| GPU utilization/VRAM | | | |
| Ready endpoints | | | |
| Cloud GPU quota | | | |

Avoid unverified metric names or high-cardinality labels.

## 7. Optional bounded scaling exercise [LAB-MUTATE]

Only in approved isolated staging with spare GPUs, cost ceiling, test duration and stop thresholds:

1. Confirm the authoritative TrueFoundry/GitOps scaling configuration and immutable release.
2. Record initial replica count, GPU node allocatable and Ready endpoints.
3. Send a bounded synthetic workload at low arrival rate.
4. Increase load within the approved rate/token budget; observe scaling decision, Pod placement, image/model cold start and time to Ready.
5. Record queue, TTFT/ITL, errors, GPU VRAM, token throughput and cost.
6. Return to baseline; verify stream draining and stable endpoints.
7. Restore the known-good release and clean up only lab-owned resources.

If capacity is unavailable, conduct a tabletop and mark the live exercise Not run.

## 8. Failure classification

| ID | Scenario | LPHL | FFT | Evidence | Safe response |
|---|---|---|---|---|---|
| S1 | HPA metric unknown | | | | |
| S2 | Stale metric | | | | |
| S3 | Pods Pending after scale | | | | |
| S4 | No allocatable GPU | | | | |
| S5 | Cloud quota exceeded | | | | |
| S6 | GPU SKU unavailable | | | | |
| S7 | New node driver/plugin missing | | | | |
| S8 | Model cold start slow | | | | |
| S9 | 503 during scale-up | | | | |
| S10 | TTFT worsens at peak | | | | |
| S11 | ITL worsens | | | | |
| S12 | KV OOM at concurrency | | | | |
| S13 | Replica imbalance | | | | |
| S14 | Scaling oscillation | | | | |
| S15 | Rollout GPU shortage | | | | |
| S16 | Scale-down stream interruption | | | | |
| S17 | PDB prevents eviction | | | | |
| S18 | N-1 reserve insufficient | | | | |
| S19 | Cost rises without throughput | | | | |
| S20 | Wrong model on new replica | | | | |

LPHL = Lowest Proven Healthy Layer; FFT = First Failed Transition. Distinguish tabletop from live evidence.

## 9. Acceptance gates

| Gate | Pass / Fail / Not run | Evidence |
|---|---|---|
| A1 Approved scope and SLO | | |
| A2 Scaling dimensions/ownership | | |
| A3 Capacity and failure reserve | | |
| A4 GPU/KV/quota validation | | |
| A5 Cold-start budget | | |
| A6 Signal/metrics validation | | |
| A7 S1–S20 classifications | | |
| A8 Rollback, drain and cleanup | | |

## 10. Final evidence

```text
Date/time, approval and workspace:
Image/model and GPU SKU:
SLO and request/token distribution:
Sustainable tokens/s per replica:
Replicas, GPU quota, N-1 and surge reserve:
GPU VRAM/KV and context budget:
Scaling signal and metric validation:
Node provision, Pod Ready and cold-start timings:
TTFT/ITL, queue, errors, cost:
S1–S20 classification and LPHL/FFT:
Rollback release, stream drain and cleanup:
Unexecuted gates and review owner:
```

**Next:** Chapter 8.2 — Horizontal Pod Autoscaling for Inference Workloads.
