# Chapter 056 — Hands-On Lab: Golden Signals for LLM Inference

**Part 08 | Chapter 056 | Published lab plan; live execution pending**

## Goal

Inventory inference metrics/traces, normalize golden signals, validate latency histograms, completion semantics, queue/KV/GPU saturation and privacy-safe SLO alerts; capture review evidence.

## Safety

**[SAFE-READ]** authorized metrics/Kubernetes discovery. **[LAB-MUTATE]** approved isolated staging, synthetic prompts and bounded load. **[CHANGE-CONTROL]** production exporters, tracing, Prometheus rules, alert routing and model instrumentation. Do not collect raw prompts, completions, PHI, Secrets or unbounded tenant IDs.

## Lab 0 — Preflight

Record UTC, context, Kubernetes/runtime/Prometheus/OTel versions, model digest, gateway, metric names, histogram buckets, SLO targets, retention, approval, cost ceiling and rollback.

## Lab 1 — Telemetry inventory [SAFE-READ]

```bash
date -u
kubectl config current-context
kubectl get pods -A -o wide
kubectl get services -A
kubectl get servicemonitors -A 2>/dev/null || true
kubectl get podmonitors -A 2>/dev/null || true
kubectl get events -A --sort-by=.lastTimestamp | tail -100
```

Discover actual exporters/targets before assuming metric names.

## Lab 2 — Canonical metric mapping [DESIGN]

| Golden signal | Proposed normalized metric | Source |
|---|---|---|
| latency | llm_ttft_seconds_bucket | pending |
| latency | llm_e2e_seconds_bucket | pending |
| traffic | llm_requests_total | pending |
| traffic | llm_input_tokens_total | pending |
| traffic | llm_output_tokens_total | pending |
| errors | llm_terminal_valid_completions_total | pending |
| saturation | llm_waiting_requests | pending |
| saturation | llm_kv_cache_utilization_ratio | pending |
| saturation | GPU/DCGM memory/utilization | pending |

Names are proposed normalized metrics, not guaranteed exporter names.

## Lab 3 — PromQL design [DESIGN]

```promql
sum by (model) (rate(llm_requests_total[5m]))
```

```promql
histogram_quantile(0.95,
  sum by (model, le) (rate(llm_ttft_seconds_bucket[5m]))
)
```

```promql
sum by (model) (rate(llm_terminal_valid_completions_total[5m]))
```

Verify scrape freshness, deduplication, histogram buckets, zero traffic and actual metric mappings before executing.

## Lab 4 — Completion and cancellation taxonomy [DESIGN]

Define admitted requests, terminal-valid completions, user cancellations, deadlines, partial streams, 429, 5xx and model-invalid outputs. Ensure the SLO denominator does not silently exclude server failures.

## Lab 5 — OTel trace privacy [DESIGN]

Define gateway → queue → prefill → first token → decode → terminal spans with sanitized attributes, bounded cardinality and sampling. Verify no prompt/response/secret/PHI payloads in telemetry.

## Lab 6 — Eight game days

01 TTFT spike; 02 partial stream; 03 queue metric absent; 04 DCGM MIG labels; 05 duplicate scrape; 06 dropped error traces; 07 cardinality/PII; 08 alert mismatch.

## Twenty-four failure checklist

F01 TTFT boundary; F02 TPOT; F03 ITL; F04 E2E queue; F05 partial 200; F06 cancellation; F07 429; F08 OOM; F09 token mix; F10 queue; F11 KV; F12 GPU; F13 MIG labels; F14 duplicate scrapes; F15 counter/gauge; F16 buckets; F17 histogram; F18 cardinality; F19 privacy; F20 propagation; F21 alert noise; F22 SLO denominator; F23 runbook; F24 cost.

## Twenty acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | Scope/versions | Pending |
| G02 | Metric contract | Pending |
| G03 | TTFT | Pending |
| G04 | TPOT/ITL | Pending |
| G05 | E2E | Pending |
| G06 | Terminal validity | Pending |
| G07 | Traffic/tokens | Pending |
| G08 | 429/5xx taxonomy | Pending |
| G09 | Queue | Pending |
| G10 | KV | Pending |
| G11 | GPU/DCGM | Pending |
| G12 | Histogram accuracy | Pending |
| G13 | PromQL semantics | Pending |
| G14 | Cardinality | Pending |
| G15 | OTel propagation | Pending |
| G16 | Privacy | Pending |
| G17 | SLO/error budget | Pending |
| G18 | Alert/runbook | Pending |
| G19 | Telemetry FinOps | Pending |
| G20 | Independent review | Pending |

## Evidence worksheet

```text
UTC/context/approval:
Runtime/gateway/exporter versions:
Canonical metric mapping:
TTFT/TPOT/ITL/E2E definitions:
Terminal-valid completion denominator:
Requests/tokens/429/5xx:
Queue/KV/GPU/DCGM:
Histogram buckets/PromQL:
OTel trace propagation/privacy:
SLO/error budget/alert routing:
Telemetry cost:
Eight game-day findings:
Rollback/reviewer/gates:
```

## Cleanup

Read-only/tabletop work creates no resources. Approved staging tests must restore original exporters, recording rules, traces, alerts and retention, remove only owned synthetic workloads, validate no sensitive payloads leaked, reconcile telemetry costs and retain sanitized evidence. All gates remain pending until independent review.

**Next: Chapter 057 — TTFT, TPOT, ITL, E2E Latency and Completion.**
