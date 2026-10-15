# Chapter 11.5 — Hands-on Lab: GPU, vLLM and Inference Performance Observability

**Part 11 | Approved staging GPU | Live execution pending**

## Mission and safety

Measure synthetic inference traffic end to end, correlate vLLM scheduler and KV cache with GPU metrics, distinguish bottlenecks, and validate dashboards, alerts and recovery. Use synthetic prompts only. **[SAFE-READ]** inventory; **[LAB-MUTATE]** bounded staging load; **[CHANGE-CONTROL]** production changes. Never run stress tests on production GPU pools without approval.

## Lab 0 — Inventory [SAFE-READ]

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get daemonsets -A
kubectl get events -A --sort-by=.lastTimestamp | tail -40
kubectl top nodes 2>/dev/null || true
```

Record TrueFoundry/vLLM versions, model alias and approved digest, GPU type/count/MIG, driver/CUDA, device plugin, exporter, metric names, model resource limits, request endpoint and on-call owner. Do not copy model weights or access tokens into evidence.

## Lab 1 — Verify telemetry sources

Identify gateway RED metrics, vLLM `/metrics` where exposed, DCGM exporter, Pod/container resources and traces. Check scrape health, units, labels, sampling and retention. Document missing version-dependent metrics as gaps, not zeros.

## Lab 2 — Baseline synthetic inference [LAB-MUTATE]

Use an approved low-volume harness with fixed synthetic prompt lengths and bounded output token limits. Measure successful requests, TTFT, TPOT, end-to-end duration, tokens/sec, queue depth, KV cache and GPU utilization/memory. Capture p50/p95/p99 only with sufficient samples and explicit sample size.

## Lab 3 — Bounded concurrency ramp

Increase concurrency in small preapproved steps, record offered vs accepted load, errors, queue, latency and throughput. Stop on defined safety thresholds or unexpected GPU faults. Plot latency vs throughput and identify saturation knee. Compare only equal model/GPU/configuration and traffic mix.

## Lab 4 — Troubleshooting tabletop

Rehearse CUDA unavailable, GPU OOM, Pod OOMKilled, model loading timeout, queue growth, TTFT spike, preemption and streaming disconnect. For unsafe scenarios, use logs/metrics from approved historical incidents or tabletop, not destructive injections.

## Lab 5 — Dashboards and alert validation

Confirm panels for TTFT/TPOT, queue, KV cache, tokens/sec, GPU memory/utilization, XID/ECC, Pod health, gateway errors and deployment revision. Validate one low-impact test alert and runbook routing. Ensure no raw prompt or user ID label leakage.

## Lab 6 — Recovery and cleanup

Remove lab traffic/workloads, restore original config, verify readiness and baseline performance, inspect exporter/collector health and record UTC evidence and reviewer sign-off.

## Thirty-two failure scenarios

| # | Scenario | Diagnostic focus |
|---|---|---|
| 01 | GPU node NotReady | Node health |
| 02 | GPU allocatable missing | Device plugin |
| 03 | GPU Pod Pending | Scheduling |
| 04 | NVIDIA runtime absent | Container runtime |
| 05 | CUDA driver mismatch | Compatibility |
| 06 | `nvidia-smi` unavailable | Permissions/tools |
| 07 | DCGM exporter unscheduled | DaemonSet |
| 08 | DCGM metrics missing | Scrape/field set |
| 09 | MIG metrics misattributed | Device topology |
| 10 | GPU utilization unexpectedly low | Demand/CPU |
| 11 | GPU memory near full | Model/KV cache |
| 12 | CUDA out of memory | Runtime memory |
| 13 | Kubernetes OOMKilled | Container memory |
| 14 | GPU thermal throttle | Cooling/power |
| 15 | GPU XID event | Driver/device |
| 16 | GPU ECC error | Hardware |
| 17 | Model load timeout | Storage/size |
| 18 | Model digest changed | Release drift |
| 19 | Tokenizer CPU bottleneck | Prefill |
| 20 | Input token lengths increase | Traffic mix |
| 21 | TTFT p95 regression | Prefill/queue |
| 22 | TPOT regression | Decode |
| 23 | Queue depth grows | Saturation |
| 24 | KV cache pressure | Scheduler |
| 25 | Preemption rises | Memory |
| 26 | Throughput plateaus | Capacity knee |
| 27 | Gateway 5xx increase | Serving path |
| 28 | Streaming disconnect | Timeout/cancel |
| 29 | vLLM metric renamed | Version drift |
| 30 | Histogram units incorrect | Query |
| 31 | Alert storm on high GPU utilization | Tuning |
| 32 | Synthetic load impacts shared tenant | Isolation |

For each scenario: expected/actual, environment, UTC timestamp, safe metric/log/trace evidence, root-cause hypothesis, mitigation, rollback and reviewer. Mark unsafe cases tabletop.

## Ten acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G1 | Approved staging scope, model/GPU version inventory | Pending |
| G2 | Gateway/vLLM/DCGM metric dictionary verified | Pending |
| G3 | GPU health and Pod attribution validated | Pending |
| G4 | TTFT/TPOT/token throughput measured | Pending |
| G5 | Queue/KV cache and concurrency metrics correlated | Pending |
| G6 | Bounded load curve and stop conditions documented | Pending |
| G7 | GPU fault and runtime OOM detection rehearsed | Pending |
| G8 | Dashboards/alert routing and privacy verified | Pending |
| G9 | Recovery, rollback and cleanup validated | Pending |
| G10 | SRE review and sign-off recorded | Pending |

**Pass** requires timestamped evidence and reviewer approval; blocked/unexecuted tests remain pending.

**Next:** Chapter 11.6 — Dashboards, SLIs, SLOs, Alerting and Error Budgets.
