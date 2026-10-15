# Lab 6.6 — Model-Serving Observability, Performance and Troubleshooting

**Canonical:** [Chapter 6.6](../06-model-serving-observability-performance-troubleshooting/06-model-serving-observability-performance-troubleshooting.md)  
**Mode:** **[SAFE-READ]** by default. Load generation, alert changes, rollout and fault injection are **[LAB-MUTATE]** only in an approved isolated nonproduction TrueFoundry workspace.

## 1. Objectives

Inventory inference metrics, inspect CPU/GPU saturation, correlate traces and logs, establish a latency baseline, evaluate alerts, classify 16 incidents and document recovery with acceptance evidence.

## 2. Preflight [SAFE-READ]

```bash
kubectl config current-context
kubectl -n <namespace> get deployments,pods,services -o wide
kubectl -n <namespace> get endpointslices -o wide
kubectl -n <namespace> get events --sort-by=.lastTimestamp
```

Record approved workspace, namespace, deployment, image digest, model version, owner, telemetry endpoint and SLO. Do not expose model inputs, outputs, secrets or tokens.

## 3. Metrics inventory

| Signal | Metric/exporter | Unit | Labels | Available? |
|---|---|---|---|---|
| Eligible request RPS | | req/s | | |
| 5xx and 429 rate | | ratio | | |
| P95/P99 end-to-end | | seconds | | |
| Queue wait/depth | | s / requests | | |
| Runtime execution | | seconds | | |
| Model load/warmup | | seconds | | |
| Ready replicas/endpoints | | count | | |
| CPU/RAM/throttling | | cores/bytes | | |
| GPU utilization/VRAM | | % / bytes | | |
| Streaming completion/TTFT | | ratio / seconds | | |

Do not assume illustrative metrics from the tutorial are installed. Confirm scrape targets, histogram buckets, labels and exporter versions.

## 4. Kubernetes and GPU checks [SAFE-READ]

```bash
kubectl -n <namespace> get deployments,replicasets,pods -o wide
kubectl -n <namespace> describe pod <pod>
kubectl -n <namespace> logs <pod> --tail=150
kubectl -n <namespace> logs <pod> --previous --tail=150
kubectl -n <namespace> top pods --containers
```

Metrics Server and previous logs may be unavailable. Use the approved DCGM/GPU dashboard if deployed. Distinguish host RAM from VRAM and process CPU from GPU busy.

## 5. PromQL practice (adapt metric names)

Request rate:

```promql
sum(rate(inference_requests_total[5m]))
```

5xx ratio:

```promql
sum(rate(inference_requests_total{status_class="5xx"}[5m]))
/
clamp_min(sum(rate(inference_requests_total[5m])), 1e-9)
```

P95 latency:

```promql
histogram_quantile(0.95, sum by (le, service) (rate(inference_request_duration_seconds_bucket[5m])))
```

Record actual query output, label semantics, sample volume and gaps. For low-volume models, use appropriate minimum-request guards.

## 6. Trace and log correlation

Using approved observability access, select one synthetic or non-sensitive request and record:

```text
request_id / trace_id:
gateway and auth spans:
queue/admission span:
preprocess and model execution spans:
response/stream span:
model version and revision:
sanitized error code:
critical path duration:
```

Do not export sensitive payloads. If a span is missing, note the instrumentation gap rather than inventing latency attribution.

## 7. Optional controlled performance baseline [LAB-MUTATE]

1. Deploy an approved immutable model and image in a disposable workspace.
2. Warm the model using synthetic input and verify golden correctness.
3. Generate bounded traffic at three approved concurrency levels.
4. Capture throughput, P95/P99, queue wait, CPU/RAM, GPU/VRAM, 429 and 5xx.
5. Compare cold versus warm start and note model artifact transfer time.
6. Stop immediately on instability or exceeding approved resource/traffic limits.
7. Restore known-good revision and remove lab-owned resources.

| Load level | Concurrency | RPS | P95/P99 | Queue | CPU/GPU | Errors |
|---|---|---|---|---|---|---|
| Low | | | | | | |
| Medium | | | | | | |
| High (approved) | | | | | | |

## 8. Alert review

Review one availability SLO alert, one latency/queue alert, one zero-ready-endpoints alert and one GPU/memory diagnostic alert. Record threshold, window, traffic guard, owner, runbook and false-positive risk. Do not create production alerts without change approval.

## 9. Incident tabletop

| ID | Symptom | First investigation |
|---|---|---|
| O1 | 503 and no ready endpoints | Probes/Service |
| O2 | 504 with healthy Pods | Queue/deadline |
| O3 | P99 and CPU throttling | Preprocessing |
| O4 | P99 high, GPU idle | Network/queue/batching |
| O5 | P99 high, GPU saturated | Capacity |
| O6 | GPU OOM | VRAM/concurrency |
| O7 | Host OOMKilled | RAM/cgroup |
| O8 | Cold-start spikes | Model download/load |
| O9 | Model storage 403 | Workload identity |
| O10 | CUDA initialization failure | Device/runtime |
| O11 | Rising 429 | Admission limits |
| O12 | Retry storm | Client/gateway policy |
| O13 | HTTP success, wrong prediction | Model/schema |
| O14 | Incomplete streams | Cancellation/idle timeout |
| O15 | Missing metrics | Scrape/exporter |
| O16 | High cardinality | Instrumentation labels |

For each record **Lowest Proven Healthy Layer**, **First Failed Transition**, supporting metrics/traces/logs, owner, smallest approved remediation, rollback and validation.

## 10. Acceptance gates

| Gate | Pass / Fail / Not run | Evidence |
|---|---|---|
| G1 SLI/SLO eligibility | | |
| G2 Metrics and histogram semantics | | |
| G3 Trace/log correlation | | |
| G4 CPU/GPU memory and utilization | | |
| G5 Load/queue/latency baseline | | |
| G6 Alert ownership and burn rate | | |
| G7 Failure isolation | | |
| G8 Rollback and recovery validation | | |

## 11. Evidence report

```text
Date/time / approval:
TrueFoundry workspace / model / deployment:
Image digest / model version / revision:
Request RPS, error ratio, 429/5xx:
P95/P99, TTFT, stream completion:
Queue, batching, model execution:
CPU/RAM/throttling and GPU/VRAM:
Cold start and Ready endpoints:
Metrics query, dashboard, trace references:
Alert thresholds/windows/owner:
O1–O16 classifications:
Lowest Proven Healthy Layer:
First Failed Transition:
Approved remediation / rollback:
Post-recovery SLO and cleanup:
Unexecuted gates and follow-ups:
```

**Next:** Chapter 6.7 — Production Model-Serving Deployment and Recovery Capstone.
