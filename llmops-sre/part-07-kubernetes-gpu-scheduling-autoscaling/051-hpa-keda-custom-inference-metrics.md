# Chapter 051 — HPA, KEDA and Custom Inference Metrics

**Part 07 — Kubernetes GPU Scheduling and Autoscaling | Chapter 051/100**

> **Status:** Written tutorial published. Live labs, independent technical/security review and operational acceptance remain pending. Autoscaler API versions, KEDA triggers, Prometheus adapters and model-server metrics depend on installed versions. Do not enable autoscaling for production inference without bounded staging validation and change approval.

## Learning objectives

Design GPU inference autoscaling using Kubernetes HPA, KEDA and custom/external metrics; distinguish Pod scaling from GPU node provisioning; choose leading indicators of model saturation; protect streaming requests, VRAM and GPU budgets; diagnose metric pipeline and scaling loops; define runbooks and acceptance gates.

## 1. End-to-end architecture

```text
Gateway request → model serving (vLLM/Triton/Ray/KServe)
                         │
                request/queue/token telemetry
                         │
          Prometheus / custom metrics adapter
                         │
             HPA or KEDA ScaledObject
                         │
           Deployment/compatible scale target
                         │
                 additional GPU Pods
                         │
               Kubernetes scheduler
                         │
         GPU node provisioning (CA/Karpenter)
                         │
         driver/plugin + image/model warmup
                         │
             ready serving replica / SLO
```

HPA/KEDA scale replicas of compatible targets; Cluster Autoscaler/Karpenter provision nodes. Neither guarantees immediate GPU capacity, driver readiness or model warmup.

## 2. Why CPU utilization is often misleading

LLM inference may be GPU-bound, KV-cache-bound, memory-bound or constrained by prefill/decode scheduling. CPU utilization can remain low while TTFT p99 or queue wait rises. GPU utilization alone may also be misleading under memory pressure or long-context requests.

## 3. Metric selection

| Metric | Signal | Risk |
|---|---|---|
| waiting requests | queue pressure | stale or per-replica aggregation |
| active sequences | concurrency | not token-weighted |
| queue wait p95 | admission saturation | histogram/window lag |
| TTFT p95/p99 | user experience | lagging response indicator |
| TPOT p95 | decode saturation | streaming semantics |
| KV-cache utilization | memory headroom | engine-specific metric |
| input tokens/sec | prefill demand | mixed prompt lengths |
| output tokens/sec | decode demand | generation variability |
| GPU memory | capacity risk | not direct traffic demand |
| GPU utilization | device busy | misleading for memory bottlenecks |
| terminal-valid completions/sec | useful throughput | delayed signal |

Prefer a validated, bounded leading indicator such as queue backlog per ready replica, with SLO and memory safeguards.

## 4. HPA fundamentals

HorizontalPodAutoscaler targets scalable Kubernetes resources using the scale subresource. `autoscaling/v2` supports resource, Pods, Object and External metrics. Verify API availability, custom metrics adapter and target compatibility.

```bash
kubectl api-resources | grep -Ei 'horizontalpodautoscaler|scaledobject'
kubectl get hpa -A
kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1 2>/dev/null || true
kubectl get --raw /apis/external.metrics.k8s.io/v1beta1 2>/dev/null || true
```

A missing API endpoint is a discovery result; do not install adapters without approval.

## 5. HPA example [DESIGN ONLY]

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: llm-inference
  namespace: approved-staging
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: llm-inference
  minReplicas: 2
  maxReplicas: 6
  metrics:
    - type: Pods
      pods:
        metric:
          name: llm_waiting_requests
        target:
          type: AverageValue
          averageValue: "4"
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Pods
          value: 1
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 600
      policies:
        - type: Pods
          value: 1
          periodSeconds: 300
```

This manifest requires a working custom metrics adapter that exposes `llm_waiting_requests` per Pod and an existing approved Deployment. It is not deployable as-is without those dependencies.

## 6. HPA scaling calculation

For a single metric, desired replicas are approximately:

```text
desiredReplicas = ceil(currentReplicas × currentMetric / targetMetric)
```

The controller applies tolerance, readiness handling, stabilization, rate limits and metric validity rules. Multiple metrics generally choose the largest recommendation, with caveats for errors and scale-down. Never use the simple formula as a guarantee of actual behavior.

Example: 3 ready replicas, average waiting requests 12, target 4 → unbounded recommendation 9; `maxReplicas: 6` caps the target, and scale-up policies may limit how fast replicas are added.

## 7. KEDA fundamentals

KEDA watches external events/metrics and creates/manages HPA behavior for supported scale targets. `ScaledObject` supports triggers and activation thresholds. KEDA can scale some supported workloads from zero, but GPU inference scale-to-zero is unsafe for strict TTFT SLOs unless cold-start latency is explicitly accepted.

## 8. KEDA example [DESIGN ONLY]

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: llm-queue
  namespace: approved-staging
spec:
  scaleTargetRef:
    name: llm-inference
  minReplicaCount: 2
  maxReplicaCount: 6
  pollingInterval: 30
  cooldownPeriod: 600
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://prometheus.monitoring.svc:9090
        metricName: llm_waiting_requests_total
        query: sum(llm_waiting_requests{namespace="approved-staging"})
        threshold: "8"
```

Verify the actual Prometheus series, namespace labels, KEDA version and scaler query semantics; this is illustrative. KEDA-managed HPA should not compete with a separately authored HPA targeting the same Deployment.

## 9. Metric cardinality and aggregation

Do not aggregate per-Pod gauges as if they were monotonically increasing counters. Gauge backlog `sum()` across ready serving replicas may be appropriate; counter rates require `rate()`. Histograms require correct `histogram_quantile()` aggregation. Validate absent series, scrape staleness, label churn and scale-to-zero edge cases.

## 10. Queue and token-aware scaling

Requests vary dramatically in prompt tokens, output tokens, context window and model complexity. A queue of ten long-context requests can be more expensive than fifty short requests. Consider predicted token work, prefill/decode demand, KV cache and model-specific goodput; cap concurrency and token admission at the gateway.

## 11. GPU capacity coupling

```text
Replica scale-out time =
  node provision (if needed)
+ GPU driver/device-plugin ready
+ image pull
+ model artifact download
+ CUDA/model warmup
+ readiness stabilization
```

Set minimum warm capacity and max replicas based on physical GPU supply, cloud quotas and cost. Autoscaling a GPU Pod into Pending does not improve throughput.

## 12. Scale-down and streaming drain

Never terminate active streams abruptly to chase utilization. Use graceful termination, endpoint removal, preStop/drain coordination, sufficient terminationGracePeriodSeconds, PDB where applicable, and model-server-aware active-request tracking. HPA scale-down stabilization reduces oscillation but is not a substitute for stream-safe shutdown.

## 13. SLO and safety envelope

Define TTFT p95/p99, TPOT, E2E latency, queue wait, 429/error rate, terminal-valid completion rate and GPU-hours/valid completion. Establish hard `maxReplicas`, cloud quota, token admission, minimum warm replicas, scale-up budgets, cooldown and abort criteria.

## 14. Safe-read operational inspection

```bash
date -u
kubectl config current-context
kubectl get hpa -A
kubectl get deployments -A
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -100
kubectl get crd | grep -Ei 'scaledobjects|keda'
kubectl -n APPROVED_NS describe hpa APPROVED_HPA
```

Only inspect approved namespaces and avoid dumping sensitive environment variables.

## 15. Troubleshooting decision tree

```text
Inference SLO degrades?
 ├─ no metric → exporter/adapter/KEDA/Prometheus
 ├─ metric stale → scrape/query/window
 ├─ HPA target unknown → API metric mapping
 ├─ desired replicas unchanged → thresholds/stabilization
 ├─ replicas Pending → GPU allocatable/node autoscaler
 ├─ replicas starting slowly → image/model warmup
 ├─ scale oscillation → query noise/cooldown
 ├─ scale-down drops streams → drain semantics
 ├─ token saturation despite replicas → gateway fairness
 └─ GPU cost spikes → maxReplicas/warm pool/capacity
```

## 16. Twenty-four production failure scenarios

| ID | Failure | Evidence |
|---|---|---|
| F01 | CPU metric misses GPU saturation | SLO |
| F02 | custom metrics adapter missing | API discovery |
| F03 | external metrics API unavailable | HPA conditions |
| F04 | KEDA controller absent | CRD/Pods |
| F05 | KEDA trigger query invalid | scaler logs |
| F06 | Prometheus series missing | query |
| F07 | stale scrape data | freshness |
| F08 | counter used as gauge | query audit |
| F09 | histogram aggregated incorrectly | p95 validation |
| F10 | label cardinality explosion | TSDB |
| F11 | metric per-Pod mapping incorrect | adapter |
| F12 | competing HPAs target same Deployment | owner audit |
| F13 | minReplicas too low for TTFT | cold start |
| F14 | maxReplicas exceeds GPU capacity | Pending |
| F15 | GPU quota prevents scale-out | admission |
| F16 | node provisioning too slow | autoscaler |
| F17 | model download stalls warmup | readiness |
| F18 | scale-up rate too conservative | backlog |
| F19 | scale-down too aggressive | streaming errors |
| F20 | HPA oscillates under burst | replica history |
| F21 | long-context token work underestimated | token metrics |
| F22 | KV-cache pressure not captured | VRAM |
| F23 | rollback leaves conflicting autoscaler | GitOps diff |
| F24 | idle GPU replicas inflate cost | FinOps |

## 17. Sixteen operational runbooks

1. Inventory HPA/KEDA and scale targets.
2. Validate model-server inference metrics.
3. Restore custom/external metrics API.
4. Recover KEDA scaler/controller health.
5. Repair Prometheus query and series freshness.
6. Validate gauges, counters and histograms.
7. Recover per-Pod metric mapping.
8. Remove competing autoscaler ownership.
9. Tune minimum warm replicas and cold-start SLO.
10. Recover GPU quota and node capacity bottlenecks.
11. Diagnose image/model readiness delays.
12. Tune scale-up rate and backlog targets.
13. Protect streaming requests during scale-down.
14. Contain oscillation and KV/token saturation.
15. Reconcile autoscaling GPU-hours and cost.
16. Execute HPA/KEDA rollback and RCA.

## 18. Eight tabletop game days

1. Prometheus adapter returns no metrics during traffic burst.
2. KEDA trigger silently reports zero queue depth.
3. HPA scales Pods beyond available GPUs.
4. Cold-start model downloads exceed TTFT SLO.
5. Scale-down terminates active streaming requests.
6. Long-context burst saturates KV cache despite low queue count.
7. Conflicting HPA and KEDA controllers oscillate replicas.
8. Rollback restores app but not scaler limits.

Tabletop by default; live injection requires isolated-staging approval.

## 19. Twenty acceptance gates

G01 scope/versions; G02 HPA/KEDA inventory; G03 metric correctness; G04 adapter APIs; G05 KEDA trigger; G06 Prometheus freshness; G07 gauge/counter semantics; G08 cardinality; G09 replica calculation; G10 min/max bounds; G11 GPU quota; G12 node provisioning; G13 model warmup; G14 stream-safe drain; G15 token/KV fairness; G16 TTFT/TPOT SLO; G17 rollback; G18 FinOps; G19 game-day evidence; G20 independent review.

See [companion lab](../labs/051-hpa-keda-custom-inference-metrics-check.md). All gates remain pending until independently evidenced.

**Next: Chapter 052 — Cluster Autoscaler, Karpenter and GPU Warmup.**
