# Chapter 11.7 — Enterprise Observability and Incident Detection Capstone Lab

**Part 11 | Final enterprise SRE assessment | Live execution pending**

## Mission and scope

Prove that a TrueFoundry-hosted inference service can be observed, diagnosed and recovered across gateway, Kubernetes, vLLM, GPU and telemetry backends. Use **only approved synthetic staging traffic**. No real prompts, responses, PHI, tokens, model weights or customer data.

**[SAFE-READ]** inspect; **[LAB-MUTATE]** isolated staging fault drills; **[CHANGE-CONTROL]** production changes. Hardware faults, sensitive-data leakage and destructive storage scenarios are **tabletop only** unless a separate controlled test environment is explicitly authorized.

## Lab 0 — Preflight and baseline

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get svc -A
kubectl get daemonsets -A
kubectl get events -A --sort-by=.lastTimestamp | tail -50
kubectl get servicemonitors,podmonitors,prometheusrules -A 2>/dev/null || true
kubectl top nodes 2>/dev/null || true
```

Record owner, approval, cluster, namespace, model alias/version, GPU type/MIG, TrueFoundry agent, vLLM/driver, Prometheus/OTel/logging backend, synthetic load limits, SLO, alert receivers and rollback contact. Inventory actual metric names; missing sources are gaps, not zero.

## Lab 1 — End-to-end synthetic request

Send approved synthetic request and capture gateway request outcome, streaming completion, TTFT, TPOT, safe structured logs, trace ID, runtime queue/KV and GPU telemetry where supported. Align UTC timestamps and deployment revision. Note instrumentation gaps.

## Lab 2 — Dashboards, SLOs and routing

Verify availability and streaming SLIs, budget calculations, Grafana no-data handling, Prometheus rule syntax/evaluation, Alertmanager grouping/receivers and one non-production notification. Never page a production on-call through test routing.

## Lab 3 — Five game days

1. **Gateway latency/errors:** inject a controlled synthetic failure or use replay/tabletop; correlate RED, logs and traces.
2. **Inference saturation:** bounded concurrency ramp; stop on approved thresholds; correlate TTFT, queue, KV cache and GPU.
3. **Pod readiness/restart:** disposable staging workload; correlate Kubernetes Events and previous logs.
4. **Telemetry degradation:** disposable Collector/backend interruption or tabletop; quantify drops, blind window and recovery.
5. **Incident commander exercise:** combine alerts, communications, mitigation, rollback and post-incident evidence.

For each game day record trigger, impact, alert latency, time to acknowledge, diagnosis, containment, rollback, service recovery and follow-up. Unsafe fault injection remains tabletop.

## Thirty-six troubleshooting scenarios

| # | Scenario | Investigation |
|---|---|---|
| 01 | Gateway 5xx | RED and traces |
| 02 | Gateway timeout | Upstream/downstream |
| 03 | HTTP 200 but stream incomplete | Completion SLI |
| 04 | TTFT p95 regression | Prefill/queue |
| 05 | TPOT regression | Decode |
| 06 | Input token mix shift | Workload |
| 07 | Inference queue growth | Saturation |
| 08 | KV cache exhaustion | Memory |
| 09 | vLLM preemption | Scheduler |
| 10 | CUDA OOM | Device memory |
| 11 | Pod OOMKilled | cgroup |
| 12 | GPU allocated but CUDA unavailable | Driver/runtime |
| 13 | GPU utilization low | CPU/network |
| 14 | GPU XID/ECC | Hardware |
| 15 | GPU thermal throttle | Device health |
| 16 | DCGM exporter absent | Metrics |
| 17 | MIG attribution wrong | Topology |
| 18 | Model load failure | Artifact/permissions |
| 19 | Pod Pending | Scheduling |
| 20 | CrashLoopBackOff | Events/logs |
| 21 | TrueFoundry agent errors | Reconciliation |
| 22 | Prometheus target DOWN | Scrape |
| 23 | Recording rule invalid | Validation |
| 24 | Cardinality explosion | TSDB |
| 25 | Remote-write lag | Queue |
| 26 | Logs dropped by collector | Backpressure |
| 27 | Kubernetes Events expired | Retention |
| 28 | Traceparent lost | Propagation |
| 29 | Trace backend unavailable | Export |
| 30 | Tail sampling loses trace | Sampling |
| 31 | No-data displayed healthy | Coverage |
| 32 | SLO denominator wrong | Eligibility |
| 33 | Burn-rate alert too noisy | Thresholds |
| 34 | Alertmanager routing broken | On-call |
| 35 | Sensitive prompt in telemetry | Security incident |
| 36 | Rollback fails to restore baseline | Recovery |

For every scenario document expected/actual, UTC time, environment, sanitized evidence, hypothesis, mitigation, rollback and reviewer. Record simulated versus actually executed tests.

## Twelve critical acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G1 | Approved scope, safety limits and rollback | Pending |
| G2 | Full telemetry source inventory and gap register | Pending |
| G3 | Logs/Events and privacy controls verified | Pending |
| G4 | Prometheus scrape, metrics and recording rules verified | Pending |
| G5 | Traces correlated across supported services | Pending |
| G6 | GPU/vLLM TTFT, TPOT, queue and KV telemetry verified | Pending |
| G7 | Dashboards accurate with no-data policy | Pending |
| G8 | SLIs, SLOs and error-budget math approved | Pending |
| G9 | Alert routing, notification and escalation tested | Pending |
| G10 | Five game days executed or marked tabletop with gaps | Pending |
| G11 | Recovery, rollback, cleanup and blind-window evidence | Pending |
| G12 | SRE/security/service-owner sign-off | Pending |

**Pass** requires timestamped evidence, score at least 85/100 and all critical gates; a tabletop does not count as executed recovery. Missing customer-impact metrics, exposed sensitive data, broken paging or unverified rollback block production acceptance.

## 100-point assessment

| Domain | Points | Earned |
|---|---:|---:|
| Architecture/source inventory | 10 | Pending |
| Logs, Events and privacy | 10 | Pending |
| Metrics and cardinality | 10 | Pending |
| Tracing/correlation | 10 | Pending |
| GPU/vLLM metrics | 15 | Pending |
| Dashboards/coverage | 10 | Pending |
| SLIs/SLOs/error budget | 10 | Pending |
| Alert routing | 10 | Pending |
| Incident response/recovery | 10 | Pending |
| Governance/sign-off | 5 | Pending |
| **Total** | **100** | **Pending** |

## Evidence package and cleanup

Provide: service map, version inventory, approved metric dictionary, sanitized log/trace samples, PromQL/rules validation, dashboard captures, SLO spec, alert delivery evidence, five game-day timelines, rollback commits, unresolved gaps and reviewer decisions. Restore staging configuration and confirm baseline health. Never publish secrets or private telemetry in the repository.

**Part 11 publication complete: 7/7 chapters and labs. Formal review and live execution pending.**
