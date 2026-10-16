# Chapter 12.5 — Hands-on Lab: FinOps, GPU Cost Governance and Rightsizing

**Part 12 | Approved staging | Live execution pending**

## Mission and safety

Build a GPU cost-allocation worksheet, measure inference unit economics, test rightsizing candidates under SLOs, review budget guardrails and rehearse cost incidents. **[SAFE-READ]** inspect billing and cluster state. **[LAB-MUTATE]** bounded synthetic staging. **[CHANGE-CONTROL]** GPU SKU, node pool, replica, scaling and reservation changes. Never incur unapproved GPU spend.

## Lab 0 — Inventory

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get resourcequota -A
kubectl get hpa -A
kubectl get events -A --sort-by=.lastTimestamp | tail -40
```

Record approved cloud billing source, period/currency, region, SKU, actual rates/discounts, model revision, GPU count, TrueFoundry/vLLM version, SLO, team owner and spend cap. Sanitize account IDs and confidential contract rates in published evidence.

## Lab 1 — Attribution and unit-cost worksheet

Create a dataset with billing node-hours, workload allocation windows, model output tokens, successful eligible requests, retry count and unallocated cost. Reconcile sum of attributed and unallocated costs to the invoice. Calculate cost per million output tokens and per successful request, explicitly including or excluding shared costs.

## Lab 2 — Synthetic rightsizing comparison

Benchmark baseline and approved candidate GPU/model/runtime combinations with representative input/output token distributions, p95 TTFT/TPOT, streaming completion, error rate, queue, GPU memory, cold start and steady throughput. Compare fully loaded costs and HA headroom; reject candidates that fail SLO or quality.

## Lab 3 — Forecast and budgets

Forecast monthly GPU-hours for steady, peak and stress traffic. Apply verified rate cards, reservations and network/storage estimates. Define anomaly alerts, team budget, owner, approval and emergency override.

## Lab 4 — Five game days

1. **GPU spend spike:** reconcile unexpected node count and billing.
2. **Idle warm pool:** distinguish waste from required latency/HA reserve.
3. **Rightsizing regression:** identify TTFT or streaming-completion failure and roll back.
4. **Spot/preemptible loss:** tabletop fallback and interruption recovery.
5. **Missing billing data:** mark cost unknown, restore export and reconcile.

Each game day captures scope, UTC timestamps, actual versus tabletop, impact, mitigation, rollback, reviewer and evidence.

## Thirty-two failure scenarios

| # | Scenario | Investigation |
|---|---|---|
| 01 | GPU node cost spikes | Invoice |
| 02 | Idle GPU nodes persist | Warm pool |
| 03 | Node pool exceeds max | Autoscaler |
| 04 | Scale-out exceeds budget | Guardrail |
| 05 | GPU cloud quota exhausted | Provider |
| 06 | Namespace cost center missing | Tags |
| 07 | Unallocated costs grow | Attribution |
| 08 | Double-counted GPU-hours | Join |
| 09 | Wrong billing currency | Finance |
| 10 | Wrong region/SKU rate | Pricing |
| 11 | Discount omitted | Billing |
| 12 | Spot price volatility | Purchase option |
| 13 | Reservation underutilized | Commitment |
| 14 | Storage/egress omitted | Full cost |
| 15 | Telemetry ingestion spike | Monitoring |
| 16 | Model artifact egress surge | Networking |
| 17 | Output token counter reset | Metrics |
| 18 | Retries inflate token cost | Reliability |
| 19 | Prompt/output tokens mixed | Definition |
| 20 | TTFT regression after rightsizing | SLO |
| 21 | TPOT regression after rightsizing | Decode |
| 22 | GPU OOM after model change | Memory |
| 23 | KV cache insufficient | Runtime |
| 24 | GPU low utilization but high queue | Bottleneck |
| 25 | Warm replica removed | Cold start |
| 26 | N-1 HA capacity lost | Reliability |
| 27 | Scale-to-zero breaks latency SLO | Availability |
| 28 | GPU SKU incompatible with CUDA | Runtime |
| 29 | MIG accounting incorrect | Allocation |
| 30 | Spot interruption loses service | Recovery |
| 31 | Cost dashboard stale | Freshness |
| 32 | Savings claim not invoice-verified | Governance |

For every scenario record expected/actual, safe UTC evidence, impact, mitigation, rollback and reviewer. Unsafe or expensive scenarios are tabletop.

## Twelve acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G1 | Approved scope, spend cap and ownership | Pending |
| G2 | Cloud billing source and rate basis verified | Pending |
| G3 | GPU-hours and allocation reconciled | Pending |
| G4 | Unit economics definitions documented | Pending |
| G5 | Workload/token distribution representative | Pending |
| G6 | Rightsizing benchmarks SLO-constrained | Pending |
| G7 | GPU memory and runtime compatibility checked | Pending |
| G8 | N-1 reserve and cold-start policy protected | Pending |
| G9 | Budgets, forecasts and anomalies reviewed | Pending |
| G10 | Dashboard freshness and privacy checked | Pending |
| G11 | Rollback and cost recovery validated | Pending |
| G12 | SRE/FinOps/service-owner sign-off | Pending |

Pass requires evidence and sign-off. Synthetic projections are not realized savings until invoice reconciliation.

**Next:** Chapter 12.6 — Operational Runbooks, Maintenance and Incident Command.
