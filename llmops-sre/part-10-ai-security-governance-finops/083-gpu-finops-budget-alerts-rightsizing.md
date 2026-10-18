# Chapter 083 — GPU FinOps, Budget Alerts and Rightsizing

**Part 10 — AI Security, Governance and FinOps · Chapter 083 of 100**  
**Status:** Chapter and offline executable lab published. Live GPU benchmarking, cloud billing reconciliation and independent technical/FinOps review pending.

## Objectives
Build GPU inference cost visibility, allocate shared spending, forecast budgets, alert on runaway usage, and rightsize serving deployments without violating latency, throughput, quality or high-availability SLOs.

## 1. Reference architecture
```text
client -> AI gateway -> model router -> GPU inference pools
              |             |               |
       token/tenant usage  model ID      GPU metrics
              |             |               |
              +------ metering pipeline -----+
                             |
                normalized usage ledger
                             |
              cost allocation + unit economics
                             |
           budgets / forecasts / anomaly alerts
                             |
        rightsizing proposal -> canary -> rollback
```
Reconcile inference-layer metrics with infrastructure billing. Do not assume that utilization alone measures efficiency: KV cache, prefill/decode balance, batch size, latency tails, reserved capacity and idle headroom all matter.

## 2. Cost components and allocation
| Component | Examples | Allocation method |
|---|---|---|
| GPU compute | accelerator node-hours, reserved/spot usage | measured active GPU-seconds plus agreed idle allocation |
| CPU/RAM | tokenizer, gateway, embedding, vector search | workload resource share |
| Storage | model weights, adapters, checkpoints | bytes and retention |
| Network | cross-zone traffic, egress, downloads | metered transfer |
| Observability | logs, traces, metrics cardinality | ingest and retention |
| Operations | support, engineering, redundancy | explicit shared-overhead rule |

Define chargeback/showback policy for idle capacity, N+1 reserve and shared services. Keep allocations auditable; do not pretend that GPU duty cycle maps perfectly to invoice cost.

## 3. Core formulas
```text
gpu_hour_cost = gpu_count * billed_hours * price_per_gpu_hour
allocated_cost = direct_cost + assigned_shared_overhead
cost_per_1m_tokens = allocated_cost / completed_tokens * 1_000_000
cost_per_good_request = allocated_cost / SLO_compliant_requests
utilization = active_gpu_seconds / provisioned_gpu_seconds
budget_burn_ratio = projected_period_spend / approved_budget
```
For self-hosted GPUs, provisioned node-hours are billed even during idle periods. A realistic forecast includes commitments, spot interruption risk, scaling lag, data transfer, region, licensing and storage. GPU pricing in examples is synthetic and not a market quote.

## 4. Rightsizing methodology
1. Establish baseline: p50/p95/p99 latency, TTFT, TPOT, throughput, goodput, queue time, error rate and GPU/KV memory.
2. Separate prefill-heavy and decode-heavy traffic; segment by prompt/output token distributions.
3. Identify idle nodes, oversharded replicas, low batch occupancy and high memory fragmentation.
4. Propose model placement, quantization, tensor/pipeline parallelism, autoscaling or replica changes.
5. Validate quality and safety before adopting quantization or model changes.
6. Stress-test realistic concurrency, N-1 failover and cold-start/recovery behavior.
7. Compare cost per good request and SLO compliance, not just average GPU utilization.
8. Canary changes with guardrails and automatic rollback triggers.
9. Update cost allocation and capacity forecast.
10. Review after traffic shifts, new model revisions or price changes.

## 5. Budget alerts and anomaly detection
Use daily spend, month-to-date actual, forecast end-of-period, per-tenant burn, idle GPU-hours, cost per good request and unallocated spend. Alert on forecast crossing approved thresholds and sudden cost/usage anomalies. Distinguish expected traffic spikes from accidental retry loops, overprovisioning and malicious denial-of-wallet activity (Chapter 082).

Example **illustrative** policy:
```yaml
budget:
  currency: USD
  monthly_limit: 10000
  forecast_alert_levels: [0.70, 0.85, 0.95]
  critical_burn_ratio: 1.00
rightsizing_guardrails:
  p95_latency_regression_pct_max: 10
  goodput_regression_pct_max: 5
  availability_slo_required: true
  n_minus_one_validation_required: true
```
Production limits require approval and workload baselines. Alerts should route to SRE and FinOps owners and include a documented escalation path.

## 6. SRE/DBRE operational scenarios
| Scenario | Investigate | Action |
|---|---|---|
| GPU cost rises, requests flat | longer prompts, output tokens, retries | tune token admission and batch efficiency |
| Low GPU utilization, high latency | serialization, CPU tokenizer, network, queue | profile bottleneck before removing capacity |
| High utilization, OOM | KV cache, model size, fragmentation | enforce memory-aware scheduling |
| Autoscaling oscillates | queue signals, cold start, cooldown | adjust policies and test |
| Tenant cost allocation drifts | usage labels, shared overhead | reconcile ledger with billing |
| Spot capacity interrupts | disruption rate, fallback | preserve SLO with reserved capacity |
| Cheaper quantization harms quality | eval scores, safety regressions | rollback model revision |

## 7. Production runbooks

### RB-083-A — Budget forecast breach
Confirm billing freshness and ledger integrity, attribute spend by workload, model and tenant, identify idle/duplicate GPU pools, activate approved budget controls, and document impact and owner.

### RB-083-B — Rightsizing rollback
Compare canary p95/p99 latency, TTFT, goodput, OOM and error budget. Restore last approved replica/model configuration on threshold breach and retain sanitized benchmark evidence.

### RB-083-C — Cost anomaly with retry storm
Correlate token volume, 429/503, retries, queue age and cost. Apply Chapter 082 gateway admission controls, stop recursive agent loops and reconcile actual spend.

### RB-083-D — Billing mismatch
Check timezone, currency, amortization, reservation discounts, GPU SKU, cloud invoice lag and shared allocation rules. Publish reconciled correction without rewriting raw billing history.

## 8. Acceptance criteria
| Area | Points |
|---|---:|
| Complete cost inventory and chargeback/showback | 20 |
| Trusted metering, reconciliation and unit economics | 20 |
| Forecasting, budget alerts and anomaly response | 20 |
| Rightsizing benchmark with SLO/N-1 guardrails | 20 |
| Automated tests, runbooks and audit | 20 |
| **Total** | **100** |

Any rightsizing change that violates critical reliability, isolation or quality guardrails blocks production rollout. Educational target >=90/100. Live benchmark, billing validation and FinOps review are pending.

## 9. Companion lab
[Chapter 083 — Executable GPU FinOps, Budget Alerts and Rightsizing Lab](../labs/083-gpu-finops-budget-alerts-rightsizing-check.md). This deterministic Python lab uses synthetic hourly GPU pricing, workload samples and budget policy; it does not query live cloud bills or benchmark GPUs.

## References
- [OpenCost](https://www.opencost.io/)
- [FinOps Foundation](https://www.finops.org/framework/)
- [NVIDIA DCGM Exporter](https://github.com/NVIDIA/dcgm-exporter)
- [Kubernetes Resource Management](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [OpenTelemetry Metrics](https://opentelemetry.io/docs/concepts/signals/metrics/)

**Next:** Chapter 084 — Policy-as-Code, Change Management and Audit.
