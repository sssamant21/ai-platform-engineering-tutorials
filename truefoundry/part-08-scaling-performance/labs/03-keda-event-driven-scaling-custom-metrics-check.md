# Lab 8.3 — KEDA Event-Driven Scaling and Custom Metrics Check

**Canonical:** [Chapter 8.3](../03-keda-event-driven-scaling-custom-metrics/03-keda-event-driven-scaling-custom-metrics.md)  
**Default:** [SAFE-READ]. Creating ScaledObjects, traffic generation or fault injection requires [LAB-MUTATE] in approved isolated staging.

## 1. Objectives

Inspect KEDA, confirm TrueFoundry/HPA ownership, validate an authorized Prometheus trigger, observe the scaling control loop, measure GPU time-to-Ready, safely drain streams, and classify 20 failure scenarios.

## 2. Approval and environment record

```text
Workspace/namespace/cluster:
Owner/change approval:
Immutable image/model/tokenizer revision:
GPU SKU/quota, maximum replicas and cost:
SLO TTFT/ITL/error and stream completion:
Synthetic workload, duration and stop thresholds:
Prometheus endpoint/auth owner:
Authoritative rollback release:
```

Do not use real prompts or inspect secret values.

## 3. Read-only inventory

```bash
kubectl config current-context
kubectl get crd scaledobjects.keda.sh
kubectl -n <namespace> get scaledobjects.keda.sh
kubectl -n <namespace> get hpa,deployments,pods -o wide
kubectl -n <namespace> get endpointslices -o wide
kubectl -n <namespace> get events --sort-by=.lastTimestamp
kubectl -n <namespace> get resourcequota,pdb
kubectl get nodes -o wide
```

If KEDA is absent, mark Not installed and perform a tabletop rather than installing an operator without authorization.

## 4. Trigger contract worksheet

| Field | Expected | Observed / evidence |
|---|---|---|
| Event source and authorized endpoint | | |
| Real query and returned single value | | |
| Metric units and aggregation | | |
| Sample freshness and scrape interval | | |
| Trigger authentication mode | | |
| Activation threshold | | |
| Target threshold | | |
| Polling interval | | |
| Cooldown and HPA stabilization | | |
| Min/max replicas and GPU budget | | |
| TrueFoundry/HPA ownership | | |

Explain why `cooldownPeriod` is not a general substitute for HPA scale-down stabilization.

## 5. Control-loop evidence

| Stage | Time | Expected | Actual |
|---|---|---|---|
| Synthetic demand begins | | | |
| Metric sample visible | | | |
| KEDA trigger Active | | | |
| HPA desired replicas increase | | | |
| Deployment creates Pod | | | |
| GPU scheduler places Pod | | | |
| Image/model warmup complete | | | |
| Ready EndpointSlice updated | | | |
| Client SLO recovered | | | |

Record desired, current and Ready separately. An HPA recommendation without Ready GPU replicas is not usable capacity.

## 6. Optional bounded staging exercise [LAB-MUTATE]

1. Verify isolated namespace, spare GPU quota, SLO, maximum spend and rollback.
2. Confirm there is no competing HPA/TrueFoundry replica controller.
3. Validate real Prometheus query and auth without exposing credentials.
4. Deploy the Chapter 8.3 example **only after replacing hypothetical values** and validating the installed KEDA version.
5. Generate bounded synthetic demand and record the control-loop timeline.
6. Stop for SLO breach, rising 429/5xx, prolonged queue, GPU Pending or VRAM pressure.
7. Reduce demand; observe stabilization, graceful stream drain and replica convergence.
8. Test a disposable lab-only metrics outage if separately approved; verify fallback behavior.
9. Restore the known-good ScaledObject/TrueFoundry release and remove only lab-owned resources.

If KEDA or approved capacity is unavailable, mark the live exercise Not run.

## 7. Failure tabletop

| ID | Scenario | LPHL | FFT | Evidence | Safe response |
|---|---|---|---|---|---|
| K1 | KEDA missing | | | | |
| K2 | ScaledObject Not Ready | | | | |
| K3 | Wrong scale target | | | | |
| K4 | Prometheus unreachable | | | | |
| K5 | Trigger auth denied | | | | |
| K6 | Multiple query series | | | | |
| K7 | Missing/stale samples | | | | |
| K8 | Wrong units | | | | |
| K9 | HPA absent | | | | |
| K10 | Competing HPA | | | | |
| K11 | Max replicas too low | | | | |
| K12 | GPU Pods Pending | | | | |
| K13 | GPU quota exhausted | | | | |
| K14 | Slow model warmup | | | | |
| K15 | Queue persists | | | | |
| K16 | Replica oscillation | | | | |
| K17 | Scale-down stream drops | | | | |
| K18 | Scale-to-zero request loss | | | | |
| K19 | Metrics fallback unsafe | | | | |
| K20 | Cost without SLO gain | | | | |

LPHL = Lowest Proven Healthy Layer; FFT = First Failed Transition. Mark tabletop-only work accurately.

## 8. Acceptance gates

| Gate | Pass / Fail / Not run | Evidence |
|---|---|---|
| A1 Controller ownership | | |
| A2 Trigger validity and auth | | |
| A3 KEDA/HPA policy semantics | | |
| A4 GPU capacity/cold start | | |
| A5 Safe scale-down | | |
| A6 Bounded staging test | | |
| A7 K1–K20 classification | | |
| A8 Rollback and recovery SLO | | |

## 9. Final evidence report

```text
Date/time and approval:
Workspace/cluster/namespace:
Model/image revision and GPU SKU:
KEDA/ScaledObject/HPA and controller ownership:
Trigger query/units/auth/freshness:
Activation/threshold, polling/cooldown:
Desired/current/Ready replicas:
GPU scheduling and cold-start timings:
TTFT/ITL, queue, KV, 429/5xx, stream completion:
K1–K20 LPHL/FFT classifications:
Rollback release and cleanup:
A1–A8 results and unexecuted gates:
```

**Next:** Chapter 8.4 — GPU Node Autoscaling and Cluster Capacity.
