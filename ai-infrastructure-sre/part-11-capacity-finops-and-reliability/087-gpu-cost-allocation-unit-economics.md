# Chapter 087 — GPU Cost Allocation and Unit Economics

**Part 11 — Capacity, FinOps and Reliability | AI Infrastructure Engineering & SRE**

**Status:** Documentation published. Billing reconciliation, lab execution, live staging measurement, independent FinOps/security review and operational acceptance remain **pending**.

## 1. Mission

Build a defensible GPU FinOps model for Kubernetes-based AI infrastructure that allocates infrastructure costs to accountable teams and measures the cost of **successful training and SLO-compliant inference**. Separate raw GPU-hours, allocated time, productive compute, billed commitments and avoidable spend. Produce auditable showback and chargeback reports without creating incentives to weaken reliability, privacy or model quality.

**Prerequisites:** Chapters 083–086, Kubernetes labels and quotas, GPU telemetry/DCGM, cloud billing exports, distributed training, inference serving, observability, cost governance and basic financial controls.

### Outcomes

- Reconcile provider invoices with node, GPU, namespace, team and workload consumption.
- Define ownership, allocation keys, shared-cost policies and unallocated cost controls.
- Calculate GPU-hour, successful training step, successful token and SLO-compliant request economics.
- Attribute idle capacity, checkpoint/retry overhead, reservation amortization, storage, network and model serving overhead.
- Compare showback, chargeback and unit-cost optimization without inventing actual cloud rates.
- Publish a versioned cost ledger, dispute process, runbooks and independent acceptance evidence.

## 2. Cost accounting boundaries

\`\`\`text
Cloud / colo invoice + hardware depreciation + licenses
            ↓
Account / project / cluster / node / GPU SKU
            ↓
GPU allocation + CPU/RAM + storage + network + shared services
            ↓
Tenant / namespace / model / training job / inference service
            ↓
Unit economics:
  training: $ / successful step, $ / target-quality run
  inference: $ / 1M successful tokens, $ / SLO-compliant request
            ↓
Showback → approved chargeback → optimization / capacity planning
\`\`\`

**Showback** reports cost without transferring funds. **Chargeback** allocates or bills cost to consumers according to an approved policy. Neither should be based on unreviewed telemetry alone.

## 3. Cost taxonomy and ownership

| Cost pool | Typical source | Allocation method |
|---|---|---|
| GPU compute | provider bill, on-prem depreciation | allocated GPU-hours, SKU, rate |
| CPU/RAM on GPU nodes | node price decomposition | proportional or explicit policy |
| GPU commitments | savings plan, reservation, contract | commitment coverage and policy |
| Idle reserved capacity | billed but unallocated nodes | platform/shared or accountable reservation owner |
| Storage | checkpoint, model registry, data | bytes-month, requests, lifecycle |
| Networking | cross-AZ, egress, RDMA fabric | metered usage or shared policy |
| Observability/security | telemetry, logging, scanners | weighted shared service |
| Failure/retries | aborted training, failed inference | owning workload plus platform-caused attribution |
| Licensing/support | vendor contracts | service or enterprise shared allocation |
| Energy/colo | on-prem power and rack | measured/estimated approved allocation |

**Do not double-count:** if GPU node hourly cost includes CPU and RAM, do not add their full cost again. On-prem depreciation and cloud instance charges are different accounting bases; document policy.

## 4. Data contract

Required join keys:
- UTC time interval and billing period.
- cloud account/subscription/project, cluster ID, node ID and GPU SKU;
- namespace, workload UID, tenant, cost center and model/service owner;
- GPU requested/allocated count and duration;
- job outcome, attempts, steps, successful tokens, request count and SLO compliance;
- reservation/commitment ID, price basis and discount attribution;
- storage/network cost pool, currency and exchange-rate policy;
- confidence and completeness flags.

**Governance:** use stable workload UIDs instead of mutable pod names, map unlabeled/orphaned costs to a visible suspense pool, restrict cost center edits and redact customer/model-sensitive identifiers. Do not expose raw cloud billing exports publicly.

## 5. Key formulas

### 5.1 Allocated GPU-hours

\[
H_{\mathrm{GPU}}=\sum_i G_iT_i
\]

For a workload allocating 8 GPUs for 3 hours, \`H_GPU=24\`. This does not indicate how much useful work was completed.

### 5.2 Weighted GPU cost

\[
C_{\mathrm{GPU}}=\sum_i H_i\,R_i
\]

where \`R_i\` is the approved effective GPU-hour rate for the same SKU, period and accounting basis. **An instance-hour price is not automatically a GPU-hour price**; allocation of bundled CPU/RAM and idle GPUs requires an explicit policy.

### 5.3 Training cost per useful step

\[
C_{\mathrm{step}}=\frac{C_{\mathrm{total\,training\,attempts}}}{N_{\mathrm{successful\,optimizer\,steps}}}
\]

Include failed attempts, checkpoint/storage and networking where attributable. Also report **cost to reach target model quality** to avoid rewarding faster but poorer convergence.

### 5.4 Inference cost per million successful tokens

\[
C_{\mathrm{1M\,tokens}}=
\frac{C_{\mathrm{inference\,service}}}{N_{\mathrm{successful\,tokens}}}\times10^6
\]

Track prompt and output tokens separately where their compute profiles differ. Count failed/retried requests in cost numerator, but not successful-token denominator. State whether tokens include cache hits, input tokens or generated output.

### 5.5 Cost per SLO-compliant request

\[
C_{\mathrm{SLO\,request}}=
\frac{C_{\mathrm{serving\,total}}}{N_{\mathrm{requests\,meeting\,defined\,SLO}}}
\]

Define request eligibility, latency window, error exclusions and aggregation consistently with the actual SLO.

### 5.6 Allocation reconciliation

\[
C_{\mathrm{invoice}}=
C_{\mathrm{allocated}}+C_{\mathrm{shared}}+C_{\mathrm{unallocated}}+C_{\mathrm{reconciliation\,variance}}
\]

Document discounts, credits, taxes, amortization, exchange rates and rounding. A zero reconciliation variance is the target **after** source adjustments, not a reason to hide unknown costs.

## 6. Synthetic worked example

A training run has:
- 8 GPUs × 4 hours × 1 attempt = 32 GPU-hours.
- 8 GPUs × 2 hours × 1 failed retry = 16 GPU-hours.
- **Total: 48 GPU-hours.**
- Synthetic approved rate: **$2/GPU-hour** (illustrative only).
- GPU compute cost: **$96**.
- Other attributable storage/network costs: **$12**.
- Successful optimizer steps: **12,000**.

\[
C_{\mathrm{step}}=(96+12)/12000=\$0.009
\]

If the failed attempt were excluded, reported cost per step would be understated. This rate is a fictional exercise input, not a current cloud price.

### Inference illustration

A serving pool incurs a **synthetic $600/day** and produces **120 million successful output tokens**.

\[
C_{\mathrm{1M\,output\,tokens}}=600/120=\$5
\]

The example does not assert that $5 is a competitive or achievable rate; actual unit cost depends on GPU SKU, model, request mix, concurrency, caching, SLO and commercial terms.

## 7. Shared-cost allocation models

| Model | Appropriate use | Risk |
|---|---|---|
| Direct attribution | dedicated GPU nodes/jobs | missing shared overhead |
| Allocated GPU-hours | pooled accelerator clusters | charges idle reservations incorrectly if policy unclear |
| Reservation owner pays | capacity held for one team | can discourage prudent headroom |
| Proportional usage | shared storage/network/telemetry | imperfect causal attribution |
| Fixed platform tax | stable shared services | masks actual growth drivers |
| Hybrid | dedicated + pooled + shared | needs strong reconciliation |

Select one documented method per cost pool and period. Track **unallocated percentage** and material variances, with a clear owner for disputes and corrections.

## 8. Commitments and realized savings

A discount commitment and a physical capacity reservation may be separate. Track:
- gross list-rate cost versus net invoice cost;
- commitment amortization and effective coverage;
- unused reservation/commitment exposure;
- freed GPUs versus avoided purchases versus actual invoice reduction;
- spot interruption cost, checkpoint loss and retries.

**Optimization decision:** a 25% reduction in allocated GPU-hours can produce **zero immediate invoice savings** when the contract is fixed; however, freed capacity can still improve queue wait, resilience or defer procurement. Report each outcome separately.

## 9. Kubernetes chargeback implementation

Recommended architecture:

\`\`\`text
Billing export ───────────────┐
Kubernetes workload metadata ─┤
GPU allocation/telemetry ─────┤→ versioned cost ledger
Scheduler and queue events ───┤       ↓
Training/inference metrics ───┤  reconciliation + QA
Storage/network billing ──────┘       ↓
                               tenant showback dashboard
                                      ↓
                              approved chargeback export
\`\`\`

Use labels such as \`cost-center\`, \`owner-team\`, \`service\` and \`environment\` with an admission policy; do not trust arbitrary user-provided billing labels for money transfers without validation. Keep a historical owner map when jobs or namespaces are deleted.

## 10. FinOps dashboards and alerts

Show:
- billed GPU compute, allocated GPU-hours and idle reserved GPU-hours by SKU/tenant;
- cost per successful training step, target-quality run and successful inference token;
- SLO-compliant request unit cost and p99/error-rate trend;
- committed spend, utilization, expiry and reservation coverage;
- failed attempt/retry cost and checkpoint overhead;
- shared/unallocated cost percentage and invoice reconciliation variance;
- budget variance, forecast variance and anomaly explanations.

Alert on missing cost labels, unexpected idle reservation growth, retry-driven spend spikes, cost per successful unit increases and reconciliation drift. Avoid alerting solely on GPU utilization without context.

## 11. Six production runbooks

**A — Unattributed GPU spend:** locate cloud bill, node and workload owner mapping; isolate missing metadata; correct via approved ledger adjustment, not retroactive guesswork.

**B — Cost anomaly:** verify actual invoice/rate changes, GPU-hour growth, retries, model rollout, traffic and storage/egress; protect SLO before scaling down.

**C — Commitment underutilization:** identify expiring/idle commitments, upcoming demand and cancellation constraints; coordinate FinOps and procurement.

**D — Training cost regression:** compare GPU-hour attempts, steps/sec, checkpoint failures, data pipeline and NCCL; optimize useful work, not only allocated time.

**E — Inference unit-cost regression:** inspect traffic mix, token lengths, batching, cache hit ratio, p99 and error rate; canary any change.

**F — Chargeback dispute:** freeze ledger version, reproduce allocation, verify owner and cost center, approve correction, preserve audit trail.

## 12. Sixteen hands-on exercises

1. Inventory GPU cost pools and contractual rate basis.
2. Build stable Kubernetes workload-to-owner mapping.
3. Compute allocated GPU-hours and idle exposure.
4. Reconcile a synthetic cloud invoice to a ledger.
5. Allocate shared storage/network/platform costs.
6. Calculate training cost per successful optimizer step.
7. Calculate cost to target model quality.
8. Calculate inference cost per million successful output tokens.
9. Calculate cost per SLO-compliant request.
10. Attribute failed attempts and checkpoint overhead.
11. Model reservation/commitment amortization.
12. Distinguish nominal, avoidable and realized savings.
13. Build showback by team and GPU SKU.
14. Simulate chargeback dispute and ledger correction.
15. Create cost anomaly and reconciliation alerts.
16. Produce audit evidence, peer review and acceptance score.

## 13. Twelve troubleshooting scenarios

| Symptom | Investigate |
|---|---|
| GPU spend rises, GPU-hours flat | rate/discount/commitment changes |
| GPU-hours fall, invoice flat | fixed commitments |
| High utilization, high token cost | model mix, throughput, retries |
| Training steps cost rises | failed attempts or NCCL |
| Cost labels missing | admission and historical mapping |
| Namespace deleted | workload UID and owner snapshot |
| Shared cost double-counted | cost pool boundaries |
| Spot looks cheap, job retries rise | failed GPU-hours |
| Model quality falls after optimization | cost-to-quality gate |
| Serving p99 worsens after rightsizing | rollback and SLO gate |
| Invoice differs from ledger | credits/tax/FX/timing |
| Chargeback disputed | reproducible ledger version |

## 14. Acceptance gate (100 points)

| Gate | Points |
|---|---:|
| Billing sources, ownership and data quality | 20 |
| Invoice-to-ledger reconciliation | 20 |
| Training and inference unit economics | 20 |
| Shared cost, commitments and savings integrity | 15 |
| SLO/quality-aware optimization and controls | 15 |
| Runbooks, audit trail and independent review | 10 |
| **Total** | **100** |

**Pass:** ≥85/100 with no critical double-counting, privacy breach, false savings claim, unauthorized chargeback, unsupported financial rate or SLO/quality regression. Real invoice and benchmark acceptance remain pending until independently verified.

## 15. References

- [FinOps Foundation Framework](https://www.finops.org/framework/)
- [FinOps FOCUS Specification](https://focus.finops.org/)
- [Kubernetes Recommended Labels](https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/)
- [NVIDIA DCGM](https://docs.nvidia.com/datacenter/dcgm/latest/)
- [Kubernetes Resource Management](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [OpenCost](https://www.opencost.io/)

**Next:** Chapter 088 — Scaling and Queue Admission Control.
