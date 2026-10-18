# Chapter 075 — Performance and Capacity Acceptance Lab

**Part 09 — Performance Engineering and Capacity · Chapter 075 of 100 · Final Part 09 Capstone**

**Status:** Tutorial and executable companion lab published; production validation, live lab execution and independent technical/security review pending.

## Mission
Act as the SRE release owner for a multi-replica LLM inference service. Decide whether a candidate deployment can sustain forecast traffic while preserving latency, successful-token goodput, KV memory headroom, N-1 resilience, rollout safety and unit economics. **No deployment is approved solely because a synthetic lab passes.**

## 1. End-to-end acceptance architecture
```text
representative synthetic cohorts + immutable model configuration
                            |
                  open-loop arrival driver
                            |
       gateway quotas -> bounded queue -> scheduler
                            |                |
                        errors           GPU replicas
                            |                |
          TTFT / TPOT / E2E / output tokens / KV occupancy
                            |
              SLO-compliant goodput + cost
                            |
      capacity forecast + N-1 + rollout surge + warmup
                            |
          evidence bundle -> release gate -> sign-off
```

## 2. Prerequisite chapter integration
| Chapter | Evidence required |
|---|---|
| 066–067 | cohort latency, token throughput and baseline methodology |
| 068 | concurrency, queueing, backpressure, deadlines and admission |
| 069 | KV-cache bytes, active token budget and OOM margin |
| 070 | quantization quality and safety gate |
| 071 | GPU cost per million good output tokens |
| 072 | cold start, warm-pool cost and readiness |
| 073 | load, stress, spike, soak and failure test results |
| 074 | forecast demand, N-1 capacity and rollout headroom |

## 3. Acceptance contract
Document service model digest, tokenizer, quantization, GPU SKU, region, worker count, context and output limits, workload cohort mix, safe RPS per replica, peak forecast, billing assumptions and test window.

**Example illustrative thresholds, not production defaults:**
- p95 TTFT <= 2.0 seconds and p95 TPOT <= 0.15 seconds for the interactive cohort.
- At least 99% of offered requests fully complete inside the agreed request SLO.
- No OOM, unbounded queue growth, unsafe retries or critical quality/safety regression.
- N-1 surviving capacity >= forecast peak RPS.
- Rollout unavailable capacity >= forecast peak RPS and enough GPU slots exist for configured surge.
- Cost per million **good output tokens** <= an approved budget.
- Warmup and route propagation finish before the documented scale-out lead-time deadline.

For each threshold, record numerator, denominator, sample count, cohort, confidence and owner. Tail latency computed only over completed requests must be accompanied by timeout/rejection counts.

## 4. Test matrix
| Phase | Action | Pass/fail evidence |
|---|---|---|
| Preflight | verify permissions, quotas, synthetic data, model digest | signed run plan |
| Baseline | warm workers and measure stable traffic | reference latency/goodput |
| Load | hold forecast demand with open-loop arrivals | SLO and error gates |
| Ramp/stress | increase offered rate to capacity knee | safe throughput ceiling |
| Spike | bounded burst, no retry amplification | queue and rejection recovery |
| Soak | sustained staging traffic | memory/queue stability |
| KV pressure | long-context cohort near budget | no OOM/preemption surprises |
| Quantization | matched baseline/candidate evaluation | quality and safety gate |
| N-1 | remove one approved staging replica | surviving goodput and recovery |
| Rollout | canary with maxSurge/maxUnavailable | readiness and surge quota |
| Cold start | empty artifact cache vs warm pool | p95 time-to-ready |
| FinOps | bill all GPU-hours including standby | cost per good token |

## 5. Executable staging workflow
1. Open a change record identifying owner, approver, blast radius, test namespace, maximum GPU count, spend cap, timeout and abort authority.
2. Pin model artifacts and record image digest, runtime versions, tokenizer and GPU driver versions.
3. Use synthetic de-identified prompts and fixed cohort distributions. Never send real customer content to an unapproved benchmark.
4. Capture **offered arrivals** independently from successful completions; record failed, partial, timed-out and rejected streams.
5. Warm the service and execute baseline, ramp, stress, spike and approved-duration soak.
6. Measure per-cohort p95 TTFT/TPOT/E2E, goodput, KV occupancy, preemptions, GPU memory, GPU-hours and cost per million good tokens.
7. Inject one controlled replica loss; check N-1 and readiness, then restore before rollout tests.
8. Verify maxSurge quota and rollout maxUnavailable capacity. Abort if pending GPU pods, OOM or SLO burn exceeds agreed limits.
9. Run quality/safety gates against the pinned candidate and held-out evaluation corpus.
10. Archive sanitized results, reproducible commands, dashboard snapshots, run ID, revision, timestamp and sign-off decisions.

## 6. Stop conditions
**Immediate abort:** unapproved endpoint/traffic, suspected privacy leak, unsafe artifact, critical safety regression, GPU OOM, persistent runaway queue, exhausted error budget, breach of approved spend/time/GPU cap or unexpected impact outside staging. A test that triggers an abort is a **failed acceptance run**, even if some throughput numbers look good.

## 7. Failure investigation and runbooks

### RB-075-A — Acceptance test abort
Stop load generator and fault injection, record timestamps, identify breached stop condition, restore staging state, verify traffic and escalate per on-call procedure. Preserve sanitized evidence; do not rerun until root cause and authorization are documented.

### RB-075-B — Latency or goodput failure
Compare offered/completed/rejected counts, TTFT vs TPOT, queue age, active sequences, KV occupancy and GPU saturation. Isolate workload cohort or scheduler regression; roll back through approved deployment controls.

### RB-075-C — N-1 or rollout failure
Confirm actual ready replicas rather than desired count; inspect failure-domain placement, maxSurge/maxUnavailable, pending GPU quotas, cold-start and routing delay. Restore old capacity before continuing.

### RB-075-D — Quantization or quality regression
Freeze candidate, compare paired held-out results, critical safety cases, tokenizer and kernel versions. Do not waive safety gates to achieve performance or cost targets.

### RB-075-E — Unit economics failure
Audit billed GPU-hours, idle standby, retries, output-token accounting and SLO-compliant denominator. Report GPU-only versus fully loaded costs separately.

## 8. Evidence bundle
```text
acceptance/
  run-manifest.json
  config-digests.txt
  synthetic-cohort-definition.md
  arrival-schedule.csv
  request-outcomes.csv
  latency-goodput-summary.json
  gpu-kv-metrics.json
  forecast-n-minus-one.json
  quantization-quality-gates.json
  cost-assumptions.json
  fault-injection-timeline.md
  abort-log.md
  approval-and-rollback.md
```
Store evidence in approved access-controlled systems. Do not include raw prompts, completions, tokens, secrets or personal identifiers.

## 9. Final 100-point scorecard
| Domain | Points |
|---|---:|
| Workload reproducibility, SLO measurement and load-driver validity | 15 |
| Latency, queueing and successful-token goodput | 15 |
| KV memory, concurrency and quantization quality | 15 |
| Cold start, warm pools and readiness | 10 |
| Stress, soak, failure and recovery drills | 15 |
| Forecasting, N-1, failure domains and rollout headroom | 15 |
| GPU cost, privacy, operational runbooks and evidence | 15 |
| **Total** | **100** |

**Educational pass recommendation:** >=90/100 **and** all critical safety, quality, N-1 and SLO gates pass. A high score never overrides a critical failed gate. Record independent reviewer names/approval in the organization’s normal system; no review is implied by publication.

## 10. Companion lab
[Chapter 075 — Executable Performance and Capacity Acceptance Lab](../labs/075-performance-capacity-acceptance-lab-check.md). The lab computes a deterministic release-gate decision from synthetic evidence and includes automated regression tests. It does **not** measure GPU hardware or certify production.

## References
- [Google SRE Workbook — Handling Overload](https://sre.google/workbook/handling-overload/)
- [Kubernetes Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [OpenTelemetry documentation](https://opentelemetry.io/docs/)
- [vLLM benchmarking](https://docs.vllm.ai/en/latest/benchmarking/)

**Next:** Chapter 076 — LLM Threat Modeling and Trust Boundaries, beginning Part 10 — AI Security, Governance and FinOps.
