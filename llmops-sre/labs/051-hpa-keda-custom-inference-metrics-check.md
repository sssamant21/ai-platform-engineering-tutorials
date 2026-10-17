# Chapter 051 — Hands-On Lab: HPA, KEDA and Custom Inference Metrics

**Part 07 | Chapter 051 | Published lab plan; live execution pending**

## Goal

Inspect inference autoscaling, validate custom metric semantics, model GPU Pod scale-out and warmup, plan stream-safe scale-down, design bounded HPA/KEDA staging tests and collect evidence.

## Guardrails

**[SAFE-READ]** authorized cluster inventory. **[LAB-MUTATE]** approved isolated staging, synthetic traffic, explicit GPU-hour cap. **[CHANGE-CONTROL]** production HPA/KEDA, adapters, Prometheus rules, node autoscaler and model-server concurrency. Do not use PHI, customer prompts, Secrets or unapproved traffic.

## Lab 0 — Preflight

Record UTC, context, Kubernetes/HPA/KEDA versions, scale target, model digest, GPU SKU, adapter/Prometheus endpoint, metric names/labels, SLO, min/max replicas, quota, cost ceiling and rollback.

## Lab 1 — Autoscaler inventory [SAFE-READ]

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

Record current/desired replicas, HPA conditions, missing metrics, target ownership and Pending GPU Pods.

## Lab 2 — Metric API discovery [SAFE-READ]

```bash
kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1 2>/dev/null || true
kubectl get --raw /apis/external.metrics.k8s.io/v1beta1 2>/dev/null || true
```

Missing APIs indicate an uninstalled/inaccessible adapter; do not install one without approval.

## Lab 3 — Scaling math [DESIGN]

Given 3 replicas, average queue depth 12 and target 4, compute `ceil(3×12/4)=9`; cap at maxReplicas 6, then apply HPA rate limits and readiness rules. Compare metric latency, warmup time and TTFT.

## Lab 4 — HPA example [DESIGN]

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
```

Requires existing approved Deployment and custom metrics adapter. Do not apply to production.

## Lab 5 — KEDA trigger design [DESIGN]

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
  triggers:
  - type: prometheus
    metadata:
      serverAddress: http://prometheus.monitoring.svc:9090
      metricName: llm_waiting_requests_total
      query: sum(llm_waiting_requests{namespace="approved-staging"})
      threshold: "8"
```

Verify metric series, KEDA version, namespace labels and target ownership. Do not install alongside a competing HPA.

## Lab 6 — Synthetic test matrix [DESIGN]

| Test | Expected evidence |
|---|---|
| baseline | stable TTFT/TPOT and replicas |
| short burst | queue leads scale-out |
| long context | token-aware pressure visible |
| missing metrics | safe degraded scaling behavior |
| GPU pool full | Pending and node autoscaler evidence |
| model cold start | readiness latency |
| active streaming scale-down | graceful completion |
| rollback | single autoscaler owner, original bounds |

## Eight tabletop game days

01 adapter outage; 02 zero metric; 03 GPU capacity; 04 model warmup; 05 stream termination; 06 KV pressure; 07 competing controllers; 08 rollback drift.

## Twenty-four failure checklist

F01 CPU mismatch; F02 adapter; F03 external API; F04 KEDA absent; F05 trigger; F06 missing series; F07 stale scrape; F08 gauge/counter; F09 histogram; F10 cardinality; F11 Pod mapping; F12 competing HPA; F13 min warm; F14 max GPU; F15 quota; F16 node scale; F17 model startup; F18 scale-up; F19 scale-down; F20 oscillation; F21 long context; F22 KV; F23 rollback; F24 FinOps.

## Twenty acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | Scope/versions | Pending |
| G02 | HPA/KEDA inventory | Pending |
| G03 | Metric correctness | Pending |
| G04 | Adapter APIs | Pending |
| G05 | KEDA trigger | Pending |
| G06 | Prometheus freshness | Pending |
| G07 | Gauge/counter semantics | Pending |
| G08 | Cardinality | Pending |
| G09 | Replica calculation | Pending |
| G10 | Min/max bounds | Pending |
| G11 | GPU quota | Pending |
| G12 | Node provisioning | Pending |
| G13 | Model warmup | Pending |
| G14 | Stream-safe drain | Pending |
| G15 | Token/KV fairness | Pending |
| G16 | TTFT/TPOT SLO | Pending |
| G17 | Rollback | Pending |
| G18 | FinOps | Pending |
| G19 | Game-day evidence | Pending |
| G20 | Independent review | Pending |

## Evidence worksheet

```text
UTC/context/approval:
HPA/KEDA/adapter versions:
Scale target/owner:
Metric query/type/freshness:
Queue/TTFT/TPOT/KV:
Current/desired/min/max replicas:
GPU allocatable/quota/node autoscaler:
Cold-start and readiness:
Stream-safe drain:
Synthetic traffic and abort:
GPU-hours/cost:
Eight game-day findings:
Rollback/reviewer/gates:
```

## Cleanup

Read-only/tabletop work creates no resources. Approved staging tests must restore original HPA/KEDA and adapter configuration, remove only owned synthetic workloads, validate replica and GPU-node baselines, reconcile GPU-hours and retain sanitized evidence. All gates remain pending until independent review.

**Next: Chapter 052 — Cluster Autoscaler, Karpenter and GPU Warmup.**
