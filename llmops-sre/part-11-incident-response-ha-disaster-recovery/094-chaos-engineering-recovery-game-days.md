# Chapter 094 — Chaos Engineering and Recovery Game Days

**Part 11 — Incident Response, HA and Disaster Recovery · Chapter 094 of 100**  
**Status:** Published with offline lab. Live staging fault injection, independent security review and operational acceptance pending.

## Objectives
Design hypothesis-driven chaos experiments for LLM serving, RAG, agent/tool workflows and control-plane dependencies. Establish blast-radius boundaries, safety interlocks, steady-state SLOs, recovery criteria and evidence-based game-day reports.

## 1. Chaos engineering versus random disruption
A chaos experiment tests a **falsifiable reliability hypothesis** under a controlled fault. It is not permission to terminate arbitrary pods or delete state. Define the workload, failure mode, owner, maximum blast radius, rollback mechanism, monitoring, stop conditions and authorized execution window before injection.

```text
  Hypothesis + SLO + safety approval
                 |
    synthetic baseline / dry run
                 |
     controlled fault injection
                 |
  metrics + traces + user outcomes
                 |
   abort / rollback / recovery
                 |
   compare hypothesis / evidence
                 |
        RCA / COA / backlog
```

## 2. LLM-specific steady state
HTTP 200 is insufficient. Record request success, 429/5xx, p95 time-to-first-token, inter-token latency, full stream completion, tool-call success, answer grounding, safety/refusal behavior, queue age, token throughput, GPU memory pressure, tenant fairness and cost. Use synthetic prompts and safe fixtures; never leak raw PHI, secrets or tenant data into fault logs.

## 3. Fault catalog
| Experiment | Failure domain | Hypothesis | Abort condition |
|---|---|---|---|
| GPU worker termination | serving data plane | N-1 headroom preserves success/TTFT | error-budget burn |
| zone route loss | availability | surviving zones absorb load | sustained 5xx/latency |
| model registry outage | artifact/control | warmed replicas remain available | failed model restarts |
| KServe webhook failure | admission | existing inference stays healthy; promotions stop | policy bypass |
| Ray Serve controller disruption | orchestration | established routes continue where supported | request failures |
| Redis cache outage | dependency | source protection prevents stampede | DB saturation |
| vector index lag | RAG | freshness alarm blocks stale answers | ACL/grounding failure |
| tool API 429/timeout | agents | bounded retries and idempotency | duplicate side effects |
| Kafka backlog | async ingestion | lag alarms and replay restore freshness | unrecoverable offset |
| regional failover | DR | RPO/RTO and security gates hold | split-brain or residency |
| cost/usage ledger delay | FinOps | admission remains within budget | unbounded spend |
| observability pipeline outage | telemetry | fallback health checks still detect incidents | blind operation |

## 4. Safety and governance
- Prefer a disposable, isolated staging cluster and synthetic data.
- Identify an incident commander, experiment owner, observer, security approver and rollback operator.
- Define exact namespaces, tenant IDs, workloads, fault scope and maximum duration.
- Require dry run and abort conditions, including data integrity, cross-tenant access, spend and latency.
- Use least-privilege service accounts and change tickets; no blanket cluster-admin or disabled admission policies.
- Preserve immutable release and model signatures; do not inject destructive faults into shared stateful infrastructure.
- Validate alert routing and human acknowledgment before injecting.
- Keep a manual kill switch and independent control channel.
- Avoid concurrent chaos experiments unless explicitly designed and approved.

## 5. Experiment template
```yaml
experiment_id: llm-chaos-094-gpu-worker
environment: isolated-staging
hypothesis: "N-1 GPU capacity sustains approved inference SLO"
scope:
  namespace: llm-chaos-staging
  tenant: synthetic-tenant
  max_affected_replicas: 1
steady_state:
  min_success_rate: 0.99
  max_p95_ttft_ms: 1800
  max_stream_failure_rate: 0.01
guardrails:
  require_owner_approval: true
  require_rollback_ready: true
  require_tenant_isolation: true
  require_budget_guard: true
abort:
  on_security_violation: true
  on_data_loss: true
  on_excess_error_budget_burn: true
recovery:
  require_slo_restored: true
  require_model_digest_verified: true
```
This is a **planning example**, not a Chaos Mesh/Litmus executable resource. Adapt to the installed fault-injection framework and approved API version.

## 6. Game-day workflow
1. Choose a recent incident or documented reliability risk and write a testable hypothesis.
2. Baseline real workload mix using synthetic traffic and pinned model/prompt/index revisions.
3. Calculate N-1 safe GPU capacity, failure budget, dependency limits and cost ceiling.
4. Validate rollback, kill switch, incident communications and observability.
5. Obtain change/security approvals and verify test isolation.
6. Start synthetic traffic; capture timestamped baseline and expected behavior.
7. Inject one bounded fault and monitor signals and downstream dependencies.
8. Abort immediately on security/privacy/data-integrity or agreed SLO threshold violations.
9. Restore faulted component; verify actual inference and quality, not just pod readiness.
10. Record time-to-detect, time-to-mitigate, time-to-recover, RPO/RTO and error-budget cost.
11. Produce evidence, gaps, named remediation owners and retest plan.

## 7. Read-only Kubernetes reconnaissance
```bash
kubectl config current-context
kubectl get nodes -L topology.kubernetes.io/zone
kubectl get deploy,sts,pods,pdb -n llm-chaos-staging
kubectl get events -n llm-chaos-staging --sort-by=.lastTimestamp
kubectl get hpa -n llm-chaos-staging
kubectl get endpointslices -n llm-chaos-staging
```
Do not execute `kubectl delete`, node drain, network fault or region cutover in production based solely on this tutorial.

## 8. Incident runbooks
**RB-094-A — Preflight fails:** cancel injection; correct missing approval, monitoring, capacity, isolation or rollback readiness; rerun preflight.

**RB-094-B — Unexpected blast radius:** activate kill switch, halt all injections, isolate synthetic traffic, page incident commander and security owner as indicated.

**RB-094-C — SLO breach:** stop fault, restore last approved route/replicas, cap retries and admission, verify streaming and customer-visible success.

**RB-094-D — Data integrity or tenant boundary:** stop immediately, fence side-effecting workers, preserve sanitized evidence, invoke security/data response; do not resume until approved.

**RB-094-E — Failed recovery:** inspect scheduling, GPU device plugin, artifact cache, controller reconciliation, gateway and dependency readiness; escalate and record actual RTO.

**RB-094-F — Game-day closeout:** compare hypothesis to observed evidence, score detection/mitigation, open corrective actions, assign deadlines and schedule retest.

## 9. Experiment reporting
Record experiment ID, ticket, UTC timestamps, immutable revision IDs, affected failure domain, baseline and post-fault metrics, security/financial impact, whether abort fired, observed recovery time, unexpected dependencies, confidence limitations, RCA/COA and sign-offs. A failed hypothesis is useful evidence; it is not a reason to suppress or rewrite results.

## 10. Assessment rubric — 100 points
| Domain | Points |
|---|---:|
| Hypothesis, failure modes and blast radius | 20 |
| SLOs, safety and authorization | 20 |
| Controlled injection and telemetry | 20 |
| Abort, rollback and recovery validation | 20 |
| Evidence, runbooks and remediation tracking | 20 |
| **Total** | **100** |

Educational pass >=90/100 with zero critical safety, privacy, data-integrity or tenant-isolation violations. Offline simulation does not prove production resilience.

## 11. Companion lab
[Chapter 094 — Executable Chaos Experiment Safety Gate Lab](../labs/094-chaos-engineering-recovery-game-days-check.md) provides deterministic preflight, abort and recovery checks with pytest.

## References
- [Principles of Chaos Engineering](https://principlesofchaos.org/)
- [Google SRE Workbook](https://sre.google/workbook/)
- [Chaos Mesh](https://chaos-mesh.org/docs/)
- [LitmusChaos](https://docs.litmuschaos.io/)
- [Kubernetes Pod Disruptions](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)

**Next:** Chapter 095 — Incident Response and Disaster Recovery Acceptance Lab.
