# Chapter 093 — Backup, DR, RTO/RPO and Region Failover

**Part 11 — Incident Response, HA and Disaster Recovery · Chapter 093 of 100**  
**Status:** Published with an executable offline lab. Live cross-region restore, data-consistency verification, failover timing and independent security/operational review pending.

## Learning objectives
Design and rehearse disaster recovery for LLM inference, RAG, agents and platform dependencies. Define realistic recovery point objective (RPO) and recovery time objective (RTO), identify backup boundaries, test restores, prevent split-brain and unsafe cross-region traffic shifts, and verify model, data and policy consistency.

## 1. Recovery architecture
```text
                   global traffic steering
                    /                 \
             primary region       DR region
             gateway / IAM        gateway / IAM
             model serving        warm/cold standby
             vector index         restored replica
             agent/tool state     replicated state
             budget ledger        fenced ledger
                   \                 /
             immutable backups + signed artifacts
                          |
                  audit + DR orchestration
```
The disaster recovery plan must cover more than GPU replicas: model artifacts and registry metadata, prompts, embeddings and vector indexes, PostgreSQL/Redis/Kafka/MongoDB or Snowflake dependencies, tool execution state, quota reservations, credentials, encryption keys, policy decisions, network endpoints and observability.

## 2. RPO/RTO inventory
| Component | State | Typical recovery risk | Validation |
|---|---|---|---|
| model weights/tokenizer | immutable artifact | missing version or signature | digest and provenance |
| model registry/prompt config | versioned metadata | lost approved release | signed manifest and Git history |
| vector index | rebuildable but mutable | stale retrieval or ACL mismatch | freshness, recall, tenant isolation |
| PostgreSQL/MongoDB | durable state | backup lag, WAL/oplog gaps | point-in-time restore, integrity |
| Redis | cache or durable state | cache cold start, lost sessions | warmup and state classification |
| Kafka | event log | offset and replay duplication | checkpoint/offset audit |
| agent workflow | side effects and checkpoints | duplicate tool execution | idempotency and fencing |
| usage/budget ledger | financial state | double charging or overspend | reconciliation |
| identity/secrets | trust and encryption | unavailable KMS/credentials | least privilege, key access |

RPO is the **maximum acceptable data loss** measured as time; RTO is the **maximum acceptable time to restore service**. Both must be explicitly approved per dependency and measured in drills. A zero RPO claim requires synchronous durability semantics and validation, not simply multi-region replicas.

## 3. Backup strategy
- Classify authoritative data versus rebuildable caches and derived indexes.
- Use encrypted, immutable/versioned backups with separation of duties and restricted delete privileges.
- Define retention, legal hold, residency, backup frequency, restore keys and independent recovery credentials.
- Record recovery metadata: source revision, snapshot timestamp, log sequence/offset, encryption key ID, schema version, model and tokenizer digests.
- Test restores regularly; successful backup jobs do not prove recoverability.
- Protect credentials and backup inventory from the same failure or compromise affecting production.
- Plan consistency across dependent services; individual snapshots may not form a globally consistent recovery point.

## 4. Regional failover decision gates
1. Declare incident and freeze incompatible releases and schema changes.
2. Confirm primary failure and incident-command authority.
3. Check secondary region GPU quotas, warmed model capacity, artifact integrity and required dependencies.
4. Verify DR data freshness against each approved RPO and classify known gaps.
5. Fence primary writers and side-effecting tool workers to avoid split-brain and duplicate execution.
6. Verify identity, KMS, secrets, residency and tenant authorization in the DR region.
7. Restore durable stores in dependency order, then rebuild or verify vector indexes and caches.
8. Reconcile Kafka offsets, tool idempotency keys and token/budget reservations.
9. Run synthetic inference, retrieval ACL, safety, quality and streaming completion checks.
10. Shift approved canary traffic; measure errors, TTFT, throughput, spend and total RTO.
11. Expand traffic only after health and security gates pass; preserve rollback and failback plan.
12. Document customer impact, actual RPO/RTO, evidence, RCA/COA and residual data gaps.

## 5. Kubernetes and cloud preflight (read-only)
```bash
kubectl config current-context
kubectl get nodes -L topology.kubernetes.io/zone
kubectl get deploy,sts,pods,pdb -A
kubectl get pvc,pv -A
kubectl get events -A --sort-by=.lastTimestamp
```
For managed PostgreSQL, MongoDB, Kafka, vector stores and object storage, use the provider's approved restore/replication inspection commands. Do not initiate production promotion, DNS cutover, destructive restore or credential rotation from an educational lab.

## 6. DR runbooks

### RB-093-A — Regional inference outage
Protect primary if reachable, assess secondary capacity and latency, activate incident command, verify model digests and policies, canary the DR route, then gradually shift traffic under approval.

### RB-093-B — Database point-in-time recovery
Identify consistent restore timestamp, required WAL/oplog/archive chain, encryption keys and schema revision; restore to isolated environment; validate application data, ACLs and checkpoint offsets before promotion.

### RB-093-C — RAG index restore/rebuild
Restore authoritative documents and metadata first; rebuild with compatible embedding model; validate freshness, recall and tenant ACL filtering before enabling retrieval.

### RB-093-D — Agent side-effect fencing
Disable or fence primary workers, ensure idempotency keys and leases prevent dual writers, reconcile incomplete tool calls and confirm no duplicate external actions.

### RB-093-E — Backup compromise
Quarantine suspected backups, verify immutability and signer, rotate compromised access under security command, restore only from trusted clean recovery points and preserve forensic evidence.

### RB-093-F — Failback
Validate recovered primary data against authoritative DR state, plan reverse replication and fencing, confirm model/config parity, canary primary traffic, then restore normal routing without concurrent writers.

## 7. Troubleshooting matrix
| Symptom | Diagnosis | Guardrail |
|---|---|---|
| snapshot exists but restore fails | key access, corrupt chain, version mismatch | isolated restore validation |
| vector results stale | index checkpoint behind source | no silent stale retrieval |
| duplicate tool actions | missing idempotency/fencing | block dual writers |
| inference 503 in DR | no GPU quota, model cold start | measured capacity |
| cross-region 403 | IAM/KMS/residency mismatch | no broad permission bypass |
| overspend after failover | duplicate usage reservations/retries | reconcile budget ledger |
| split-brain | stale DNS/leases and active primary | fencing before promotion |
| missed RTO | cold restore or dependency ordering | record actual RTO, revise plan |

## 8. Staging game day
Use synthetic tenant records and approved test accounts. Simulate unavailable primary gateway, lagging database replica, stale vector index, inaccessible KMS, insufficient DR GPUs, orphaned agent action and a compromised backup manifest. Record exact RPO/RTO and audit evidence. Do not destroy a real production region or promote live databases without a separate approved change.

## 9. Observability and governance
Monitor backup age, restore success, replica lag, object version availability, model artifact parity, index freshness, key health, secondary capacity, RTO timers, RPO deltas, cross-region costs, dual-writer detection and customer SLOs. Assign owners, escalation thresholds, drill cadence and exception approval.

## 10. Acceptance rubric — 100 points
| Area | Points |
|---|---:|
| Dependency and backup inventory | 20 |
| RPO/RTO, consistency and restore testing | 20 |
| Split-brain, security and residency controls | 20 |
| Regional failover/failback and inference SLO | 20 |
| Automated tests, evidence and RCA/COA | 20 |
| **Total** | **100** |

Educational pass >=90/100, zero critical data-loss, dual-writer, tenant-isolation or residency violations. Offline simulation is not a production DR certification.

## 11. Companion lab
[Chapter 093 — Executable Backup, DR, RTO/RPO and Region Failover Lab](../labs/093-backup-dr-rto-rpo-region-failover-check.md) provides synthetic dependency recovery assessment, freshness and fencing gates, and pytest cases.

## References
- [Google SRE Workbook — Disaster Recovery](https://sre.google/workbook/disaster-recovery/)
- [AWS Well-Architected — Reliability Pillar](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/welcome.html)
- [Kubernetes Disaster Recovery](https://kubernetes.io/docs/concepts/cluster-administration/)
- [NIST Contingency Planning Guide SP 800-34](https://csrc.nist.gov/pubs/sp/800/34/r1/upd1/final)

**Next:** Chapter 094 — Chaos Engineering and Recovery Game Days.
