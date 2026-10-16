# Chapter 12.5 — FinOps, GPU Cost Governance and Rightsizing

**Part 12 — Production Operations | TrueFoundry SRE canonical tutorial**

## Objectives and operating boundaries

Build a repeatable GPU FinOps operating model for TrueFoundry-hosted vLLM inference. Attribute costs to model, environment and team; measure useful inference throughput per dollar; rightsize GPU nodes, model replicas and warm pools without violating customer SLOs; establish budgets, forecasts and change controls.

**[SAFE-READ]** inspect billing exports, Kubernetes inventory and metrics. **[LAB-MUTATE]** synthetic staging benchmarks with approved GPU spending limits. **[CHANGE-CONTROL]** production replicas, node pools, reservations, model configuration, autoscaling and cloud commitments. Never disable production redundancy or expose billing account identifiers, secrets, customer prompts or PHI in shared evidence.

## 1. The inference cost stack

```text
Cloud invoice
  ├─ GPU compute instance-hours (allocated nodes, idle included)
  ├─ CPU-only control/worker nodes
  ├─ block/object storage and model artifact transfers
  ├─ load balancers, NAT, network egress, cross-zone traffic
  ├─ monitoring, logs, metrics and trace ingestion/retention
  ├─ managed platform/license charges (where applicable)
  └─ support, discounts, reservations and taxes (invoice-dependent)
                   │
   Kubernetes node/pod allocation and usage
                   │
   TrueFoundry workload/model/tenant attribution
                   │
   unit economics + SLO + governance decisions
```

Do not assume a TrueFoundry built-in billing API or a universal GPU chargeback field. Verify which cloud billing exports, tags, cluster labels and inference metrics are actually available. Provider invoices are authoritative for billed cost; Kubernetes allocation is an attribution estimate.

## 2. Allocation versus utilization

A GPU node costs money while provisioned even if GPU utilization is zero. Separate **allocated GPU-hours** (GPU capacity reserved to a workload), **node billed GPU-hours** (provider instance billing) and **active GPU-hours** (derived from measured utilization; not necessarily billable). GPU sharing/MIG changes attribution but not necessarily the total instance bill.

Cost-allocation dimensions: cloud account, region, cluster, namespace, team, environment, model alias/revision, deployment, GPU SKU and cost center. Maintain label governance and an unallocated-cost bucket; never force unknown costs into an arbitrary tenant.

## 3. Unit economics and efficiency

```text
gpu_node_cost = sum(instance_hourly_rate × billed_hours)
allocated_gpu_cost = node_cost × approved_allocation_fraction
cost_per_1m_output_tokens = attributed_cost / output_tokens × 1,000,000
cost_per_successful_request = attributed_cost / successful_eligible_requests
gpu_cost_per_slo_compliant_token = attributed_gpu_cost / compliant_output_tokens
```

All rates and prices must be obtained from actual billing/contract data and the correct region, purchase option, currency and billing period. Define whether metrics include prompt tokens, completion tokens, cached tokens, retries, failed requests and idle time. A lower cost per token is not a win if TTFT, streaming completion or quality degrades.

**Illustrative only:** 4 GPU nodes at $3 per node-hour for 24 hours cost $288 before storage/network/discounts. With 36 million successful output tokens, that is $8 per million output tokens in GPU node cost. This is a synthetic arithmetic example, **not a current cloud price quote**.

## 4. Cost telemetry and chargeback

Join billing data and Kubernetes allocation by stable cloud resource/node identifiers and time interval. Attribute shared infrastructure with a documented policy (GPU requests, Pod GPU allocation, active time, reserved capacity or a hybrid). Keep actual billed amount, allocated amount, unallocated amount and allocation methodology visible.

For per-model unit economics, instrument completed requests, output token counters, token mix, latency, retries and SLO success. Do not use raw request IDs, tenant PII or prompts as Prometheus labels. Verify runtime metric names rather than assuming vLLM or TrueFoundry exports a specific series.

## 5. GPU rightsizing methodology

1. Establish a **representative** traffic baseline: steady, peak, burst, context length, input/output tokens and concurrent streams.
2. Benchmark the current model on current GPU type and engine version under TTFT/TPOT, error and completion SLOs.
3. Compare candidate GPU SKUs, MIG profiles (if supported), quantization, tensor parallelism, context limit and concurrency controls in staging.
4. Account for memory fit, KV cache, GPU utilization, CPU/network bottlenecks, cold start and node-provision latency.
5. Add failure-domain and burst headroom, warm replicas, minimum availability and PDB requirements.
6. Calculate **fully loaded** unit cost and savings after discounts, storage, transfer, monitoring and reserved capacity.
7. Canary the approved change; roll back on SLO, quality, reliability or cost regression.

Never rightsize using GPU utilization alone. A low-utilization GPU may be intentionally warm for strict TTFT or disaster recovery.

## 6. Scheduling and scaling controls

Review namespace ResourceQuota, GPU requests, HPA/KEDA min/max, Cluster Autoscaler/Karpenter node-pool limits, consolidation, cooldown, GPU warm pool and cloud quota. Protect priority services from noisy tenants. Scale-to-zero is only appropriate where measured cold-start latency and traffic policy permit it. Spot/preemptible GPU nodes require workload-specific interruption handling and sufficient on-demand fallback.

## 7. Forecasting and budget guardrails

Forecast per-model demand using traffic × token distribution × validated throughput per replica, with growth and seasonality scenarios. Report base/peak/stress monthly cost and committed spend separately. Use cloud budget alerts, team cost-center ownership, anomaly detection and monthly variance review. For large GPU purchases or reservations, verify utilization commitments and cancellation terms.

## 8. FinOps dashboards

Minimum panels: billed GPU cost, allocated/unallocated GPU cost, GPU-hours, idle warm capacity, per-model output tokens, successful requests, cost per million tokens, cost per successful request, TTFT/TPOT, queue depth, GPU memory/utilization, GPU SKU mix, node-pool scaling, budget variance and forecast. Include currency, time range, pricing basis, data freshness and SLO context. Missing billing export is **unknown cost**, not zero.

## 9. Twelve production runbooks

1. **Unexpected GPU spend spike:** reconcile invoice, node count, autoscaler, SKU and recent changes.
2. **Idle GPU node persists:** inspect pending Pods, daemon overhead, consolidation and warm-pool policy.
3. **Unallocated cloud costs:** fix resource IDs/tags and allocation policy before chargeback.
4. **Cost per token increases:** inspect token mix, retries, queue, throughput and discounts.
5. **Scale-out costs exceed budget:** verify max replicas/node limits and incident exception policy.
6. **Rightsizing causes TTFT regression:** compare prefill, warm capacity, GPU memory and load profile.
7. **GPU OOM after quantization change:** inspect model compatibility, KV cache and runtime overhead.
8. **Spot interruption impacts service:** inspect drain, fallback capacity and workload priorities.
9. **Reservation underutilized:** reconcile commitment, SKU/region eligibility and forecast.
10. **Telemetry/billing gap:** check export freshness, joins, retention and missing intervals.
11. **Cross-zone/network transfer surge:** inspect model artifact pulls, traffic path and failover.
12. **FinOps change violates HA:** restore reserved N-1 capacity and update policy.

## 10. Acceptance and governance

Require finance-approved cost basis, reproducible benchmarks, allocation policy, SLO-preserving rightsizing evidence, N-1 reserve, forecast scenarios, budget alerts, change record, rollback and sign-off. Do not claim savings before invoice validation.

Complete the [companion lab](../labs/05-finops-gpu-cost-governance-rightsizing-check.md): **32 failure scenarios, 12 acceptance gates and five cost/reliability game days**. Formal review and live execution pending.

**Next:** Chapter 12.6 — Operational Runbooks, Maintenance and Incident Command.
