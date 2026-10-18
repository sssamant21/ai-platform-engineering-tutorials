# Chapter 095 — Incident Response and Disaster Recovery Acceptance Lab

**Part 11 — Incident Response, HA and Disaster Recovery · Chapter 095 of 100 · Part 11 capstone**  
**Status:** Published with an offline executable acceptance harness. Real multi-system staging drills, independent technical/security review and production sign-off pending.

## Mission
Prove that an LLM platform can detect, contain, mitigate and recover from correlated incidents while maintaining model provenance, tenant isolation, safety, durable state and agreed SLO/RPO/RTO. This capstone integrates Chapters 086–094 into a measurable, evidence-backed acceptance exercise.

## 1. Enterprise scenario
A regional GPU-zone failure coincides with a degraded model registry and delayed vector-index replication. An agent tool has an in-flight external side effect, while the secondary region is available but has an older retrieval checkpoint.

```text
       Global gateway / policy / budgets
                /                \
      Primary region          DR region
      GPU inference           GPU standby
      Ray/KServe/TFY          approved runtime
      vector index           replicated index
      state + tool queue      restored state
                \                /
         signed artifacts / registry
                   |
          telemetry + incident command
```

The incident commander must decide whether to preserve primary traffic, fail over, degrade safely, or stop affected tasks. **Do not promote a region solely because its pods are Ready.**

## 2. Required deliverables
- Architecture and dependency graph: serving, gateway, model artifacts, RAG, tool services, databases, event streams, IAM/KMS, cost ledger and observability.
- Approved incident severity, communications, roles and escalation tree.
- Defined SLOs: inference availability, TTFT, stream completion, tool correctness, grounding and safety.
- Per-dependency RPO/RTO, last known-good signed release and immutable configuration.
- N-1 GPU capacity and multi-zone placement evidence.
- Primary writer/tool fencing and idempotency strategy.
- Staging fault injection plan, abort conditions and rollback authority.
- Actual recovery measurements, customer-impact estimate and audit evidence.
- RCA/COA with owners, deadlines and retest criteria.

## 3. Integrated failure sequence
| Phase | Injected scenario | Required evidence |
|---|---|---|
| A | Inference 5xx/timeout spike | SEV, alert and error-budget analysis |
| B | GPU worker/zone loss | N-1 throughput, queue and TTFT |
| C | Ray Serve/KServe control-plane degradation | desired vs observed revision, conditions |
| D | Model registry unavailable | verified cached manifest and signatures |
| E | Model/tokenizer mismatch | digest/compatibility gate blocks rollout |
| F | RAG replication lag | freshness/RPO and tenant ACL checks |
| G | Agent tool timeout | idempotency, retry cap and side-effect fencing |
| H | DR regional failover | writer fencing, residency, KMS, capacity |
| I | Failback | reverse replication, safe routing and SLO recovery |
| J | Closeout | timeline, RCA/COA and remediation retest |

Execute one bounded fault at a time unless a separately approved combined-fault experiment is part of the test. Never use production credentials or sensitive prompts in the lab.

## 4. Acceptance gates
**G1 — Incident command:** on-call alert, SEV classification, named incident commander, communication and escalation.

**G2 — Availability:** external synthetic inference, stream completion, TTFT and error-budget signals recover within approved SLOs.

**G3 — N-1:** measured remaining-zone safe capacity supports peak traffic or documented admission controls protect the platform.

**G4 — Provenance:** model, tokenizer, runtime, prompts and policies are pinned and verified; no unsigned fallback.

**G5 — Data recovery:** each authoritative dependency meets its own RPO; backup restore and replay consistency verified.

**G6 — Fencing:** primary writers and side-effecting workers cannot operate concurrently with promoted DR writers.

**G7 — Security:** IAM/KMS, tenant isolation, retrieval ACL, privacy and residency controls remain enforced.

**G8 — Budget:** token spend, retries, GPU standby and external API usage remain bounded and reconciled.

**G9 — Recovery:** measured RTO meets target; no open critical safety or data-integrity violations.

**G10 — Evidence:** sanitized UTC timeline, logs/metrics, approvals, RCA/COA, owner and retest schedule.

All critical gates are mandatory. A high numerical score cannot override a security, provenance, split-brain or data-loss failure.

## 5. Read-only staging reconnaissance
```bash
kubectl config current-context
kubectl get nodes -L topology.kubernetes.io/zone
kubectl get deploy,sts,pods,pdb,hpa -A
kubectl get events -A --sort-by=.lastTimestamp
kubectl get endpointslices -A
kubectl get crd | grep -Ei 'ray|serving|inference'
```
Use platform-version-matched TrueFoundry, Ray Serve and KServe commands to inspect controllers and release revisions. Production mutation, destructive database restores, credential rotation and region cutover require separate authorized runbooks.

## 6. Game-day timeline
**T-7 days:** approve scope, define SLOs and RPO/RTO, prepare synthetic workloads, confirm safe GPU capacity and signed backups.

**T-1 day:** check observers, dashboards, rollback, change tickets, fault isolation, emergency communications and abort mechanisms.

**T+0:** record baseline; start isolated synthetic inference and agent workflows.

**T+5:** inject approved bounded fault; detect and classify customer impact.

**T+10:** freeze promotions, check fault domains, activate containment and measure recovery.

**T+20:** if failover is needed, verify writer fencing, recovery checkpoints, model provenance, IAM/KMS and region residency.

**T+30:** restore healthy serving, retrieval freshness, tool state and usage ledger; validate end-to-end inference.

**T+60:** close or escalate based on real RTO and SLO evidence; capture lessons and remediation tasks.

The times are an illustrative rehearsal timeline, **not guaranteed recovery durations**.

## 7. Runbooks
**RB-095-A — Incident declaration:** page incident commander, freeze risky changes, establish communication and timestamped evidence.

**RB-095-B — Correlated failure triage:** separate GPU/data-plane, orchestration/control-plane, model artifact, RAG, tools and durable state.

**RB-095-C — Regional failover:** prove secondary readiness, per-dependency RPO, fencing, signed model identity, IAM/KMS and residency before canary routing.

**RB-095-D — Security or data-integrity breach:** halt fault injection and promotion, preserve sanitized evidence, escalate security/data owners and prohibit unsafe fallback.

**RB-095-E — Failback:** reverse replication, verify authoritative state, enforce single writer, validate model/config parity and canary original region.

**RB-095-F — Acceptance closeout:** report actual measurements and uncertainty, assign COA owners and due dates, require retest for failed gates.

## 8. Troubleshooting
| Symptom | Investigation | Response |
|---|---|---|
| Ready pods but 503 | endpoints, gateway, warmup | validate external inference |
| failover exceeds RTO | artifact transfer, GPU cold start | improve warm standby and runbook |
| RPO breach | replication/backup lag | block promotion or approved degraded mode |
| duplicate tool action | idempotency/fencing | stop workers, reconcile side effects |
| wrong model after rollback | manifest/tokenizer/config drift | restore signed compatible release |
| post-failover cost spike | retries, token reservations, GPU | cap admission, reconcile ledger |
| missing audit evidence | telemetry/control-plane outage | stop acceptance, repair evidence path |

## 9. 100-point acceptance rubric
| Domain | Points |
|---|---:|
| Incident command, alerting and communications | 10 |
| Fault diagnosis and evidence | 10 |
| N-1 and multi-zone recovery | 15 |
| Model/registry integrity and compatibility | 10 |
| Data backup, RPO and state consistency | 15 |
| Fencing, IAM, tenant isolation and residency | 15 |
| End-to-end inference SLO and RTO | 15 |
| RCA/COA, audit and retest plan | 10 |
| **Total** | **100** |

**Educational target: >=90/100 plus all critical gates passed.** A lab simulation pass is not operational certification. Publish final Part 11 acceptance only after real staging tests and independent sign-off.

## 10. Companion executable lab
[Chapter 095 — Integrated Incident Response and DR Acceptance Lab](../labs/095-incident-response-disaster-recovery-acceptance-lab-check.md) implements a deterministic acceptance evaluator and pytest suite.

## References
- [Google SRE Workbook — Incident Response](https://sre.google/workbook/incident-response/)
- [Google SRE Workbook — Disaster Recovery](https://sre.google/workbook/disaster-recovery/)
- [Kubernetes Pod Disruptions](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)
- [NIST Contingency Planning Guide SP 800-34](https://csrc.nist.gov/pubs/sp/800/34/r1/upd1/final)

**Next:** Chapter 096 — Enterprise LLM Inference Reference Architecture, beginning Part 12.
