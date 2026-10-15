# Lab 8.2 — Horizontal Pod Autoscaling for Inference Workloads Check

**Canonical:** [Chapter 8.2](../02-horizontal-pod-autoscaling-inference-workloads/02-horizontal-pod-autoscaling-inference-workloads.md)  
**Default:** [SAFE-READ]. HPA creation, workload generation and policy changes require [LAB-MUTATE] in approved isolated staging.

## 1. Objectives and guardrails

Prove HPA ownership, target, metric freshness, desired-replica computation, GPU placement, model-aware readiness, scale-down drain and rollback. Use synthetic requests and explicit GPU/cost ceilings. Do not install cluster-wide adapters or edit TrueFoundry-managed production resources.

## 2. Preflight

```bash
kubectl config current-context
kubectl -n <namespace> get hpa,deployments,pods -o wide
kubectl -n <namespace> get endpointslices -o wide
kubectl -n <namespace> get resourcequota,pdb
kubectl -n <namespace> get events --sort-by=.lastTimestamp
kubectl get nodes -o wide
```

| Check | Expected | Observed |
|---|---|---|
| Approved workspace/namespace | | |
| TrueFoundry/GitOps ownership | | |
| Immutable model and image | | |
| GPU quota and node capacity | | |
| Known-good scaling policy | | |
| TTFT/ITL/error SLO and test ceiling | | |

## 3. HPA calculation exercise

For a single metric, `desired = ceil(current × observed / target)` before tolerance, readiness and stabilization adjustments.

| Current replicas | Observed metric | Target | Raw desired | After policies |
|---:|---:|---:|---:|---|
| 3 | 80% CPU | 60% | 4 | |
| 2 | 6 waiting requests/Pod | 2/Pod | 6 | |
| 4 | 1 waiting request/Pod | 2/Pod | 2 | |

Explain why maxReplicas, missing metrics, scale-down stabilization and Ready replicas can change the final outcome.

## 4. Metric and adapter validation [SAFE-READ]

```bash
kubectl -n <namespace> describe hpa <hpa-name>
kubectl top pods -n <namespace>
kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1
kubectl get --raw /apis/external.metrics.k8s.io/v1beta1
```

Run optional commands only where supported and authorized. Record missing adapters as Not configured, not runtime failure.

| Signal | Actual metric/API | Fresh? | HPA status | Evidence |
|---|---|---|---|---|
| CPU | | | | |
| Waiting requests | | | | |
| Token demand | | | | |
| Ready replicas | | | | |
| GPU quota/allocatable | | | | |

## 5. Optional HPA staging test [LAB-MUTATE]

1. Confirm authoritative TrueFoundry deployment supports external HPA ownership; otherwise use a separate lab-owned Deployment.
2. Confirm resource requests, metric adapter (if custom), min/max and rollback.
3. Deploy an approved lab HPA from Chapter 8.2 through the supported workflow.
4. Generate bounded synthetic traffic and record HPA current/desired, Pod Pending, GPU placement and model warmup.
5. Verify actual Ready EndpointSlices, TTFT/ITL, queue, errors and cost ceiling.
6. Stop on SLO violation, queue growth, GPU shortage, unexpected spend or stream failures.
7. Remove traffic, observe stabilization and stream drain, then restore known-good scaling configuration.
8. Confirm model revision and baseline health; clean up only lab-owned resources.

If no approved GPU capacity or adapter exists, mark live steps Not run and use tabletop.

## 6. Failure tabletop

| ID | Failure | LPHL | FFT | Evidence | Safe response |
|---|---|---|---|---|---|
| H1 | Target missing | | | | |
| H2 | CPU unknown | | | | |
| H3 | Custom metric missing | | | | |
| H4 | Label mismatch | | | | |
| H5 | Metric stale | | | | |
| H6 | At max replicas | | | | |
| H7 | Pods Pending | | | | |
| H8 | GPU nodes unavailable | | | | |
| H9 | Pods not Ready | | | | |
| H10 | Scale too late | | | | |
| H11 | Replica oscillation | | | | |
| H12 | Traffic imbalance | | | | |
| H13 | Stream interruption | | | | |
| H14 | HPA/TrueFoundry conflict | | | | |
| H15 | Rollout 503 | | | | |
| H16 | TTFT high | | | | |
| H17 | ITL high | | | | |
| H18 | PDB blocked | | | | |
| H19 | GPU cost overrun | | | | |
| H20 | Wrong model revision | | | | |

LPHL = Lowest Proven Healthy Layer; FFT = First Failed Transition. A tabletop does not prove live recovery.

## 7. Acceptance gates

| Gate | Pass / Fail / Not run | Evidence |
|---|---|---|
| A1 Ownership and scope | | |
| A2 HPA calculations and metric types | | |
| A3 Metrics/adapter validity | | |
| A4 GPU capacity and readiness | | |
| A5 Policies and graceful drain | | |
| A6 Bounded scaling test/tabletop | | |
| A7 H1–H20 classification | | |
| A8 Rollback and post-recovery SLO | | |

## 8. Final evidence report

```text
Date/time and change approval:
Workspace/namespace/owner:
Image/model revision and GPU SKU:
HPA target, min/max, behavior and metric:
Adapter API, metric freshness and conditions:
Current/desired/Ready replicas and GPU Pending:
TTFT/ITL, queue, 429/5xx, stream completion:
Scaling time, cold start, cost ceiling:
H1–H20 LPHL/FFT classifications:
Known-good rollback and cleanup:
A1–A8 results and unexecuted gates:
```

**Next:** Chapter 8.3 — KEDA Event-Driven Scaling and Custom Metrics.
