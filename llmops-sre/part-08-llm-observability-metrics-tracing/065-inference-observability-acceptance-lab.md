# Chapter 065 — Inference Observability Acceptance Lab

**Part 08 — LLM Observability, Metrics and Tracing · Final Capstone**  
**Publication status:** Tutorial and companion lab published. **Technical review, security review and live execution:** pending.

## Mission and acceptance objective
Build and validate an end-to-end observability contract for an LLM inference service: safe structured logs, metrics, traces, synthetic probes, SLO burn-rate alerts, dashboards, operational runbooks and a repeatable evidence bundle. This capstone integrates Chapters 056–064. The lab uses **synthetic local traffic**, not a production GPU or live model; production readiness requires separate deployment-specific acceptance.

### Learning outcomes
1. Identify the inference user journey and explicit eligible/bad event taxonomy.
2. Validate telemetry coverage across gateway, routing, queue and serving stages.
3. Verify no prompt, completion, token, credential or user identifier enters observability records.
4. Detect elevated error ratio, latency, saturation, missing telemetry and cardinality explosions.
5. Correlate safe trace context with request outcome without sensitive attributes.
6. Demonstrate on-call triage, rollback decision criteria and error-budget governance.
7. Produce a scored, auditable acceptance report with evidence and limitations.

## 1. Reference architecture
```text
  synthetic client / probe
           |
  gateway + auth boundary
           |
     model router ---- queue
           |             |
       model server / stream completion
           |
    bounded SLI counters + duration histogram
           |
   OTEL spans ---- structured safe logs
           |               |
     Collector/filter ----+---- Prometheus scrape
                                |
                    recording + SLO alerts
                                |
                     dashboards / on-call
                                |
                   acceptance evidence
```
The synthetic lab implements only the contract validation, calculation and simulation layers. A real rollout additionally requires an authenticated gateway, model server, exporter mapping, collector configuration, dashboards, alert routing and GPU telemetry.

## 2. Observability acceptance matrix
| Layer | Required evidence | Failure signal | Owner |
|---|---|---|---|
| Gateway | eligible/bad counts, status taxonomy | false 200 success or dropped requests | API SRE |
| Inference | TTFT, TPOT, completion, queue and saturation | user-visible latency or incomplete stream | Inference SRE |
| Logs | fixed allowlist, bounded fields, zero content | leaked prompt, token or secret | App + Security |
| Traces | gateway→router→worker span hierarchy, safe attributes | missing propagation, excessive sampling | Platform SRE |
| Metrics | bounded label sets, histogram bucket integrity | cardinality growth or misleading p95 | Observability |
| SLO | 30-day objective, budget math, burn-rate rules | paging delay or false alert | Service owner |
| Probes | fixed synthetic payload, timeout, status and schema | false green health | On-call |
| Operations | runbooks, escalation, evidence and rollback criteria | unactionable page | Incident commander |

### Required metric contract
`llm_sli_requests_total{service,environment}` and `llm_sli_bad_requests_total{service,environment}` must count at the same boundary. Document whether overload 429, client cancellation and partial streaming completion are bad events. `llm_request_duration_seconds_bucket{service,environment,le}` is an illustrative classic histogram; actual exporter names may differ. Use only approved low-cardinality dimensions.

## 3. Acceptance scenarios
| ID | Scenario | Pass condition |
|---|---|---|
| A01 | Healthy synthetic requests | no bad events; SLO budget preserved |
| A02 | Serving-side failure | bad-event count increments and burn rate rises |
| A03 | Streaming 200 but incomplete | completion SLI records bad event |
| A04 | Slow first token | TTFT objective breach detected independently of availability |
| A05 | Queue saturation | queue signal present; alert corroborated by user impact |
| A06 | Prompt and credential canary | sensitive marker absent from every exported event |
| A07 | Dynamic model/request labels | unknown values map to bounded `other` |
| A08 | Trace propagation | parent-child spans valid; no content attributes |
| A09 | Synthetic probe error/timeout | probe fails closed with bounded reason |
| A10 | Missing telemetry | no-data detected, not represented as success |
| A11 | High SLO burn | long and short windows both evaluated |
| A12 | Recovery | alerts clear appropriately and incident evidence recorded |

## 4. Executable capstone
See [Chapter 065 companion lab](../labs/065-inference-observability-acceptance-lab-check.md). It includes:
- a Python-only synthetic acceptance engine and reproducible pytest suite;
- tests for event classification, privacy, cardinality, trace relationships, probes and error-budget arithmetic;
- deterministic failure drills and an evidence checklist;
- a 100-point production-readiness rubric.

Run in a disposable environment and capture versions, commands, sanitized test output and limitations. **Published documentation is not evidence that tests were executed.**

## 5. SLO and budget governance
For target S and rolling eligible/bad counts N/B:
```text
allowance = N × (1 − S)
consumed = B / allowance
remaining = 1 − consumed
burn_rate = (B / N) / (1 − S)
```
These formulas require N > 0 and 0 < S < 1. If instrumentation is absent or denominator is zero, report **unknown/no traffic**, not healthy. Page on sustained multi-window burn, not merely a single instantaneous ratio. A 99.9% SLO has 0.1% error allowance; a 2% error ratio burns at 20× the allowed rate.

## 6. Incident simulations and troubleshooting
**SIM-1 — Prompt leakage:** add a synthetic secret field to a raw input. Confirm safe serializer drops it; if emitted, halt telemetry export in a controlled drill and invoke security incident process.

**SIM-2 — Counter drift:** increment bad events beyond eligible events. Acceptance must reject inconsistent counts and investigate event boundaries.

**SIM-3 — Silent metric outage:** simulate absent series. A dashboard must show no data and raise telemetry-health investigation; it must not infer zero failures.

**SIM-4 — Alert storm:** multiple services breach thresholds simultaneously. Verify ownership, grouping, inhibition, escalation and silence expiry in a non-paging environment.

**SIM-5 — Partial stream failure:** simulate successful HTTP status with incomplete generation. Ensure stream-completion SLI is independently bad.

**SIM-6 — Saturation and latency:** raise synthetic queue and TTFT. Distinguish cause indicators from user-facing SLI breaches.

## 7. Production runbooks

### RB-065-A — User-impacting inference failure
**[SAFE-READ]** verify request volume, eligible/bad taxonomy, 5xx/429, streaming completion, TTFT and SLO burn windows; correlate deployments and safe trace IDs. **[CHANGE-CONTROL]** invoke approved rollback, traffic shift or capacity remediation. Record owner, timeline, impact and recovery verification.

### RB-065-B — Observability blind spot
Check scrape freshness, exporter health, collector drops, rule evaluations and probe coverage. Escalate loss of visibility as an operational risk. Avoid asserting SLO compliance until data quality is restored.

### RB-065-C — Sensitive telemetry exposure
Contain at source, restrict downstream access, notify security/privacy, follow retention/evidence rules and rotate credentials if exposed. Validate remediation with synthetic canaries only; do not copy real content into incident tickets.

### RB-065-D — Error-budget exhaustion
Validate 30-day rolling counts and exclusions, calculate consumed budget, convene service owner and SRE, prioritize reliability changes, and document exceptions to any release restriction.

### RB-065-E — Capstone acceptance handoff
Deliver sanitized test reports, alert policy, SLI taxonomy, dashboard inventory, on-call escalation, ownership, deployment mapping, security review and unresolved risks. Do not sign off a production service from synthetic tests alone.

## 8. 100-point acceptance rubric
| Category | Points |
|---|---:|
| SLI event taxonomy and user-journey coverage | 15 |
| Metrics, labels and histogram correctness | 15 |
| Safe logging and trace privacy | 15 |
| SLO burn-rate and budget calculations | 15 |
| Synthetic probes and fault detection | 10 |
| Dashboard design and telemetry freshness | 10 |
| Runbooks, on-call routing and incident drills | 10 |
| Security, cost, evidence and change controls | 10 |
| **Total** | **100** |

**Acceptance recommendation:** at least 90/100 **and** no critical privacy leakage, unowned page, missing SLI or unverified rollback path. Score is a review rubric, not an automatically verified certification.

## 9. Exit criteria
- [ ] All local synthetic tests executed and passing.
- [ ] SLI taxonomy and actual exporter mapping approved.
- [ ] Metrics, logs and traces show no sensitive content.
- [ ] Alert rules, probes and dashboards reviewed for missing-data handling.
- [ ] Incident simulations and runbooks signed off.
- [ ] Independent technical and security review completed.
- [ ] Production-like execution evidence and acceptance owner recorded.

## 10. References
- [OpenTelemetry documentation](https://opentelemetry.io/docs/)
- [Prometheus alerting rules](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/)
- [Google SRE Workbook — Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/)
- [Grafana dashboard documentation](https://grafana.com/docs/grafana/latest/dashboards/)

**Part 08 complete on publication of this chapter. Next: Chapter 066 — Inference Benchmark Design and Traffic Cohorts (Part 09).**
