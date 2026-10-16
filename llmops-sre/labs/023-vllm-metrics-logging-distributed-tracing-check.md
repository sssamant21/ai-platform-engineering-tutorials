# Chapter 023 — Hands-On Lab: vLLM Metrics, Logging and Distributed Tracing

**Part 04 | Chapter 023 | Published lab plan; not executed**

## Goal

Design privacy-safe vLLM observability, verify metric schemas, define TTFT/TPOT and complete SSE SLIs, correlate queue/KV/GPU telemetry, build SLO burn alerts and test missing-data failure modes.

## Safety

**[SAFE-READ]** approved metrics, sanitized logs and traces. **[LAB-MUTATE]** isolated staging synthetic inference and telemetry changes with explicit cost limits. **[CHANGE-CONTROL]** production scrape, alert, logging, trace, DCGM or gateway changes. Never capture PHI, raw prompts, generated output or secrets.

## Lab 0 — Preflight

Record UTC, context/namespace, vLLM image/version, model alias, exporter and Prometheus versions, OTel Collector, DCGM exporter, GPU SKU, owners, telemetry retention, approvals and rollback.

```bash
kubectl config current-context
kubectl get pods,deploy,svc -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -60
```

## Lab 1 — Golden-signal inventory [DESIGN]

| Signal | Required definition |
|---|---|
| Complete SSE | terminally valid / eligible |
| TTFT | first generated token boundary |
| TPOT/ITL | generated-token interval |
| E2E | request to completion |
| Queue | wait and running/waiting requests |
| KV | cache occupancy/preemptions |
| Gateway | 401/403/429/5xx/retries |
| GPU | HBM/utilization/XID/ECC |
| FinOps | GPU-hours / valid completion |
| Freshness | last successful scrape/log/trace |

## Lab 2 — Metric schema verification [SAFE-READ; authorized local port-forward]

For a verified service/namespace:

```bash
kubectl -n MODEL_NAMESPACE port-forward svc/APPROVED_VLLM_SERVICE 18000:8000
```

In a separate terminal, if the pinned runtime exposes /metrics:

```bash
curl --fail --silent --show-error http://127.0.0.1:18000/metrics | head -80
```

Use localhost only, obtain approval for service access, sanitize output and close the tunnel. Document actual metric names, types, units, labels and freshness; never infer names from another version.

## Lab 3 — PromQL design [DESIGN]

Using verified counters and histogram buckets, draft request rate, 5xx ratio, p95 TTFT and queue alerts. Include counter reset, histogram units, low sample volume, missing-data detection and cardinality checks. Do not deploy placeholder metric names.

## Lab 4 — Trace and logging policy [DESIGN]

Draw client → gateway → serving → scheduler → GPU. Identify spans that can be instrumented in your pinned stack, context propagation, sampling and redaction. Audit that raw prompts, responses, tokens, cookies and authorization headers are absent from standard logs and traces.

## Lab 5 — Synthetic SLO worksheet [DESIGN]

```text
target success = 99.9%
eligible = 100,000
failed/incomplete = 250
allowed failures = 100
observed success = 99.75%
burn = 2.5×
```

Distinguish intentional cancellations, 429 quotas and partial HTTP 200 streams using a documented eligibility policy. Design multi-window burn and missing-scrape alerts.

## Lab 6 — GPU correlation [SAFE-READ]

On an authorized GPU host:

```bash
nvidia-smi
nvidia-smi --query-gpu=index,name,uuid,memory.total,memory.used,utilization.gpu --format=csv
```

Compare GPU memory, utilization and DCGM freshness with runtime KV/preemption, queue, TTFT/TPOT and complete SSE. Missing DCGM fields are unknown, not zero.

## Lab 7 — Eight tabletop game days

01 metric rename; 02 stale scrape; 03 prompt in trace; 04 collector drops; 05 partial SSE miscount; 06 missing DCGM; 07 low-volume burn noise; 08 high-cardinality cost.

## Twenty-four failure checklist

F01 rename; F02 units; F03 resets; F04 stale; F05 target; F06 cardinality; F07 prompt label; F08 token logs; F09 PHI trace; F10 context; F11 sampling; F12 collector; F13 log pipeline; F14 DCGM; F15 XID; F16 TTFT boundary; F17 TPOT metadata; F18 partial SSE; F19 429; F20 queue/GPU; F21 revision; F22 low traffic; F23 missing-data; F24 telemetry cost.

## Twenty acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | Pinned schema | Pending |
| G02 | SLI definitions | Pending |
| G03 | Complete SSE | Pending |
| G04 | TTFT boundary | Pending |
| G05 | TPOT definition | Pending |
| G06 | Queue metrics | Pending |
| G07 | KV/preemption | Pending |
| G08 | GPU/DCGM | Pending |
| G09 | Scrape freshness | Pending |
| G10 | Histogram units | Pending |
| G11 | Cardinality | Pending |
| G12 | Structured logs | Pending |
| G13 | Trace propagation | Pending |
| G14 | Privacy | Pending |
| G15 | SLO burn | Pending |
| G16 | Missing-data alert | Pending |
| G17 | Safe rollout | Pending |
| G18 | Rollback | Pending |
| G19 | Telemetry FinOps | Pending |
| G20 | Independent review | Pending |

## Evidence template

```text
UTC/context/approval:
vLLM/exporter/OTel/DCGM versions:
Verified metric names/units/labels:
TTFT/TPOT/complete SSE:
Queue/KV/GPU/freshness:
Structured logs and trace redaction:
SLO burn and alert thresholds:
Telemetry cost and retention:
Game-day mode/findings:
Rollback/reviewer/gates:
```

## Cleanup

Read-only/tabletop work creates no resources. Close port-forward sessions; stop separately approved synthetic load, remove only owned telemetry resources through the authoritative controller and reconcile GPU-hours and telemetry costs. Gates remain pending until independently reviewed.

**Next: Chapter 024 — vLLM Slowness, CUDA OOM and Recovery Runbooks.**
