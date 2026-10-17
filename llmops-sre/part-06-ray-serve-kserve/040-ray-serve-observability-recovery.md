# Chapter 040 — Ray Serve Observability and Recovery

**Part 06 — Ray Serve and KServe | Chapter 040 of 100**

> **Publication:** Written tutorial and companion lab published. No live labs executed; technical/security review and operational acceptance pending. Ray/KubeRay metric names, status APIs and recovery semantics vary by version. Verify the pinned release.

## Objectives and outcomes

Build a multi-layer observability model for Ray Serve inference, diagnose failure domains, correlate request-level SLOs with Ray actors and Kubernetes GPUs, implement actionable alerts, recover from failed deployments, and validate full-stack rollback. Practice with safe-read commands and a CPU-only synthetic telemetry lab. This chapter integrates Chapters 036–039.

## 1. Architecture and telemetry boundaries

```text
Client synthetic probes (E2E, TTFT, TPOT, terminal validity)
             │
API gateway (TLS/auth, rate limits, retries, tenant fairness)
             │
Ray Serve proxy (routes, request errors, latency)
             │
Serve application / deployment router (pending, in-flight)
             │
Replica actors (startup, inference, cancellation, errors)
             │
Ray Core head/workers (actors, tasks, object store, nodes)
             │
KubeRay operator (RayService reconciliation, desired/ready)
             │
Kubernetes nodes / GPU plugin / CUDA / DCGM
             │
Prometheus, tracing, alerting, logs, SLO, FinOps
```

Ray head health is not the same as Serve application health. A ready replica is not proof of valid model output or complete streaming.

## 2. Golden signals and evidence ownership

| Layer | Golden signals | Owner |
|---|---|---|
| client | E2E p95/p99, TTFT, TPOT, terminal completion | application |
| gateway | 4xx/5xx, 429, retries, admission, upstream timeout | edge |
| Serve proxy | route availability, request latency, error rate | serving |
| deployment | pending/ongoing requests, replica health, queue delay | serving |
| Ray Core | actor restarts, node liveness, object-store memory | platform |
| KubeRay | reconciliation, head/worker status | platform |
| GPU | per-device VRAM, utilization, ECC/Xid, throttling | GPU SRE |
| FinOps | GPU-hours, idle capacity, valid completions | FinOps |

Never infer zero traffic from a missing metric. Distinguish absent telemetry, zero-valued samples and stale series.

## 3. SLO definitions

Define success as an authenticated request with a **valid terminal completion**, not merely HTTP 200. Set workload-specific targets for availability, TTFT p95/p99, TPOT/ITL, E2E, queue time, error-budget burn and fairness. For streaming, count aborted and incomplete responses explicitly.

```text
availability = valid_terminal_completions / eligible_requests
error_budget = (1 - SLO_target) × eligible_requests
goodput = valid_completions meeting latency/quality policy per second
gpu_cost_per_valid_completion = allocated_GPU_cost / valid_completions
```

These are policy definitions: exclude requests only according to a documented, consistently applied eligibility policy.

## 4. Metrics design and units

Prefer Prometheus counters for monotonic events, gauges for current queue/replica states and histograms for latency distributions. Exporter metric names and units must be inspected for the installed Ray version. Use histogram buckets with `histogram_quantile` only if actual buckets exist and aggregation labels are correct. Do not call a cumulative duration counter a p99.

Example PromQL templates use **custom application instrumentation**, not assumed Ray-native metric names:

```promql
sum(rate(inference_requests_total{outcome="valid"}[5m]))
/
clamp_min(sum(rate(inference_requests_total{eligible="true"}[5m])), 1e-9)
```

```promql
histogram_quantile(
  0.99,
  sum by (le, model) (rate(inference_e2e_seconds_bucket[5m]))
)
```

Validate denominator labeling: a single event counter with `eligible` and `outcome` must emit consistent series. Prefer one clearly defined request counter for the denominator and a separately defined valid-completion counter for the numerator; document any mismatch. Do not aggregate incompatible model or tenant cohorts into a misleading p99.

## 5. Tracing and correlation

Propagate a generated request/correlation ID across gateway, Serve proxy, deployment handles, replica actors and backend calls. Trace queue wait, model prefill, decode, response serialization and terminal completion where instrumentation permits. Use sampled traces, bounded cardinality and redaction. Never log prompts, completions, access tokens or PHI.

## 6. Log collection and retention

Capture timestamp (UTC), immutable release ID, RayService/cluster/Pod/replica identifiers, model revision, failure class and sanitized request ID. Collect Ray controller/proxy/replica logs separately. Set retention and access controls. Treat logs as untrusted and do not expose Ray dashboard or internal management APIs to public networks.

## 7. Read-only Kubernetes triage

```bash
date -u
kubectl config current-context
kubectl get crd | grep -i ray
kubectl get rayservices,rayclusters -A
kubectl get pods,svc -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -100
```

For an authorized namespace:

```bash
kubectl -n RAY_NAMESPACE describe rayservice APPROVED_RAYSERVICE
kubectl -n RAY_NAMESPACE get pods -o wide
kubectl -n RAY_NAMESPACE get endpointslices
kubectl -n RAY_NAMESPACE logs deployment/APPROVED_OPERATOR_DEPLOYMENT --since=15m --tail=100
```

The operator Deployment may live in a separate namespace and may use another name. Discover it first. If Ray CRDs are absent, stop; do not install an operator without approval.

## 8. GPU evidence

On an authorized GPU host:

```bash
nvidia-smi
nvidia-smi topo -m
nvidia-smi --query-gpu=index,name,uuid,memory.total,memory.used,utilization.gpu,temperature.gpu --format=csv
```

Compare GPU UUID, Kubernetes device-plugin allocation, Ray logical reservations and actual model/KV VRAM. Capture DCGM and node events where available. Do not assume utilization is a proxy for SLO-qualified goodput.

## 9. Health model

```text
KubeRay reconciled?
  → Ray head and workers alive?
  → Serve application deployed?
  → proxy route reachable?
  → replica initialized?
  → model artifacts/tokenizer compatible?
  → synthetic golden inference valid?
  → terminal stream complete?
  → latency SLO met?
```

A failed stage determines the next investigation. Maintain a synthetic canary independent of the user traffic path when possible.

## 10. Alerts and escalation

| Alert | Detection | First response |
|---|---|---|
| terminal validity burn | client synthetic + real eligible requests | isolate model/route |
| TTFT p99 regression | histogram/client traces | inspect queue/prefill |
| TPOT regression | decode cohorts | inspect GPU/KV |
| replica unavailable | Serve status | inspect actor startup |
| pending actors | Ray scheduling | inspect GPU quota/placement |
| Ray head unavailable | Ray/KubeRay | inspect control plane |
| GPU VRAM near limit | per-device GPU telemetry | reduce approved load |
| telemetry stale | scrape freshness | restore monitoring |
| GPU-hours spike | billing/capacity | investigate idle workers |

Use multi-window burn alerts for high-severity SLOs, with explicit traffic floors to avoid noisy low-volume alerts. Document on-call owner and escalation.

## 11. Incident workflow

1. Assign incident commander and record UTC onset, severity and affected models.
2. Compare last-known-good immutable release with active RayService, model, tokenizer and gateway policy.
3. Validate client symptoms and synthetic terminal completion.
4. Locate first failing layer via health chain and metrics.
5. Preserve sanitized logs, events, trace IDs, GPU metrics and autoscaler decisions.
6. Contain with approved admission limits, routing or full-stack rollback.
7. Verify recovery through golden fixtures, streaming, SLO, GPU VRAM and cost.
8. Complete RCA, corrective actions, owner and review.

Never restart all GPU replicas or unload production models without change control.

## 12. Recovery patterns

**Replica crash:** determine actor exception, OOM or worker loss; validate restart policy, state loss and client retry contract.

**Ray worker failure:** check node readiness, placement-group feasibility, GPU quota and pending actors. Validate distributed rank recovery separately.

**Ray head outage:** inspect operator reconciliation and cluster recovery policy; actor and request state may not survive. Do not assume transparent failover.

**Model startup failure:** verify immutable artifact digests, registry access, CUDA/driver and VRAM; revert entire release bundle.

**Queue overload:** enforce bounded pending requests, admission and retry budgets; scaling is delayed and not a substitute for backpressure.

**Metrics outage:** maintain client synthetic probes and platform events as alternate evidence while restoring exporters.

## 13. Full-stack rollback

```text
SLO regression
 → freeze nonessential changes
 → identify known-good Ray/KubeRay/Serve release
 → validate model + tokenizer + backend + gateway digests
 → approve rollback and route/drain strategy
 → restore complete immutable bundle
 → check RayService and Serve readiness
 → run synthetic golden and terminal streaming
 → observe SLO, GPU/queue and cost
 → document rollback evidence
```

Kubernetes Deployment rollback alone may leave mutable model artifacts or Serve config mismatched.

## 14. Disaster recovery and evidence

For regional failure, define multi-region inference routing, immutable model artifact availability, quotas, secrets distribution, warm GPU capacity and tested failback. Unlike a database, stateless inference generally focuses on restoration of valid completions; persistent conversation state or external vector stores have their own RPO/RTO requirements.

## 15. FinOps and reliability tradeoffs

Record GPU-hours, provisioned versus used GPU capacity, model cold-start waste, retry amplification, throughput, goodput and cost per valid completion. Avoid rightsizing based only on average GPU utilization when burst SLOs need warm capacity.

## 16. Twenty-four failure scenarios

| ID | Failure | Primary evidence |
|---|---|---|
| F01 | missing client terminal telemetry | synthetic validator |
| F02 | incorrect latency histogram units | metric HELP/TYPE |
| F03 | stale scrape interpreted as zero | freshness |
| F04 | high-cardinality tenant labels | TSDB cardinality |
| F05 | raw prompt leakage in traces | privacy audit |
| F06 | proxy route 404 | route map |
| F07 | proxy 503 | endpoints/readiness |
| F08 | Serve controller reconciliation stalled | status/logs |
| F09 | replica startup crash | actor exception |
| F10 | model artifact mismatch | digests |
| F11 | tokenizer drift | golden fixtures |
| F12 | Ray head outage | head health |
| F13 | GPU worker Pending | events/quota |
| F14 | Ray placement group infeasible | scheduler |
| F15 | CUDA OOM | GPU VRAM |
| F16 | KV exhaustion | allocator |
| F17 | queue p99 regression | pending/TTFT |
| F18 | retry storm | gateway retries |
| F19 | scale-down truncates streams | terminal validator |
| F20 | actor restart loses state | request trace |
| F21 | distributed rank hangs | NCCL/ranks |
| F22 | dashboard/Jobs API exposed | security audit |
| F23 | incomplete release rollback | digests |
| F24 | GPU cost spikes despite healthy HTTP | goodput/FinOps |

## 17. Sixteen operational runbooks

1. Verify client success, terminal validity and incident scope.
2. Audit metric schemas, units and freshness.
3. Recover missing synthetic probes and tracing.
4. Diagnose proxy route and gateway failures.
5. Recover Serve application/controller reconciliation.
6. Recover replica startup and actor crashes.
7. Diagnose model/tokenizer artifact drift.
8. Recover Ray head availability.
9. Recover GPU worker and placement-group scheduling.
10. Diagnose CUDA/KV memory pressure.
11. Recover queue saturation and retry amplification.
12. Restore streaming and cancellation semantics.
13. Diagnose distributed rank/NCCL failures.
14. Contain management exposure and privacy leaks.
15. Restore SLO alerting and GPU FinOps.
16. Execute complete RayService rollback, RCA and sign-off.

## 18. Eight tabletop game days

1. Telemetry exporter fails during a TTFT regression.
2. RayService reports healthy but synthetic inference fails.
3. Model artifact changes without image revision.
4. Ray head fails during a rollout.
5. GPU worker is lost during streaming.
6. Retry storm saturates replicas while autoscaler lags.
7. Ray dashboard becomes reachable from untrusted network.
8. Kubernetes rollback restores Pods but not model/tokenizer.

Tabletop by default. Live injection requires separate isolated staging approval.

## 19. Twenty acceptance gates

| Gate | Required evidence | Status |
|---|---|---|
| G01 | Approved scope, owners and version pins | Pending |
| G02 | Client terminal-validity SLI | Pending |
| G03 | TTFT/TPOT/E2E definitions | Pending |
| G04 | Histogram and metric unit validation | Pending |
| G05 | Telemetry freshness | Pending |
| G06 | Privacy-safe traces/logs | Pending |
| G07 | Gateway/proxy health | Pending |
| G08 | Serve app/replica readiness | Pending |
| G09 | Ray head/worker health | Pending |
| G10 | GPU scheduling and VRAM | Pending |
| G11 | Model/tokenizer provenance | Pending |
| G12 | Queue/admission/retry | Pending |
| G13 | Streaming/cancellation | Pending |
| G14 | Actor/worker recovery | Pending |
| G15 | Distributed rank health | Pending |
| G16 | Security/management isolation | Pending |
| G17 | SLO burn alerts and escalation | Pending |
| G18 | Complete-release rollback | Pending |
| G19 | GPU FinOps and DR objectives | Pending |
| G20 | Independent technical/security review | Pending |

See [companion lab](../labs/040-ray-serve-observability-recovery-check.md). All gates remain pending until independently evidenced.

**Next: Chapter 041 — KServe Architecture, CRDs and InferenceServices.**
