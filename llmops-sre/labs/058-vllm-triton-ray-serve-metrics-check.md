# Chapter 058 — Hands-On Lab: vLLM, Triton and Ray Serve Metrics

**Part 08 | Chapter 058 | Written lab published; execution pending**

## Objective

Discover native inference metrics, validate HELP/TYPE/units and scrape targets, build a cross-platform normalization contract, compare latency and request semantics, detect metric drift and protect privacy.

## Safety and prerequisites

**[SAFE-READ]** authorized cluster/metrics endpoints. **[LAB-MUTATE]** approved isolated staging with synthetic requests. **[CHANGE-CONTROL]** production exporters, scrape configuration, recording rules and alerts. Never publish raw prompts, outputs, PHI, secrets or unbounded tenant labels.

Record cluster/context, namespace, model, vLLM/Triton/Ray versions, Prometheus/OTel versions, approval, cost ceiling, expected targets and rollback.

## Lab 1 — Kubernetes discovery [SAFE-READ]

```bash
date -u
kubectl config current-context
kubectl get pods -A -o wide
kubectl get services -A
kubectl get servicemonitors -A 2>/dev/null || true
kubectl get podmonitors -A 2>/dev/null || true
kubectl get events -A --sort-by=.lastTimestamp | tail -100
```

## Lab 2 — Native metric inspection [SAFE-READ]

Run only from an authorized location against an **approved** endpoint:

```bash
# Example vLLM port; replace with discovered endpoint
curl -fsS --max-time 10 http://127.0.0.1:8000/metrics | grep -E '^(# HELP|# TYPE|vllm:)' | head -60
# Example Triton metrics port
curl -fsS --max-time 10 http://127.0.0.1:8002/metrics | grep -E '^(# HELP|# TYPE|nv_inference_)' | head -60
# Ray port is illustrative, not a default guarantee
curl -fsS --max-time 10 http://127.0.0.1:8080/metrics | grep -E '^(# HELP|# TYPE|ray_serve_)' | head -60
```

A missing metric is a discovery finding, not permission to invent a time series.

## Lab 3 — Normalization worksheet

| Platform | Native metric | Unit/type | Canonical signal | Evidence |
|---|---|---|---|---|
| vLLM | discover | discover | TTFT | pending |
| vLLM | discover | discover | queue/KV | pending |
| Triton | discover | discover | requests | pending |
| Triton | discover | discover | queue/compute | pending |
| Ray Serve | discover | discover | request latency | pending |
| Ray Serve | discover | discover | queued/ongoing | pending |
| gateway | discover | discover | terminal-valid | pending |

Check batching, retry, stream completion, labels, cardinality and timestamp boundaries.

## Lab 4 — PromQL design [NORMALIZED METRICS ONLY]

```promql
sum by (platform, model) (rate(llm_admitted_requests_total[5m]))
```

```promql
histogram_quantile(0.95,
  sum by (platform, model, le) (
    rate(llm_first_token_latency_seconds_bucket[5m])
  )
)
```

Verify real mappings, counter resets, scrape duplication and histogram buckets before use.

## Lab 5 — SLO and privacy tabletop

Design a terminal-valid completion SLI, per-platform TTFT/E2E dashboards, scrape freshness alerts and rollback. Inspect sample labels for user IDs, request IDs, prompts and secrets without copying sensitive values.

## Eight game days

01 vLLM rename; 02 Triton units; 03 batching count; 04 Ray double count; 05 gateway retries; 06 duplicate scrapes; 07 label privacy; 08 truncated streams.

## Twenty-four failure checklist

F01 vLLM disabled; F02 rename; F03 KV missing; F04 TTFT boundary; F05 Triton endpoint; F06 units; F07 exec count; F08 ensemble; F09 Ray endpoint; F10 Ray rename; F11 router/replica; F12 latency semantics; F13 retries; F14 partial stream; F15 exporter rollout; F16 duplicate scrape; F17 buckets; F18 type; F19 cardinality; F20 privacy; F21 DCGM MIG; F22 stale scrape; F23 alert; F24 FinOps.

## Twenty acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | Environment/versions | Pending |
| G02 | Endpoint inventory | Pending |
| G03 | Scrape health | Pending |
| G04 | vLLM scheduler/KV | Pending |
| G05 | vLLM latency | Pending |
| G06 | Triton request/exec | Pending |
| G07 | Triton queue/compute units | Pending |
| G08 | Ray router/replica | Pending |
| G09 | Ray latency semantics | Pending |
| G10 | Gateway attempts | Pending |
| G11 | Terminal validity | Pending |
| G12 | Token accounting | Pending |
| G13 | Histogram accuracy | Pending |
| G14 | PromQL/dedup | Pending |
| G15 | GPU/DCGM correlation | Pending |
| G16 | Labels/privacy | Pending |
| G17 | SLO alerts | Pending |
| G18 | Rollback/schema drift | Pending |
| G19 | Telemetry FinOps | Pending |
| G20 | Independent review | Pending |

## Evidence worksheet

```text
UTC/context/approval:
Runtime versions and model digest:
Endpoint URL (internal only)/target health:
HELP/TYPE and native metric names:
Unit and request/token/attempt semantics:
vLLM queue/KV/TTFT:
Triton request/exec/queue/compute:
Ray router/replica/latency:
Gateway terminal-valid completion:
PromQL/histogram/dedup:
DCGM GPU correlation:
Privacy/cardinality/SLO alerts:
Telemetry cost:
Eight game-day findings:
Rollback/reviewer/gates:
```

## Cleanup

Read-only/tabletop exercises create no resources. Approved staging changes must restore scrape targets, rules and alerts, remove only owned synthetic workloads, verify metrics/labels and reconcile cost. No gate passes without evidence and independent review.

**Next: Chapter 059 — GPU DCGM, Node and Kubernetes Observability.**
