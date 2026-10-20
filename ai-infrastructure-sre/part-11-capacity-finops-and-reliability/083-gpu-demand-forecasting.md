# Chapter 083 — GPU Demand Forecasting

**Part 11 — Capacity, FinOps and Reliability | AI Infrastructure Engineering & SRE**

**Status:** Published documentation only. Offline forecasting exercises, staging validation, security review and independent acceptance remain pending.

## 1. Mission

Forecast GPU demand for distributed training, model inference and shared Kubernetes GPU platforms while preserving reliability, quota fairness, data residency, workload deadlines and cost controls. Translate business demand into **GPU-hours, GPU concurrency, node counts, reserved/on-demand capacity, lead times, buffer capacity and budget scenarios**.

**Prerequisites:** GPU architecture, Kubernetes scheduling, Kueue/Volcano or another batch queue, DCGM/Prometheus, training/inference SLOs, node provisioning and cloud quota operations.

**Learning objectives:** build a trustworthy workload inventory, distinguish demand from observed utilization, model queue pressure and training deadlines, forecast inference capacity from peak request rates, evaluate forecast uncertainty, and create a capacity procurement and operational acceptance plan.

## 2. What is being forecast?

\`\`\`text
Business roadmap / experiments / model releases
        ↓
Training runs: GPU count × runtime × retries × parallelism
        +
Inference: requests/sec × tokens × model footprint × latency SLO
        +
Shared platform: node shape, MIG, fragmentation, quotas, headroom
        ↓
GPU-hour demand and peak concurrent GPUs
        ↓
Cloud / on-prem capacity, quotas, reservations, lead times
        ↓
Cost, reliability, risk and procurement decision
\`\`\`

**GPU-hours** measure consumption, not peak schedulability. A team requiring 128 GPUs for two hours cannot necessarily run on a fleet of 16 GPUs despite the same total GPU-hours over a day. Capacity forecasts must separately model peak simultaneous GPUs, placement constraints, GPU model and memory, multi-node gang scheduling, data locality and SLO deadlines.

## 3. Workload inventory and normalization

| Dimension | Capture | Why |
|---|---|---|
| Team/tenant | owner, quota, priority | governance and fairness |
| Workload | training, fine-tuning, batch inference, online serving | different demand shape |
| Accelerator | SKU, HBM, interconnect, MIG support | workload compatibility |
| GPU request | count and memory profile | scheduling feasibility |
| Runtime | p50/p95, retries, failures | GPU-hours and variance |
| Arrival | daily/weekly seasonality, release windows | peak concurrency |
| Deadline | completion SLO, queue SLO | burst requirements |
| Placement | region/AZ, topology, residency | constrained supply |
| Availability | planned maintenance, failures, capacity holds | usable GPUs |
| Cost | price basis, commitments, data/network/storage | total economics |

Never treat average DCGM GPU utilization as a direct measure of requested or reserved GPU capacity. GPUs can be allocated but idle because of I/O stalls, checkpointing or poor workload parallelism.

## 4. Data pipeline and quality controls

Recommended inputs:
- Kubernetes Pod and node allocatable/requested accelerator resources.
- Kueue/Volcano/job-controller pending, admitted, running and failed workload events.
- DCGM device-level utilization, memory and health where enabled.
- Training runtime, tokens/step, throughput, checkpoint duration and retry history.
- Inference requests/sec, tokens/sec, p95/p99 latency, batching and autoscaler history.
- Cloud capacity quotas, reservations, spot interruptions, procurement lead time.
- Cost allocation labels, node prices, electricity/colo costs if applicable.

Use **UTC timestamps**, immutable job IDs and versioned GPU SKUs. Reconcile missing samples, clock skew, duplicate jobs, node restarts, dynamic MIG configurations and partial scrape coverage. Keep tenant and model identifiers access-controlled; avoid high-cardinality public dashboards.

### 4.1 Minimum forecasting dataset

\`\`\`csv
date,team,workload,gpu_sku,gpu_count,runtime_hours,queued_hours,attempts,deadline_hours
2026-10-01,team-a,training,H100,8,4,1,1,12
2026-10-01,team-b,batch-inference,L40S,2,6,0.5,1,24
2026-10-02,team-a,training,H100,16,3,2,2,12
\`\`\`

The sample is synthetic; no claim is made about real workloads or prices.

## 5. Core capacity formulas

### 5.1 Training GPU-hours

\[
H_{\mathrm{training}}=\sum_j G_j T_j A_j
\]

where \`G\` = GPUs allocated, \`T\` = runtime per attempt, and \`A\` = expected number of attempts **if** runtime is representative per attempt. For variable retries, use measured total GPU-hours instead of multiplying blindly.

### 5.2 Average concurrency

\[
C_{\mathrm{avg}}=\frac{H_{\mathrm{GPU}}}{H_{\mathrm{period}}}
\]

This is **not** the peak GPU requirement. Estimate peak concurrency from arrival schedules, queue simulations or historical concurrent-GPU time series.

### 5.3 Usable fleet capacity

\[
C_{\mathrm{usable}}=C_{\mathrm{installed}}\times(1-m)\times(1-r)
\]

where \`m\` is the fraction unavailable for maintenance/failures and \`r\` is the fractional reliability headroom on the remaining fleet. This is a planning approximation; avoid double-counting overlapping holds. Topology, GPU SKU, MIG and fragmentation may further reduce usable capacity.

### 5.4 Capacity requirement

\[
C_{\mathrm{installed}} \geq
\left\lceil\frac{C_{\mathrm{peak,required}}}{(1-m)(1-r)}\right\rceil
\]

Round **up to feasible node/GPU bundles** and validate multi-node placement, quotas, region and topology. A GPU count alone does not guarantee admission of a gang-scheduled job.

## 6. Worked forecast: synthetic example

Assume 8-GPU nodes, peak demand of **60 simultaneously allocated GPUs**, maintenance/unavailability \`m=0.05\`, and reserve headroom \`r=0.15\`.

\[
C_{\mathrm{installed}} \geq
\lceil 60/(0.95\times0.85)\rceil
=\lceil74.30\rceil=75
\]

Round to **10 nodes × 8 GPUs = 80 installed GPUs**. Approximate usable capacity = \`80 × 0.95 × 0.85 = 64.6 GPUs\`. The spare capacity is a **planning buffer**, not guaranteed failover capacity for every topology.

If a 64-GPU job must start as a single gang, 80 installed GPUs might still be insufficient after node failures or fragmentation. Model intact 8-GPU node bundles and queue policy explicitly.

## 7. Demand forecasting approaches

| Method | Suitable when | Weakness |
|---|---|---|
| Moving average | stable, low-variance usage | misses launch spikes |
| Seasonal baseline | repeated weekly/monthly patterns | weak for novel workloads |
| Driver-based model | roadmap, experiments, customer adoption | relies on input assumptions |
| Quantile forecast | headroom and queue SLO planning | tail uncertainty |
| Monte Carlo simulation | stochastic arrivals, durations, failures | assumptions and computational cost |
| Queue simulation | deadlines, gang scheduling, priorities | requires accurate job distributions |
| Scenario planning | major model launch or supply shock | subjective scenarios |

**Preferred operational design:** baseline time-series forecast + driver-based launches + scenario/queue simulation + backtesting. Use p50 for budget planning and a higher quantile (e.g. p90/p95) for capacity-risk decisions only after calibration; quantile choice is a business/SLO decision, not a universal rule.

## 8. Forecast horizons and cadence

- **Daily/weekly (0–14 days):** scheduler backlog, spot interruptions, maintenance, imminent launches.
- **Monthly (1–3 months):** cloud quotas, node pool scaling, committed usage, capacity reservations.
- **Quarterly (3–12 months):** hardware procurement, region strategy, rack/power/network/storage growth.
- **Annual:** model portfolio, accelerator generation migration, datacenter supply contracts.

Forecast updates must incorporate model release calendar, new customer traffic, retraining cycles, batch backlog and decommissioned workloads. Track revisions and explain significant forecast changes.

## 9. Inference demand model

Inference GPUs are driven by **peak** requests/tokens per second, batching efficiency, model memory footprint, KV-cache size, context length, latency SLO and redundancy.

\`\`\`text
Peak request/token demand
 ÷ measured sustainable throughput per GPU at target p99
 = baseline GPU replicas
 + availability and zone-failure headroom
 + rollout / warmup buffer
\`\`\`

Do not estimate inference GPU count from training GPU-hours. Benchmark with realistic prompt/output lengths, concurrency, quantization and serving engine. Validate cold starts, model load time, GPU memory admission and failure-zone placement.

## 10. Kubernetes and scheduling implications

Check:
- requested \`nvidia.com/gpu\` and vendor-specific MIG resources against allocatable.
- pending pods due to quota, taints, topology, node selectors, extended resources or insufficient CPU/memory.
- Kueue cohort borrowing, preemption and queue wait distributions; actual behavior depends on version/configuration.
- GPU node group provisioning and cloud quota delays.
- fragmentation from mixed GPU types, MIG profiles and gang scheduling.
- planned drains, driver upgrades, GPU health quarantines and rack/AZ outages.

A forecast is invalid if it counts incompatible GPU SKUs as interchangeable or assumes multi-node collectives can run on disconnected fabric islands.

## 11. FinOps and supply scenarios

Compare:
1. **Base:** observed trend and committed launches.
2. **High demand:** larger model, launch spike, more experiments and longer contexts.
3. **Supply-constrained:** quota denial, hardware lead time, region outage or spot interruption.
4. **Efficiency:** batching, right-sizing, lower retries, improved checkpoint cadence.
5. **Reliability stress:** AZ/node loss plus reserve requirement.

Compute cost with **actual contractual rates**, reservation coverage, spot volatility, storage/network egress and idle commitments. Avoid inventing current cloud GPU prices. Cost per useful training step/token and SLO-compliant served request are more meaningful than raw cost per GPU-hour alone.

## 12. Alerts and operational thresholds

Monitor:
- projected peak GPU demand versus usable compatible supply;
- p95 queue wait and deadline miss rate;
- pending GPU requests by reason and SKU;
- inference p99 and autoscaling headroom;
- forecast error (MAE, WAPE with zero-demand safeguards) and quantile coverage;
- capacity reservation/commitment utilization;
- procurement/quota lead time and expiring commitments.

Alert when projected demand crosses a documented procurement/scale lead-time threshold. Use **burn-rate style alerts** for service SLOs and forecast-specific alerts for capacity risk; do not conflate them.

## 13. Production runbooks

**A — Sudden demand spike:** identify source, compatible supply and priority; protect serving SLO, use approved quotas/preemption, escalate capacity owner, validate queue recovery.

**B — GPU quota/capacity shortage:** distinguish cloud quota from physical capacity and scheduler constraints; request quota/reservation, use approved region fallback, communicate lead time.

**C — Gang scheduling fragmentation:** inspect full-node bundles and topology, prioritize compatible placement, stage defragmentation policies; avoid evicting production jobs.

**D — Forecast drift:** compare actual versus forecast, trace changed workload drivers and data gaps, recalibrate model, document revised assumptions and procurement implications.

**E — Reliability headroom breach:** calculate capacity under node/AZ failure, protect inference replicas and critical training deadlines, freeze risky maintenance and execute approved capacity plan.

## 14. Sixteen hands-on exercises

1. Inventory GPU SKUs, nodes, MIG profiles and failure domains.
2. Reconcile GPU requests, allocation and observed utilization.
3. Build synthetic daily GPU-hour demand series.
4. Calculate average and peak GPU concurrency.
5. Forecast moving average and weekly seasonal baseline.
6. Build roadmap/driver-based demand scenario.
7. Compute p50/p90/p95 uncertainty bands.
8. Backtest forecast MAE and WAPE safely.
9. Simulate retry/failure-driven GPU-hour inflation.
10. Model 8-GPU node rounding and topology.
11. Simulate gang scheduling and queue deadlines.
12. Model inference peak and p99-compliant replicas.
13. Apply maintenance/headroom and AZ-loss scenario.
14. Compare on-demand/reserved/spot with placeholder rates.
15. Build capacity alerts and procurement trigger runbook.
16. Publish forecast, risk register and acceptance evidence.

## 15. Twelve troubleshooting scenarios

| Symptom | First investigation |
|---|---|
| High queue wait despite low GPU util | fragmentation, resource request and admission |
| GPU-hours flat but deadline misses rise | peak concurrency and gang scheduling |
| Forecast underpredicts launch | missing business driver |
| Node pool scales but pods remain Pending | quota, taints, SKU, topology, CPU/memory |
| Inference p99 breaches with spare GPUs | memory, model load, batching, wrong SKU |
| Spot cost drops but retries rise | interruptions and checkpoint loss |
| MIG capacity appears high but job cannot start | incompatible slice sizes |
| AZ loss exceeds reserve | failure-domain headroom |
| Commitment underutilized | demand mix and reservation mismatch |
| Cloud quota approved but supply absent | physical regional capacity |
| Backtest looks perfect | leakage or overly short evaluation |
| Forecast errors spike after upgrade | instrumentation/schema change |

## 16. Acceptance gate (100 points)

| Gate | Points |
|---|---:|
| Data quality and GPU inventory | 20 |
| GPU-hour and peak concurrency calculations | 20 |
| Forecast/backtest and uncertainty | 20 |
| Scheduler/topology and reliability scenarios | 15 |
| Inference and FinOps integration | 15 |
| Runbooks, peer review and evidence | 10 |
| **Total** | **100** |

**Pass:** ≥85/100 with no critical security/privacy, unsafe capacity recommendation, unvalidated SLA assumption or missing rollback/ownership. Publication does not imply execution.

## 17. References

- [Kubernetes Resource Management](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Kubernetes Scheduling](https://kubernetes.io/docs/concepts/scheduling-eviction/)
- [Kueue Documentation](https://kueue.sigs.k8s.io/)
- [NVIDIA DCGM](https://docs.nvidia.com/datacenter/dcgm/latest/)
- [Google SRE — Handling Overload](https://sre.google/sre-book/handling-overload/)
- [Prometheus Querying](https://prometheus.io/docs/prometheus/latest/querying/basics/)

**Next:** Chapter 084 — GPU Rightsizing and Utilization.
