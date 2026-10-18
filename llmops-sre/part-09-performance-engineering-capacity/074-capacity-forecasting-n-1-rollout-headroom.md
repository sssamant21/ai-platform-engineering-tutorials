# Chapter 074 — Capacity Forecasting, N-1 and Rollout Headroom

**Part 09 — Performance Engineering and Capacity · Chapter 074 of 100**  
**Status:** Published chapter and companion lab; execution and technical/security reviews pending.

## Objectives
Build workload-aware LLM inference capacity forecasts; quantify safe capacity, N-1 survivability, deployment surge, model warmup lag and uncertainty; define production change gates and FinOps controls.

## 1. Capacity architecture
```text
traffic history + growth + peak factor + cohort mix
                        |
               demand forecast (RPS)
                        |
        per-replica safe capacity from staging
                        |
        healthy ready replicas / failure domains
                        |
    N-1 reserve + rollout surge + warmup lead time
                        |
   forecast gap -> admission / scale / provision / rollback
```
The **safe** per-replica capacity is the throughput demonstrated at the agreed TTFT, TPOT, E2E, error and goodput SLOs—not peak benchmark throughput. Input/output token distribution, prompt cache hits, model version, quantization, GPU type and batching settings all affect the result.

## 2. Forecasting demand
A simple planning model:
```text
forecast_peak_rps = baseline_peak_rps × growth_factor × seasonality_factor
required_serving_replicas = ceil(forecast_peak_rps / safe_rps_per_replica)
```
Use p50/p95 demand forecasts or scenario bands rather than treating one number as certain. Separate planned launches, migrations, seasonal peaks, batch jobs and tenant mix changes. Cross-check RPS with token throughput and KV-cache residency; RPS alone can understate long-context capacity demand.

Example (illustrative): baseline peak 60 RPS, growth 1.25, seasonal factor 1.20 → forecast peak 90 RPS. At 15 safe RPS/replica, 6 serving replicas meet steady-state demand **only if all remain healthy**.

## 3. N-1, rollout and warmup headroom
For homogeneous replicas and a one-replica failure reserve:
```text
required_ready_replicas = ceil(forecast_peak_rps / safe_rps_per_replica) + 1
N_minus_1_capacity = max(0, ready_replicas - 1) × safe_rps_per_replica
```
If 7 replicas are ready, 6 × 15 = 90 RPS remains after one failure. This is a simplified **one-replica** N-1 model. If replicas share a GPU node, availability zone, storage dependency or model registry, a realistic failure domain may remove several at once. Model the largest credible simultaneous loss.

During rolling updates, distinguish **available old replicas**, **new replicas scheduled**, **new replicas truly ready**, `maxUnavailable`, `maxSurge`, model-load time, and quota. Surge capacity requires GPU nodes, memory and network resources; a pod stuck Pending does not count. If zero unavailable replicas are permitted, enough additional GPU slots must exist to launch a new pod before replacing the old one.

## 4. Autoscaling lead time
```text
time_to_capacity =
  signal_detection + autoscaler_decision + node_provision
  + image/artifact_transfer + model_load + warmup + routing
```
Measure each stage in staging. A 10-minute scale-up cannot protect a 30-second spike without pre-warmed capacity, queue/admission protection or another approved fallback. Alert on forecasted demand exceeding **ready** capacity, not merely desired replica count.

## 5. Forecast validation workflow
1. Collect at least several representative peak cycles; document missing data and known launches.
2. Pin workload mix, model digest, GPU SKU, serving runtime and SLO-safe RPS per replica.
3. Forecast baseline, expected and adverse scenarios; quantify uncertainty.
4. Calculate required ready replicas under normal, N-1 and largest credible failure-domain loss.
5. Validate rollout headroom against GPU quotas, cluster autoscaler latency and artifact locality.
6. Run approved staging N-1 and rolling replacement drills with actual model warmup.
7. Calculate billed GPU-hours, warm-pool idle spend, scale-out cost and cost per good token.
8. Set alerts and decision thresholds for lead time, ready capacity gap, queue age and SLO burn.

## 6. Metrics and SLOs
| Metric | Interpretation |
|---|---|
| offered_rps / input_tokens_s / output_tokens_s | workload demand |
| ready_replicas / desired_replicas / pending_gpu_pods | usable vs requested capacity |
| safe_rps_per_replica | benchmarked at service SLO |
| n_minus_1_capacity_rps | resilience margin |
| forecast_capacity_gap_rps | shortfall under scenario |
| cold_start_p95_seconds | scale-out delay |
| queue_age_p95 / TTFT_p95 / TPOT_p95 | overload impact |
| admission_reject_ratio / goodput | reliability outcome |
| billed_gpu_hours / cost_per_good_token | unit economics |

Keep label cardinality bounded and protect workload privacy. A dashboard showing GPU utilization alone is insufficient to sign off capacity.

## 7. Failure scenarios and troubleshooting
| Scenario | Diagnostic | Action |
|---|---|---|
| Forecast misses traffic spike | launches, seasonality, load-driver data | revise scenario bands and warm pool |
| N-1 drill violates SLO | per-replica safe capacity, queue, KV | increase ready headroom or reduce offered load |
| Rollout stalls | GPU quotas, maxSurge, scheduling | restore old capacity; secure surge slots |
| New replicas not ready | model transfer, compilation, warmup | investigate cold-start stages |
| Capacity appears high but goodput low | workload mix, tail latency, errors | recalculate safe capacity |
| AZ loss overwhelms cluster | correlated failure-domain placement | model multi-replica loss |
| Forecast too costly | idle replicas, standby, reserved pricing | evaluate approved cost/reliability tradeoff |

## 8. Production runbooks

### RB-074-A — Forecasted capacity deficit
**[SAFE-READ]** compare forecast peak to N-1 ready capacity, input/output token mix, autoscaler lead time and queue-age trend. **[CHANGE-CONTROL]** request approved GPU capacity, warm-pool adjustment or admission controls. Validate SLO goodput after change.

### RB-074-B — N-1 failure drill
Preflight owner authorization, failure-domain boundaries, stop thresholds and rollback. Remove one approved staging replica, measure goodput, queue age and recovery, restore capacity and retain sanitized evidence.

### RB-074-C — Rollout headroom exhaustion
Pause rollout; inspect `maxSurge`, `maxUnavailable`, GPU quotas, node readiness, model loading and pending pods. Do not terminate healthy old replicas to force an unsafe rollout without approved incident procedure.

### RB-074-D — Capacity/FinOps sign-off
Publish forecast assumptions, uncertainty bands, model and runtime pins, safe capacity curves, largest failure domain, rollout surge, scale-up delay, billed GPU-hours and rollback approval.

## 9. Security and acceptance
- Do not export customer prompts or tenant identifiers in capacity dashboards.
- Secure GPU quotas, artifact stores and model deployment permissions.
- Treat capacity changes as change-controlled; synthetic examples do not certify production.
- [ ] Forecast includes workload mix and uncertainty.
- [ ] Safe RPS derived from staging SLO tests.
- [ ] N-1 and correlated failure-domain loss evaluated.
- [ ] Rollout surge and cold-start lead time modeled.
- [ ] Spend and alerting controls approved.
- [ ] Technical/security reviews and live lab execution tracked independently.

## 10. Executable lab
[Chapter 074 — Capacity Forecasting, N-1 and Rollout Headroom Lab](../labs/074-capacity-forecasting-n-1-rollout-headroom-check.md).

## References
- [Kubernetes Deployment rolling updates](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-update-deployment)
- [Kubernetes HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [Google SRE Workbook — Handling Overload](https://sre.google/workbook/handling-overload/)
- [Google SRE Book — Capacity Planning](https://sre.google/sre-book/software-engineering-in-sre/)

**Next:** Chapter 075 — Performance and Capacity Acceptance Lab.
