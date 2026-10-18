# Chapter 098 — Implement SLOs, Load Testing and Capacity Governance

**Part 12 — Final Production AI SRE Capstone · Chapter 098 of 100**  
**Status:** Published. Offline lab execution, real GPU load tests, security review and production sign-off pending.

## Outcomes
Define meaningful user-facing inference SLOs; instrument OpenTelemetry/Prometheus; design realistic token-aware load tests; quantify GPU capacity and N-1 headroom; govern cost, fairness and safe scaling; validate admission and rollback under stress.

## 1. System and measurement boundaries
```text
 synthetic / canary / production clients
                 |
       gateway, identity, quotas
                 |
       router + budget admission
                 |
      GPU inference deployment
         /       |       \
      prefill  decode  KV cache
                 |
     metrics / traces / logs
                 |
      SLO + error-budget burn
                 |
     capacity forecast / gates
```
Observe both external request success and **full stream completion**. A 200 response with an aborted SSE stream is a user-visible failure. Track token usage and model revision, not just requests per second.

## 2. Service level indicators and objectives
| SLI | Definition | Illustrative objective |
|---|---|---|
| availability | successful, fully completed eligible requests / eligible requests | >=99.9% over 30 days |
| TTFT | p95 first-token latency for eligible streaming requests | <=1.8 s |
| TPOT | p95 time per output token for sufficiently long streams | <=100 ms |
| streaming completion | streams reaching terminal completion / started streams | >=99.9% |
| tool success | correct, idempotent authorized tool outcomes / eligible calls | >=99.5% |
| retrieval freshness | approved document age / index checkpoint age | workload-specific |
| tenant fairness | admission and queue latency per tier | no unapproved starvation |
| spend | cost per 1M delivered tokens, budget utilization | approved limits |

These are **examples, not universal targets**. Segment by model, region, tenant tier, prompt-length band, input/output token mix, streaming versus non-streaming and critical route. Exclude only documented non-service events from the denominator; do not silently exclude errors, throttling or model fallback.

**Error budget:** 99.9% availability permits 0.1% unsuccessful eligible requests over the window. Use burn-rate alerts with short and long windows; correlate with deployment revision and load profile.

## 3. Instrumentation and metric hygiene
- Counters: request attempts, terminal outcomes, stream interruptions, admission rejections, retries, tool results and model fallback.
- Histograms: TTFT, TPOT, end-to-end latency, queue delay, prompt/completion tokens and retrieval latency.
- Gauges: active sequences, queue depth, GPU memory, KV-cache usage, available GPU slots and replica health.
- Cost: allocated GPU-hours, paid model/tool calls, tokens delivered, reservation and ledger reconciliation.
- Traces: gateway → router → model → retrieval/tool; propagate correlation IDs without logging raw sensitive prompts.
- Restrict high-cardinality labels such as user ID, request ID and arbitrary prompt text; put sensitive diagnostic IDs in access-controlled traces when permitted.
- Explicitly define whether a timeout, client cancel, 429 and safety refusal count toward each SLI.

## 4. Token-aware workload model
Create representative traffic mixes: short Q&A, long-context summarization, multi-turn agents, RAG retrieval, tool calls and bursty batch requests. Preserve **joint distributions** of prompt tokens, generated tokens, concurrency and arrival pattern. Report warm versus cold cache, prefix reuse, quantization, model version and max-context configuration.

A test matrix should cover:
1. Baseline steady state at expected load.
2. Burst 2×–3× with bounded duration and defined abort limits.
3. Soak for memory leaks, fragmentation and cache churn.
4. N-1 zone/GPU worker loss with traffic redistributed.
5. Model warmup, artifact cache miss and rollback.
6. Retrieval latency, tool 429 and external dependency saturation.
7. Per-tenant fairness and maliciously expensive long-context requests.
8. Regional failover with explicit RTO/RPO and safe degradation.

## 5. Capacity mathematics
Define capacity in **measured safe requests/s or tokens/s for a specified workload mix**. Never add unlike units.

```text
zone_safe_capacity = observed sustained throughput at SLO pass
N-1 capacity       = sum(zone_safe_capacity) - max(zone_safe_capacity)
headroom           = N-1 capacity - forecast peak demand
utilization        = forecast peak demand / N-1 capacity
```

Account for GPU memory, KV-cache occupancy, model size, batching efficiency, prefill/decode interference, topology, network throughput, cold starts, autoscaler delays, quotas and price. Require positive headroom with an approved reserve margin. N-1 is not achieved merely by three zones if all traffic cannot be served after the largest zone fails.

## 6. Load-testing harness design
Prefer an authorized staging endpoint with synthetic requests and a bounded traffic generator such as k6 or Locust. Capture request ID, arrival time, model revision, prompt/output token counts, first token time, completion time, stream terminal status and errors. Rate-limit the generator itself; stop on budget or security breach. Never benchmark a shared production endpoint without approval.

Illustrative non-mutating observations:
```bash
kubectl get nodes -L topology.kubernetes.io/zone
kubectl get pods,hpa,pdb -n llm-staging -o wide
kubectl top nodes
kubectl get events -n llm-staging --sort-by=.lastTimestamp
```
`kubectl top` requires metrics-server or a compatible metrics API. GPU-specific utilization requires DCGM or equivalent telemetry.

## 7. Autoscaling and admission
Scale based on validated workload signals such as pending requests, queue age, active sequences and GPU saturation. CPU alone often fails to represent GPU inference pressure. Respect minimum warm replicas, GPU quota, node provisioning latency, topology and cost. Before a queue collapses, apply bounded admission, per-tenant fair queuing, maximum tokens, retry budgets and circuit breakers. Avoid uncontrolled retries that amplify load.

## 8. Governance and release gates
- SLO definitions and exclusions approved by service owner.
- Baseline and target workloads versioned with synthetic fixtures.
- Per-zone capacity measured at user-facing SLO pass.
- N-1 and burst test passed; error-budget burn below stop threshold.
- Budget and quota checks; no unapproved fallback model or region.
- Signed model/image provenance and tenant authorization unchanged.
- Observability, dashboards, alerts and rollback validated.
- Results and uncertainties reviewed before promotion.

## 9. Troubleshooting and runbooks
| Symptom | Likely cause | Response |
|---|---|---|
| TTFT high, TPOT normal | prefill saturation or queue | bound context, improve scheduling |
| TPOT high, TTFT normal | decode bottleneck, KV pressure | inspect batching, GPU memory |
| 200 responses but failed streams | downstream disconnect or timeout | count terminal outcomes |
| 429 storm | admission pressure/retry amplification | honor retry-after, cap retries |
| GPU underutilized, latency high | CPU/tokenization, gateway or RAG | trace request path |
| OOM at peak | KV-cache/context growth | reduce concurrency safely |
| N-1 failure | insufficient zone-safe throughput | reserve capacity or cap load |
| cost per token spikes | idle GPUs, fallback, tool retries | reconcile utilization and ledger |

**RB-098-A — SLO breach:** correlate burn rate with model, region, tenant and workload band; freeze promotion and reduce overload.

**RB-098-B — Capacity shortfall:** measure token mix and per-zone safe throughput; verify quota and warmup before scaling.

**RB-098-C — Load test abort:** stop synthetic traffic on critical SLO, privacy, security or budget threshold; retain evidence.

**RB-098-D — Scale rollback:** restore previous replica/admission policy and verified model revision; measure recovery.

**RB-098-E — Forecast review:** refresh peak demand, growth, GPU lead time, unit cost, N-1 reserve and next review date.

## 10. Acceptance rubric — 100 points
| Domain | Points |
|---|---:|
| Correct SLI/SLO definitions and instrumentation | 20 |
| Representative, safe load and soak testing | 20 |
| N-1 capacity and scaling governance | 20 |
| Tenant fairness, budget and release safety | 20 |
| Troubleshooting, runbooks and evidence | 20 |
| **Total** | **100** |

Educational target >=90/100 with no critical tenant, security, safety, provenance or budget violation. Offline evaluation cannot certify real GPU capacity.

## 11. Companion lab
[Chapter 098 — Executable SLO and Capacity Governance Lab](../labs/098-slos-load-testing-capacity-governance-check.md) implements deterministic SLO arithmetic, stream outcome classification, per-zone N-1 capacity and release gates with pytest.

## References
- [Google SRE Workbook — Implementing SLOs](https://sre.google/workbook/implementing-slos/)
- [Google SRE Workbook — Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/)
- [OpenTelemetry](https://opentelemetry.io/docs/)
- [Prometheus Histograms](https://prometheus.io/docs/practices/histograms/)
- [Kubernetes Autoscaling](https://kubernetes.io/docs/concepts/workloads/autoscaling/)

**Next:** Chapter 099 — Run Compound Failure and Incident Recovery Exercises.
