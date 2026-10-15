# Lab 7.6 — vLLM Performance Tuning, Observability and Troubleshooting Check

**Canonical:** [Chapter 7.6](../06-vllm-performance-tuning-observability-troubleshooting/06-vllm-performance-tuning-observability-troubleshooting.md)  
**Default:** [SAFE-READ]. Tuning, load tests and rollback are [LAB-MUTATE] only in approved isolated nonproduction.

## 1. Goal

Establish a vLLM SLO baseline, discover real metric names and histogram types, design dashboards and alerts, run a bounded A/B tuning exercise where approved, classify 20 failure scenarios and document recovery.

## 2. Preflight [SAFE-READ]

```bash
kubectl config current-context
kubectl -n <namespace> get deployments,pods,services -o wide
kubectl -n <namespace> get endpointslices -o wide
kubectl -n <namespace> get events --sort-by=.lastTimestamp
kubectl -n <namespace> logs <pod> --tail=150
```

Record workspace, image digest, model/tokenizer revision, GPU SKU, owner, monitoring endpoint and rollback release. Use synthetic inputs only.

## 3. Baseline worksheet

| Dimension | Baseline A | Source |
|---|---|---|
| Model/runtime/GPU | | |
| Context/concurrency/TP | | |
| Prompt/output length bands | | |
| Request arrival rate | | |
| TTFT P50/P95/P99 | | |
| ITL P50/P95/P99 | | |
| E2E P95/P99 | | |
| Prompt/output tokens/s | | |
| Running/waiting requests | | |
| KV occupancy/preemptions | | |
| GPU utilization/VRAM | | |
| 429/5xx/stream errors | | |

Document measurement boundary, units, window and SLO owner.

## 4. Prometheus metric discovery

For each required metric, identify the **actual** name, type, unit, labels and scrape target from the authorized deployment. Verify histogram bucket consistency before calculating P95/P99.

| Signal | Series name | Type | Unit | Labels | Validated? |
|---|---|---|---|---|---|
| TTFT | | | | | |
| ITL | | | | | |
| Request count/errors | | | | | |
| Prompt/output tokens | | | | | |
| Running/waiting | | | | | |
| KV/preemption | | | | | |
| GPU/VRAM | | | | | |

No invented series; mark missing instrumentation as a gap.

## 5. Dashboard and alert exercise

Design panels for TTFT/ITL/E2E, RPS/errors, token throughput, running/waiting, KV/preemptions, GPU/VRAM, readiness and release annotations. For each alert, specify threshold rationale, minimum traffic floor, sustain duration, owner and first runbook link. Avoid alerting on GPU utilization alone.

## 6. Optional controlled A/B tuning [LAB-MUTATE]

1. Obtain approval, isolated GPU capacity, fixed test duration and explicit stop thresholds.
2. Record immutable baseline A and execute synthetic golden inference.
3. Select **one supported** tuning parameter and document the hypothesis.
4. Deploy candidate B via installed-version-supported TrueFoundry workflow.
5. Warm up and verify actual model revision, readiness and correctness.
6. Run a bounded synthetic workload with the same prompt/output distribution as A.
7. Compare TTFT/ITL/E2E, throughput, queue, KV/preemptions, errors and VRAM.
8. Stop if tail latency, errors, queue or memory exceed limits.
9. Restore A through authoritative release control and verify post-recovery SLO.

| Metric | A | B | After rollback |
|---|---|---|---|
| Image/model/config revision | | | |
| TTFT P95/P99 | | | |
| ITL P95/P99 | | | |
| Output tokens/s | | | |
| Queue and KV | | | |
| GPU VRAM | | | |
| Error/stream completion | | | |

## 7. Failure tabletop

| ID | Scenario | LPHL | FFT | Evidence | Safe response |
|---|---|---|---|---|---|
| P1 | High TTFT, high queue | | | | |
| P2 | High TTFT, idle GPU | | | | |
| P3 | Long-prompt TTFT | | | | |
| P4 | High ITL, busy GPU | | | | |
| P5 | High ITL, KV pressure | | | | |
| P6 | Busy GPU, low tokens/s | | | | |
| P7 | High throughput, poor P99 | | | | |
| P8 | Queue does not drain | | | | |
| P9 | Burst 429 | | | | |
| P10 | No ready endpoints | | | | |
| P11 | Gateway 504 | | | | |
| P12 | Stream failure after 200 | | | | |
| P13 | Startup VRAM OOM | | | | |
| P14 | Concurrent KV OOM | | | | |
| P15 | Upgrade regression | | | | |
| P16 | NCCL slowdown | | | | |
| P17 | Missing TTFT metric | | | | |
| P18 | Invalid percentile query | | | | |
| P19 | Label cardinality explosion | | | | |
| P20 | Wrong model after rollback | | | | |

**LPHL:** Lowest Proven Healthy Layer. **FFT:** First Failed Transition. Tabletop is not live validation.

## 8. Acceptance gates

| Gate | Pass / Fail / Not run | Evidence |
|---|---|---|
| A1 Workload and baseline | | |
| A2 SLI/SLO definitions | | |
| A3 Metrics/PromQL validation | | |
| A4 Dashboard/release annotations | | |
| A5 Alerts and runbooks | | |
| A6 A/B test or tabletop | | |
| A7 P1–P20 classification | | |
| A8 Rollback and cleanup | | |

## 9. Evidence report

```text
Date/time, approval and workspace:
Image/model/vLLM/GPU/TP:
Workload and request distribution:
TTFT/ITL/E2E and tokens/s:
Running/waiting, KV, preemptions:
GPU/VRAM and error/stream completion:
Discovered metric names/types and queries:
Dashboard/alert owner and thresholds:
A/B hypothesis, decision and limits:
P1–P20 classification:
Lowest Proven Healthy Layer:
First Failed Transition:
Known-good rollback, SLO and cleanup:
Unexecuted gates:
```

**Next:** Chapter 7.7 — Production vLLM Deployment and Recovery Capstone.
