# Chapter 071 — GPU Utilization and Capacity SLOs

**Part 09 — GPU Observability | AI Infrastructure Engineering & SRE**

**Status:** Published documentation. Offline execution, live staging validation, security review and independent acceptance remain pending.

## 1. Objectives and prerequisites

Design measurable GPU fleet capacity and reliability objectives for Kubernetes AI workloads; separate utilization from useful work, measure schedulable versus usable capacity, account for MIG and GPU model differences, forecast saturation, and make safe scale-out and admission-control decisions.

Prerequisites: Chapters 041 (multi-cloud GPU cost), 063 (scheduling fairness), 067 (DCGM metrics), 068 (dashboards) and 070 (GPU health). Lab access: Prometheus/Grafana and Kubernetes read-only; a disposable staging environment for any test deployments.

## 2. Define what capacity means

A GPU count is not a reliable proxy for compute capacity. Fleet capacity has at least six distinct dimensions:

| Dimension | Example | SRE risk |
|---|---|---|
| Installed GPUs | 128 physical devices | Counts devices under maintenance or failed |
| Healthy GPUs | 123 pass approved health criteria | Health state may lag incident evidence |
| Allocatable GPUs | 120 advertised via Kubernetes device plugin | Advertised resources do not prove device functionality |
| Free schedulable GPUs | 24 GPUs on nodes satisfying workload constraints | Fragmentation, taints, topology and quotas |
| Effective workload capacity | 16 GPU equivalents for a specific model | GPU generations, MIG slices and memory differ |
| Useful throughput | 900 inference requests/s at target latency | Depends on model, batch, memory and SLO |

Use separate pools for GPU SKU, memory size, MIG profile, interconnect topology, cloud region and workload class. **Do not sum heterogeneous GPU utilization percentages into a single capacity measure** unless the weighting and objective are explicit.

\`\`\`text
Cloud quota / node group / GPU operator / device plugin
   → Kubernetes allocatable + placement constraints
      → eligible healthy capacity per workload class
         → admitted workloads → throughput and latency SLO
DCGM + node + scheduler + workload telemetry ──────────────┘
\`\`\`

### 2.1 Definitions

- **Demand**: workload GPU resource requests and actual queued workload needs, segmented by GPU type, MIG profile and priority.
- **Reserved**: capacity deliberately held for failures, planned maintenance, upgrades or priority workloads.
- **Usable**: independently health-verified and scheduler-eligible capacity.
- **Utilization**: measured device busy percentage over an interval; neither saturation nor application productivity by itself.
- **Saturation**: demand exceeds feasible scheduling or service throughput at acceptable SLOs; can occur at modest GPU busy % due to memory, topology, fragmentation or network limits.
- **Headroom**: additional compatible demand serviceable without breaching latency, failure-domain or recovery reserves.

## 3. SLI/SLO contract and error budgets

A capacity SLO must specify **population, unit, window, source of truth, exclusions, owner and alert action**. Illustrative starting points (not universal production targets):

| SLI | Formula / observation | Example SLO | Measurement caveat |
|---|---|---|---|
| GPU fleet availability | healthy and schedulable compatible GPU-minutes / expected eligible GPU-minutes | ≥99.5% per 30 days | Inventory denominator independent of exporter |
| High-priority job admission | jobs admitted within 5 minutes / eligible submitted jobs | ≥99% per 28 days | Account for intentional quota/invalid jobs |
| Inference serving availability | successful requests / eligible requests | ≥99.9% per 30 days | Measure at serving boundary |
| Inference latency | requests below per-model latency target / eligible requests | ≥99% per 30 days | Per-model and region, not fleet average |
| Training progress | productive step-minutes / expected scheduled step-minutes | workload-defined | Need job checkpoints and stalled-step evidence |
| Telemetry coverage | expected GPU-minutes with fresh valid samples / expected GPU-minutes | ≥99.9% per 30 days | Telemetry coverage is separate from GPU health |

**Error budget:** for a 99.5% monthly GPU availability SLO, allowed bad fraction = 0.005. At 30 days this equals 216 minutes of unavailability per single consistently eligible GPU, but fleet-wide GPU-minute accounting must be weighted by the actual expected population. Do not treat scheduled maintenance as an automatic exclusion; define exclusions in advance.

**Burn rate:** observed bad-event ratio divided by allowed bad-event ratio. For a 99.9% SLO, a 1% bad-event ratio corresponds to a 10× burn rate. Multiwindow burn-rate alerting should use validated event denominators and minimum traffic thresholds to avoid noise.

## 4. Prometheus and Kubernetes metrics

Start by discovering actual metric names, labels and types:

\`\`\`bash
kubectl get nodes -o wide
kubectl describe node <gpu-node> | sed -n '/Capacity:/,/System Info:/p'
kubectl get pods -A -o wide
kubectl get resourcequota -A
kubectl get priorityclass
kubectl get pods -A --field-selector=status.phase=Pending
# Forward approved exporter/Prometheus services to localhost, then inspect metrics.
\`\`\`

Examples of **templates** that must be adapted to actual exporter labels and availability:

\`\`\`promql
# Raw per-device busy percent, NOT a capacity SLO
DCGM_FI_DEV_GPU_UTIL

# Device utilization over one hour; only present series
avg_over_time(DCGM_FI_DEV_GPU_UTIL[1h])

# Fraction of framebuffer used, provided label sets match
100 * DCGM_FI_DEV_FB_USED / clamp_min(DCGM_FI_DEV_FB_TOTAL, 1)

# Current observed series, not expected GPU inventory
count(DCGM_FI_DEV_GPU_UTIL)

# Kubernetes node GPU allocatable by node; kube-state-metrics dependent
sum by (node) (
  kube_node_status_allocatable{resource="nvidia_com_gpu",unit="integer"}
)
\`\`\`

Kube-state-metrics resource label normalization varies with version and deployment; verify \`resource\` and \`unit\` values on actual samples. GPU Operator, MIG and device-plugin configurations may expose different resource names. Do not compare a count of MIG slices directly with a count of physical GPUs. An exporter disappearing makes an observed count shrink, which is **not** a safe independent denominator.

### 4.1 Baseline and service-level indicators

Record at least 14 days (prefer 28–30) of representative utilization, memory occupancy, power/thermal context, scheduler queue times, GPU health, throughput, latency, checkpoint durations and workload arrivals. Segment by GPU SKU, workload class, priority, region, node pool and business hours. Record p50/p95/p99 where appropriate, plus peak concurrency and uncertainty; do not use average utilization alone to size the fleet.

## 5. Scheduling and fragmentation

GPU scheduling requires compatible devices, sufficient GPU memory, NUMA/topology constraints, taints/tolerations, node affinity, Pod quotas, and sometimes RDMA or EFA placement. A cluster can have eight free GPUs across eight nodes yet fail a job needing eight GPUs on one NVLink-connected node.

For each workload class, compute:
1. **Requested devices** and memory/model requirements.
2. **Eligible node set** after labels, taints, quotas and topology.
3. **Largest feasible gang/placement** rather than aggregate free devices.
4. **Reserved failover/maintenance headroom**.
5. **Queue wait SLI** and admission failure causes.

Do not overcommit Kubernetes \`nvidia.com/gpu\` device resources by changing integer requests without an approved sharing technology and workload isolation model. MIG and time-slicing have different isolation and accounting properties.

## 6. Capacity forecasting and scale policy

A practical forecast uses scenario bands:

\`\`\`text
required_compatible_devices =
  ceil(peak_concurrent_workload_demand / measured_effective_devices_per_workload)
  + N+1_failure_reserve
  + maintenance_reserve
  + approved_growth_buffer
\`\`\`

This is a planning model, not a Kubernetes scheduler guarantee. Use benchmarked throughput and latency for inference and measured job placement for training; include cloud quotas, instance supply, boot/driver initialization time, warmup and data-loading bottlenecks.

| Scenario | Demand | Supply assumption | Action |
|---|---|---|---|
| Normal | baseline | all pools healthy | maintain headroom |
| Peak | p95/p99 demand | quota unchanged | pre-warm compatible capacity |
| Node failure | baseline | lose largest failure domain | validate N+1 reserve |
| Region capacity shortage | peak | cloud GPU allocation constrained | admission control and cross-region plan |
| GPU SKU migration | mixed | heterogeneous throughput | rebenchmark and segment SLOs |

### 6.1 Rightsizing guardrails

Never reduce GPU capacity solely because average GPU busy % is low. Before downsizing verify: pending job queue, memory saturation, batch burstiness, failover headroom, model warmup time, MIG fragmentation, training checkpoint RPO, topology constraints, contractual latency SLO and reserved maintenance windows. Any change must have a canary, rollback and measured post-change SLO.

## 7. Example SLO recording and alert logic

For an **application-defined** request counter pair (not DCGM metrics), assume \`ai_inference_requests_total{result="success|error",service="..."}\` is an actual monotonically increasing counter:

\`\`\`promql
# 5-minute error ratio; avoid divide-by-zero and alert only with enough traffic
sum by (service) (rate(ai_inference_requests_total{result="error"}[5m]))
/
clamp_min(sum by (service) (rate(ai_inference_requests_total[5m])), 1e-9)
\`\`\`

This metric is **illustrative** and must be instrumented and validated. Define eligible requests and outcomes explicitly; retries and cancellations can distort the denominator. A low-volume service needs a minimum-traffic condition. Burn-rate alerting should compare short and long windows and page only when the service's error budget is genuinely threatened.

For fleet GPU availability, build an inventory-backed SLI that emits expected eligible GPU state independent of DCGM. Validate node/GPU UUID/MIG mapping and prevent missing exporter series from being interpreted as healthy or zero utilization.

## 8. Fourteen hands-on labs

1. Inventory GPU models, memory, MIG profiles, quotas and node pools.
2. Reconcile installed, healthy, allocatable and schedulable GPUs.
3. Confirm DCGM metric types, labels and sampling cadence.
4. Build per-SKU utilization and memory baselines.
5. Measure queue wait and admission success for one workload class.
6. Identify a topology/fragmentation scenario despite aggregate free GPUs.
7. Define inventory-backed GPU availability SLI and exclusions.
8. Define an inference or training SLO with an explicit request/step denominator.
9. Calculate monthly error budget and burn rate with synthetic data.
10. Build dashboards separating utilization, saturation, queue and telemetry loss.
11. Forecast normal, peak and node-failure compatible capacity.
12. Perform an approved staging-only scale/canary exercise.
13. Tabletop an N+1 node failure and cloud GPU quota shortage.
14. Submit evidence, runbooks and a scored acceptance review.

## 9. Troubleshooting decision matrix

| Symptom | Investigation | Avoid |
|---|---|---|
| Low busy %, long queue | placement, quotas, MIG/topology | assuming idle capacity is usable |
| High busy %, healthy latency | throughput and headroom | paging solely on utilization |
| Missing GPU metrics | exporter targets vs inventory | counting missing GPU as healthy |
| Pending training jobs | gang placement, priority, RDMA | blind autoscaling |
| Capacity alert but quota blocked | cloud limits and SKU supply | repeated futile provisioning |
| Sudden SLO drop | workload mix, failure domain, telemetry freshness | changing capacity without cause |
| Cost reduction harms latency | rollback canary, restore reserve | optimizing on averages |

## 10. Runbooks

**A — Admission backlog:** verify queue age and workload priority → compatible GPU inventory → quotas/topology → cloud provisioning lead time; engage scheduler and capacity owners. Exit when eligible jobs meet admission SLO.

**B — GPU capacity shortfall:** reconcile health, allocatable, schedulable and observed utilization; forecast demand and failure reserve; request quota or capacity only after confirming placement constraints. Exit with headroom and SLO restored.

**C — False utilization signal:** inspect exporter HELP/TYPE, scrape freshness, GPU/MIG mapping, aggregation and workload activity. Exit when metrics match independent evidence; document blind spots.

**D — Scale-down safety:** validate canary, workload queue, p99 latency, checkpoint risk and N+1; abort on SLO degradation. Exit after documented acceptance and rollback readiness.

## 11. Acceptance assessment (100 points)

| Gate | Points |
|---|---:|
| Inventory and compatible-capacity reconciliation | 20 |
| SLI/SLO definitions and error budget mathematics | 20 |
| Metric validity and telemetry coverage | 15 |
| Fragmentation and scheduling analysis | 15 |
| Forecast, failure reserve and canary policy | 15 |
| Runbooks, evidence and independent review | 15 |
| **Total** | **100** |

**Pass:** ≥85/100 with no critical security, workload safety or SLO-measurement failure. Documentation publication does not establish live test success.

## 12. References

- [Google SRE Workbook — Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/)
- [Kubernetes Resource Management](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Kubernetes Device Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)
- [NVIDIA MIG User Guide](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/)
- [Prometheus PromQL](https://prometheus.io/docs/prometheus/latest/querying/basics/)
- [NVIDIA DCGM Exporter](https://github.com/NVIDIA/dcgm-exporter)

**Next:** Chapter 072 — Cluster Telemetry and Event Correlation.
