# Chapter 099 — Run Compound Failure and Incident Recovery Exercises

**Part 12 — Final Production AI SRE Capstone · Chapter 099 of 100**  
**Publication:** Complete. **Offline execution, staging game days, independent security review and production acceptance:** Pending.

## Objectives and prerequisites
Combine Chapters 086–098 into controlled compound-failure experiments across GPU inference, Ray Serve/KServe/TrueFoundry reconciliation, RAG, tool execution, model registry, observability and regional disaster recovery. Demonstrate detection, containment, fencing, recovery, verification, customer communications and RCA/COA with measured SLO/RPO/RTO. Prerequisites: approved synthetic staging environment, signed model manifests, GPU capacity baseline, trace/metric dashboards, isolated test tenants, restore-tested backups, incident commander and independent kill switch.

## 1. Enterprise incident scenario
```text
synthetic clients --> gateway / identity / budget --> model router
                                                   |
                                        serving control plane
                                        /        |         \
                                 GPU zone A  GPU zone B  GPU zone C
                                        \        |         /
                                        model weights / tokenizer
                                                 |
                                RAG vector store / source DB
                                                 |
                              tool broker / Kafka / usage ledger
                                                 |
                            OTel traces / metrics / alert routing
                                                 |
                            regional standby + fenced recovery
```
At T+0, GPU zone A degrades; at T+5, a model registry endpoint becomes unavailable; at T+10, retrieval index freshness falls behind its approved RPO. Simultaneously, a tool call times out after an external side effect may have occurred. Incident command must preserve tenant isolation, artifact provenance, safety, spending and exactly-once *business effect* through idempotency and reconciliation, rather than assume network retries are safe.

## 2. Experiment contract and safety boundaries
Every experiment requires an ID, hypothesis, failure domains, change ticket, owner, observer, rollback operator, explicit namespace/tenant, maximum duration, permitted injection mechanism, abort thresholds and evidence links. Run only in an isolated staging environment using synthetic traffic. Never inject a fault into production, shared clusters, real customer data, live billing or third-party APIs without separate authorization.

**Critical abort conditions:** suspected cross-tenant access, privacy exposure, data corruption, unfenced dual writers, unsigned model promotion, uncontrolled tool side effects, loss of observability, budget ceiling breach or unplanned blast radius. Stop injection immediately and invoke incident/security procedures; SLO score cannot override these gates.

## 3. Baseline and measurable objectives
| SLI | Measurement | Illustrative acceptance |
|---|---|---|
| inference availability | completed successful eligible requests / eligible requests | >=99.9% |
| p95 TTFT | latency to first token for eligible streams | <=1800 ms |
| p95 TPOT | output token cadence for eligible streams | <=100 ms |
| streaming completion | terminal completion events / started streams | >=99.9% |
| retrieval freshness | latest authoritative checkpoint minus served index checkpoint | within approved RPO |
| tool safety | reconciled, authorized side effects and idempotency | no duplicate effects |
| recovery time | failure onset to validated restored service | within approved RTO |
| N-1 capacity | safe zone throughput after largest zone fails | above peak demand + reserve |
| budget | actual plus reserved spend | within approved ceiling |

These are exercise defaults, not universal production targets. Measure from client-observed outcomes, use versioned workload profiles and synthetic tenants, and never average per-pod p95 percentiles. Include actual model revision, input/output token distribution, cache state, queue time, GPU and KV-cache saturation, gateway throttling, retry count and cost.

## 4. Controlled game-day sequence
| Phase | Fault | Hypothesis | Required proof |
|---|---|---|---|
| 0 | baseline | clean serving and retrieval | SLO, quality, policy and spend baseline |
| 1 | single GPU worker unavailable | N-1 absorbs load | queue/TTFT/stream success |
| 2 | largest zone unavailable | topology and admission protect service | safe throughput and zone placement |
| 3 | registry API unavailable | pinned warmed artifacts continue | signed model/tokenizer digests |
| 4 | Ray/KServe controller stalled | existing serving remains stable where supported | reconciler and data-plane evidence |
| 5 | RAG index lags | freshness gate blocks stale retrieval | source/index checkpoint comparison |
| 6 | agent tool times out | no duplicated external effect | idempotency and ledger reconciliation |
| 7 | telemetry export delayed | independent health path detects impact | alert fallback and UTC timeline |
| 8 | primary region fails | DR promotion satisfies fencing, RPO/RTO, residency | recovery and canary probes |
| 9 | failback | single writer and consistent artifacts | reverse replication and traffic validation |

Do **not** automatically chain faults: obtain separate approval for correlated injection. Abort immediately on critical gates, and retain an auditable record of any skipped phase.

## 5. Read-only diagnostic commands
```bash
kubectl config current-context
kubectl get nodes -L topology.kubernetes.io/zone
kubectl get pods,deploy,sts,hpa,pdb -n llm-staging -o wide
kubectl get events -n llm-staging --sort-by=.lastTimestamp
kubectl get endpointslices -n llm-staging
kubectl get networkpolicy,resourcequota,serviceaccount -n llm-staging
```
For TrueFoundry, Ray Serve, KServe, Kafka, database replication and model registries, use approved version-matched inspection endpoints. Inspect GPU DCGM telemetry and application traces from authorized monitoring systems. No production-mutating command is provided.

## 6. Runbooks
**RB-099-A — Detection and command:** Declare SEV, page incident commander, freeze deployments, establish UTC timeline and communications, capture user impact and error-budget burn.

**RB-099-B — GPU/zone loss:** Check node conditions, device plugin, scheduling, pod disruption budget, KV pressure and per-zone safe throughput. Enforce admission caps instead of retry amplification.

**RB-099-C — Registry/control plane:** Keep existing signed approved artifacts; halt unverified rollouts; compare desired and observed revisions. Do not bypass signature policy.

**RB-099-D — RAG/tool inconsistency:** Gate stale retrieval by approved RPO; fence tool workers; reconcile idempotency keys, Kafka offsets and external side effects before replay.

**RB-099-E — DR failover:** Verify primary writer fencing, restore point, immutable artifacts, KMS/IAM, tenant residency, GPU quota, synthetic canary, quality and budget before traffic shift.

**RB-099-F — Abort and recovery:** Stop injection, restore previous approved route and model, verify terminal streams and retrieval ACLs, then escalate if RTO is missed.

**RB-099-G — Failback:** Reverse replication and reconcile ledgers; confirm single writer, artifact parity, canary and quality before routing back.

**RB-099-H — Closeout:** Preserve sanitized evidence, record actual detection/mitigation/recovery times, RCA/COA, owners, due dates and retest results.

## 7. Troubleshooting matrix
| Symptom | Likely root cause | Safe next step |
|---|---|---|
| pods Ready, clients receive 503 | gateway/endpoint/model warmup | external synthetic probe and traces |
| TTFT high after zone loss | prefill queue or insufficient N-1 | bounded admission, restore capacity |
| recovery picks wrong model | registry/cache manifest drift | digest verification and block promotion |
| stale or unauthorized RAG results | index lag or ACL regression | disable affected retrieval path |
| duplicate tool operation | unfenced worker or missing idempotency | halt workers and reconcile effects |
| DR 403 | IAM/KMS/residency mismatch | restore scoped access, no bypass |
| SLO metrics absent | telemetry exporter failure | independent health probes, abort test |
| runaway GPU/tool spend | retry storm or double reservation | cap requests and reconcile ledger |

## 8. Evidence package
Store experiment plan and approval, baseline synthetic workload revision, model/container digests, zone/GPU capacity, alerts and traces, UTC phase timeline, customer-impact proxy, signed backup and RPO checkpoint evidence, primary/tool fencing proof, cost ledger, DR synthetic probe results, actual RTO, failed/aborted phases, RCA/COA and reviewer signatures. Redact prompts, credentials, personal data and sensitive tenant IDs.

## 9. Acceptance rubric — 100 points
| Domain | Points |
|---|---:|
| Hypothesis, approvals, isolation and abort readiness | 15 |
| SLO instrumentation and fault detection | 15 |
| GPU/zone and control-plane resilience | 15 |
| Artifact, RAG and tool integrity | 15 |
| DR fencing, RPO/RTO and failback | 20 |
| Evidence, runbooks, RCA/COA and retest | 20 |
| **Total** | **100** |

Educational pass >=90/100 **and every critical gate passes**. Offline simulations alone cannot certify production reliability.

## 10. Executable lab
[Chapter 099 — Compound Failure Acceptance Simulator](../labs/099-compound-failure-incident-recovery-exercises-check.md) includes a deterministic Python 3.11+ evaluator, pytest cases, a synthetic incident timeline and staging extension checklist.

## References
- [Google SRE Workbook — Incident Response](https://sre.google/workbook/incident-response/)
- [Google SRE Workbook — Disaster Recovery](https://sre.google/workbook/disaster-recovery/)
- [Principles of Chaos Engineering](https://principlesofchaos.org/)
- [Kubernetes Pod Disruptions](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)
- [OpenTelemetry](https://opentelemetry.io/docs/)

**Next:** Chapter 100 — Final AI Inference SRE Production Readiness Assessment.
