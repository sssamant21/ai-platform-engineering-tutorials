# Chapter 11.6 — Hands-on Lab: Dashboards, SLIs, SLOs, Alerting and Error Budgets

**Part 11 | Approved staging | Live execution pending**

## Mission and safeguards

Design and validate request-based availability, streaming TTFT and completion SLIs; build dashboards, multi-window burn alerts, error-budget policy and on-call escalation. **[SAFE-READ]** inspect; **[LAB-MUTATE]** synthetic staging; **[CHANGE-CONTROL]** production. Never expose prompts, patient data, secrets or unbounded user identifiers.

## Lab 0 — Preconditions

```bash
kubectl config current-context
kubectl get pods -A -o wide
kubectl get svc -A
kubectl get prometheusrules -A 2>/dev/null || true
kubectl get events -A --sort-by=.lastTimestamp | tail -40
```

Record Prometheus/Grafana/Alertmanager versions, verified request and streaming metric names, SLO owner, environment, route/model, retention and alert contact. No metric means no SLO pass.

## Lab 1 — Define SLI contract

Document numerator/denominator, eligibility, 4xx treatment, timeout and cancellation policy, retry handling, completion semantics, target, 30-day window and traffic floor. Separate request success from streaming completion and TTFT. Verify that the instrumentation can measure each numerator.

## Lab 2 — Error-budget worksheet

For a synthetic 99.9% availability SLO and 10 million eligible requests, calculate a 10,000-request budget. Simulate 3,000 bad requests, leaving 7,000. Compute burn rate from observed bad fraction and confirm matching rolling windows. Explain why request-based budget is not fixed downtime minutes.

## Lab 3 — Dashboard validation

Build staging dashboard panels for eligible RPS, bad ratio, latency, TTFT, TPOT, stream completion, GPU/KV cache, SLO attainment, budget remaining and telemetry coverage. Confirm units, timezone, no-data display and drilldown to logs/traces.

## Lab 4 — Alert rules [LAB-MUTATE]

Use verified staging recording rules for 5m/1h error ratios and traffic counts. Add a fast-burn alert with both windows, a traffic gate, safe labels and approved receiver. Validate:
```bash
promtool check rules staging-slo-alerts.yml
```
Test one controlled synthetic burn or rule test fixture without paging production. Verify Alertmanager grouping, notification, acknowledgment and runbook link. Roll back test rules.

## Lab 5 — Error-budget policy tabletop

Rehearse release decision during healthy budget, fast burn, exhausted budget and urgent security patch. Record owner, exceptions, escalation and communications.

## Lab 6 — Cleanup and evidence

Remove test alerts/workloads, restore routing and verify baseline dashboards/targets. Capture sanitized UTC query results, alert delivery proof, rollback commit and reviewer.

## Thirty-two failure scenarios

| # | Scenario | Focus |
|---|---|---|
| 01 | Missing eligible request metric | Instrumentation |
| 02 | 4xx excluded inconsistently | Eligibility |
| 03 | Backend retries double counted | Source |
| 04 | Stream 200 but incomplete | Completion |
| 05 | TTFT metric absent | Streaming |
| 06 | Latency units mismatched | Seconds/ms |
| 07 | Histogram bucket boundary unsuitable | Threshold |
| 08 | Percentiles averaged across replicas | Aggregation |
| 09 | No-data shown as zero | Coverage |
| 10 | Counter reset causes spike | Rate |
| 11 | Low request volume pages | Traffic floor |
| 12 | High error rate hidden by retries | Client SLI |
| 13 | Error budget uses wrong window | Time range |
| 14 | Burn rate formula wrong | Target |
| 15 | Recording rule missing | Discovery |
| 16 | PrometheusRule selector mismatch | Operator |
| 17 | Alert rule syntax invalid | promtool |
| 18 | Alert `for` never satisfied | Evaluation |
| 19 | Alert labels mismatch join | PromQL |
| 20 | Alertmanager receiver wrong | Routing |
| 21 | Alert silenced accidentally | Silences |
| 22 | Alert grouping floods on-call | Dedup |
| 23 | Runbook link invalid | Operations |
| 24 | Grafana datasource unavailable | Query |
| 25 | Dashboard refresh stale | Freshness |
| 26 | GPU dashboard lacks model revision | Context |
| 27 | Sensitive identifier in metric label | Privacy |
| 28 | Model-level SLO masks tenant outage | Aggregation |
| 29 | Clock skew changes incident timeline | UTC |
| 30 | Telemetry outage misreported as healthy | Monitoring |
| 31 | Budget policy blocks security fix | Governance |
| 32 | Synthetic alert pages production | Isolation |

For each record expected/actual, UTC timestamp, metric query, safe evidence, hypothesis, mitigation, rollback and reviewer. Unsafe scenarios are tabletop.

## Twelve acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G1 | Scope and SLO owner approved | Pending |
| G2 | Availability SLI contract verified | Pending |
| G3 | Streaming TTFT/completion SLIs verified or gaps declared | Pending |
| G4 | Target, window, exclusions and traffic floor approved | Pending |
| G5 | Error-budget calculations validated | Pending |
| G6 | PromQL and recording rules validated | Pending |
| G7 | Multi-window burn alert tested | Pending |
| G8 | Dashboard accuracy and no-data behavior checked | Pending |
| G9 | Alertmanager routing, grouping and escalation tested | Pending |
| G10 | Privacy and telemetry coverage reviewed | Pending |
| G11 | Recovery, rollback and cleanup validated | Pending |
| G12 | SRE/product/security sign-off recorded | Pending |

**Pass** requires timestamped evidence and owner approval. Pending/blocked is not pass.

**Next:** Chapter 11.7 — Enterprise Observability and Incident Detection Capstone.
