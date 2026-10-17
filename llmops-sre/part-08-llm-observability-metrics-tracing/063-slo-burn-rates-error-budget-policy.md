# Chapter 063 — SLO Burn Rates and Error Budget Policy

**Part 08 — LLM Observability, Metrics and Tracing**  
**Status:** Tutorial and companion lab published; technical/security review and live execution pending.

## Learning objectives
1. Define user-centric LLM inference SLIs and rolling-window SLOs.
2. Calculate allowed errors, error budget remaining and burn rate.
3. Implement multi-window multi-burn-rate (MWMB) Prometheus alert rules.
4. Separate request availability, first-token latency and streaming completion from infrastructure health.
5. Operate error-budget policies, incident escalation and release gates without confusing traffic volume and reliability.

## 1. Architecture and SLI contract

```text
client -> gateway -> auth -> inference queue -> model -> streaming response
             |                 |                     |
        requests/errors    TTFT, queue         completed stream
             \_________________|_____________________/
                           |
                  canonical SLI counters
                           |
                 Prometheus recording rules
                           |
              multi-window burn-rate alerts
                           |
                Alertmanager -> on-call
                           |
            error-budget review -> change policy
```

**Define user journeys before calculating SLOs.** A 200 response may be an unsuccessful streaming request if it disconnects before completion. Separate SLIs for availability, first-token latency, and stream completion; do not assume all users or tenants have identical expectations.

### Suggested SLI definitions
| SLI | Eligible events | Bad events | Explicit exclusions |
|---|---|---|---|
| Availability | accepted eligible inference attempts | serving-side 5xx, timeout, failed admission | client-invalid 4xx per policy |
| First-token latency | eligible streaming requests | first token later than objective or no first token | caller cancellations only if policy permits |
| Stream completion | eligible started streams | server termination, incomplete generation | deliberate user cancel, if reliably distinguished |

**Do not treat 429s as automatically excluded:** overload rejection can be a service failure. Define classification with product owners and test it. Instrument denominator and bad-event counters at the same boundary, with stable bounded labels. A trace sample is not an SLI denominator. Metrics in this chapter are **illustrative contracts** and must be mapped to real gateway instrumentation.

## 2. SLO mathematics
Let target availability be S, eligible events N, and bad events B over the same rolling window:
- Error-budget fraction = `1 - S`.
- Allowed bad events = `N × (1 - S)` (fractional planning value; integer enforcement policy must be defined).
- Consumed budget fraction = `B / (N × (1 - S))`, when N > 0.
- Remaining budget fraction = `1 - consumed`; can be negative when overspent.
- Burn rate = `(B / N) / (1 - S)`.

Example: 99.9% SLO, 1,000,000 eligible requests, 1,500 bad requests: allowance 1,000; consumed 150%; remaining −50%; burn rate 1.5 over that window.

### Multi-window burn-rate logic
For a 30-day rolling SLO, common **illustrative** alert pairs:
| Severity | Long / short windows | Burn threshold | Budget spent if sustained for long window |
|---|---|---:|---:|
| Page fast | 1h / 5m | 14.4 | 2% |
| Page slow | 6h / 30m | 6 | 5% |
| Ticket fast | 1d / 2h | 3 | 10% |
| Ticket slow | 3d / 6h | 1 | 10% |

The percent column is `burn × long-window-duration / 30d`; it is not the observed error budget consumed. Require **both** long and short windows above threshold to avoid paging for a brief spike or an already-recovered incident. Calibrate windows for request traffic and scrape/evaluation intervals; low-traffic services may need event-count gates and longer windows.

## 3. Prometheus implementation
Assume two counters:
- `llm_sli_requests_total{service,environment}`: eligible requests.
- `llm_sli_bad_requests_total{service,environment}`: bad requests.

Both must be monotonic counters from the same logical observation boundary. Instrument bad events as a subset of eligible events; verify `bad <= eligible` over equivalent intervals.

Example 1h burn rate for 99.9% target:
```promql
(
  sum by (service, environment) (increase(llm_sli_bad_requests_total[1h]))
  /
  clamp_min(sum by (service, environment) (increase(llm_sli_requests_total[1h])), 1)
) / 0.001
```
This count-based ratio is easier to reason about for sparse traffic. For alerting use `rate` with traffic gates and matching aggregation; remember that `rate` and `increase` extrapolate over samples. Use `or vector(0)` only when a missing series genuinely means zero; missing instrumentation should have its own alert.

### Recording rules and alert rules
See the executable companion lab for all four windows and tested alert assertions. A production ruleset should include service ownership, severity, environment scoping, documented SLO target and a runbook URL. Avoid one-size-fits-all paging thresholds without reliability policy approval.

## 4. Error-budget policy
**Green (budget healthy):** normal release cadence, automated safety gates, review SLI drift.

**Amber (budget rapidly burning):** investigate the burn, slow risky releases, prioritize rollback readiness, add temporary canaries and capacity checks.

**Red (budget exhausted or sustained page):** pause nonessential risky changes under an approved policy, prioritize reliability remediation, and require accountable exception approval for urgent security/incident changes.

Error budgets are a decision mechanism, not a punishment. Review exclusions, traffic mix, missing data, dependency failures, business-critical journeys and deployment impact before applying a freeze. Define whether policy uses calendar-month or rolling 30-day windows; this chapter's examples use rolling 30 days.

## 5. Production incident examples
1. **Inference 5xx spike:** error ratio 3%; 99.9% objective → burn 30; check both 1h and 5m windows before page.
2. **Slow first token:** availability healthy, TTFT SLO violated; avoid hiding user pain behind aggregate success rates.
3. **Streaming partial failures:** HTTP status 200 but response incomplete; completion SLI catches the failure.
4. **Idle service:** denominator zero; suppress meaningless ratio pages and separately check synthetic probes and target availability.
5. **Telemetry loss:** missing numerator/denominator must not be interpreted as zero errors; alert on instrumentation gaps.
6. **Regional outage:** assess traffic-weighted global SLO and region-specific impact separately; do not average per-region percentages.

## 6. Troubleshooting decision tree
- Confirm the user journey, event taxonomy and instrumentation version.
- Validate numerator is a subset of denominator and counter reset handling.
- Check scrape gaps, target health, rule evaluation and label joins.
- Compare long/short burn windows and actual event counts.
- Correlate errors with deployment revisions, queue saturation, GPU metrics, traces and incident timelines.
- Escalate to the owning service, not solely the metrics team.

## 7. Production runbooks

### RB-063-A — Fast burn page
**[SAFE-READ]** identify service/environment and windows; verify real user failures and recent changes; compare request volume, status classification, queue and TTFT. **[CHANGE-CONTROL]** initiate approved rollback, traffic shift or capacity mitigation. Track incident commander, start/end times and budget impact.

### RB-063-B — Budget exhaustion review
Calculate rolling eligible and bad counts, record exclusions and data completeness, review incident root causes, document reliability work and determine whether the release policy moves to red. Obtain explicit exception approvals for emergency fixes.

### RB-063-C — False positive / missing telemetry
Inspect scrape health, counter monotonicity, label drift, `for` state and alert routing. Restore instrumentation or update rules through reviewed change control. Never solve missing data by assuming all requests succeeded.

### RB-063-D — SLO policy governance
Quarterly review user journeys, thresholds, alert fatigue, traffic gates, on-call routing, retention, security controls and exception records with product, SRE and security owners.

## 8. Security, FinOps and acceptance
- **[SAFE-READ]** read Prometheus/Alertmanager state; **[LAB-MUTATE]** local synthetic tests; **[CHANGE-CONTROL]** production policy and alert changes.
- No prompt, completion, tenant ID, request ID or token strings in metric labels.
- Budget the number of recorded series and rule evaluation CPU before rollout.
- [ ] SLI taxonomy and exclusions approved.
- [ ] Multi-window alert tests pass with traffic and missing-data cases.
- [ ] Error-budget calculations independently checked.
- [ ] Runbooks and on-call ownership documented.
- [ ] Technical/security reviews and live execution recorded separately.

## 9. References
- [Google SRE Workbook — Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/)
- [Google SRE Workbook — Error Budget Policy](https://sre.google/workbook/error-budget-policy/)
- [Prometheus recording rules](https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/)
- [Prometheus unit testing rules](https://prometheus.io/docs/prometheus/latest/configuration/unit_testing_rules/)

**Next:** Chapter 064 — Dashboards, Synthetic Probes and On-Call.
