# Chapter 089 — GPU Platform DR and Continuity

**Part 11 — Capacity, FinOps and Reliability | AI Infrastructure Engineering & SRE**

**Status:** Documentation published. Isolated recovery drills, live checkpoint restores, cross-region failover tests, security review and independent acceptance are **PENDING**.

## 1. Mission

Build a disaster recovery (DR) and business continuity plan for GPU-based AI platforms that protects model serving, distributed training progress, model artifacts, checkpoints, secrets, cluster configuration and tenant boundaries across major infrastructure failures. Measure actual RPO/RTO and recovered service quality, rather than treating replicated files or spare GPUs as proof of recoverability.

**Prerequisites:** Chapters 083–088, Kubernetes backup/restore, GPU Operator, distributed training checkpointing, inference SLOs, cloud capacity reservations, storage replication, IAM/KMS and incident command.

### Learning objectives

1. Define recovery tiers and explicit RTO/RPO for inference and training.
2. Map failure domains and cross-region dependencies.
3. Restore model weights, checkpoints, optimizer state and dataset provenance with integrity checks.
4. Verify GPU SKU, CUDA/driver, network and quota compatibility in recovery regions.
5. Design safe traffic failover and failback with security and cost controls.
6. Run staged DR exercises, collect evidence and reject unverified recovery claims.

## 2. Recovery is an end-to-end system

\`\`\`text
Primary region
  ├─ GPU serving + model router
  ├─ training queues + distributed jobs
  ├─ checkpoints + model registry + datasets
  ├─ Kubernetes/GitOps + secrets + IAM
  └─ observability / incident tooling
             │
       verified replication
             ↓
Recovery region
  ├─ compatible GPU capacity and node pools
  ├─ recoverable model/artifact/checkpoint store
  ├─ GitOps manifests + verified secrets access
  ├─ routing, identity, DNS, networking
  └─ independent monitoring and runbooks
\`\`\`

Replication is necessary but not sufficient. An unavailable GPU SKU, missing KMS key, incompatible driver, zonal-only volume, broken DNS or untested training resume can invalidate the recovery plan.

## 3. Define workload-specific DR contracts

| Workload | Availability expectation | Recovery data | Critical metric |
|---|---|---|---|
| P0 online inference | restore SLO-compliant service within agreed RTO | model version, configuration, routing, features | end-to-end RTO, p99, errors |
| Batch inference | restart or replay safely | input offsets, idempotency keys, outputs | completion deadline, duplication |
| Distributed training | resume from last complete checkpoint | weights, optimizer, RNG, scheduler, dataset state | lost steps (RPO), resume RTO |
| Model registry | integrity and authorization | artifact hashes, lineage, signatures | verified artifact recovery |
| Control plane | restore orchestration | IaC, GitOps, secrets, policies | cluster/control recovery |
| Observability | independently detect/confirm recovery | metrics/logging/alerts | evidence continuity |

**RPO** measures maximum acceptable lost progress/data; **RTO** is time to restore defined service capability. For training, express RPO in time **and/or optimizer steps** and account for partial checkpoints.

## 4. Failure scenarios and DR tiers

| Scenario | Typical recovery action |
|---|---|
| GPU node failure | restart on compatible node |
| AZ outage | zone evacuation, preserved regional state |
| Regional outage | independent recovery region and traffic failover |
| Object store/checkpoint corruption | restore known-good immutable version |
| Kubernetes control-plane loss | GitOps/IaC recreation and verified secrets |
| IAM/KMS outage | approved recovery identity/key strategy |
| Model registry unavailable | verified artifact mirror |
| Data pipeline outage | bounded replay and provenance validation |
| Ransomware/malicious deletion | isolated immutable backups and incident response |
| Quota/capacity exhaustion | alternate compatible reserved GPU pool |

Do not conflate AZ resilience with regional DR. Cross-region recovery requires its own quota, physical capacity, network path, IAM, data residency and cost approvals.

## 5. Data protection and checkpoint integrity

A distributed checkpoint must be **atomically complete** from the perspective of the recovery procedure. Record:
- model architecture, weight/shard files, tensor precision and checksums;
- optimizer, learning-rate scheduler, gradient scaler and RNG state;
- global step, data sampler/offsets, dataset version and tokenization configuration;
- world-size and sharding layout; restart compatibility requirements;
- checkpoint completion marker, timestamp and immutable manifest;
- encryption, KMS access and replication status.

**Never** restore from a partially written checkpoint simply because a directory exists. Use a manifest/commit marker and verify hashes before resuming. Frameworks differ in their support for resharding and elastic world-size changes.

### Example manifest (illustrative)

\`\`\`yaml
checkpoint_id: train-run-042-step-120000
completed_utc: "2026-10-20T00:00:00Z"
global_step: 120000
dataset_version: dataset-v7
model_revision: model-sha256-REPLACE
world_size: 16
files:
  - path: weights/shard-0001.bin
    sha256: REPLACE_WITH_ACTUAL_SHA256
  - path: optimizer/shard-0001.bin
    sha256: REPLACE_WITH_ACTUAL_SHA256
completion_marker: COMMITTED
replicated_regions: [primary-region, recovery-region]
\`\`\`

Do not copy example values into production evidence. Real manifests must enumerate **all** required shards and state.

## 6. Model-serving continuity

For P0 inference, DR success requires:
1. compatible model/serving image and accelerator HBM in recovery region;
2. independently accessible model artifacts and keys;
3. healthy ingress/DNS/load-balancer routing and client authentication;
4. sufficient prewarmed or provisionable GPU capacity before RTO expires;
5. load test at **failover traffic** with p99, TTFT/TPOT and error gates;
6. model output quality, safety filters and tenant isolation parity.

DNS TTL alone does not determine traffic failover time. Account for health checks, caches, connection draining, client retry behavior and warmup.

## 7. Recovery-region GPU capacity

Track:
- compatible GPU SKU, HBM, interconnect and driver/CUDA support;
- regional GPU quota **and** physical supply/reservation scope;
- minimum serving replicas, node/AZ failure headroom and training priority;
- node provisioning, image pull, model load and warmup latency;
- CPU/RAM, storage bandwidth, networking, service quotas and KMS;
- data residency restrictions and cross-region egress costs.

A billing discount is not a GPU capacity reservation. Validate actual provider guarantees and failover-region supply.

## 8. Infrastructure as code and recovery sequencing

Suggested dependency-aware order:

\`\`\`text
Declare incident and recovery authority
  → establish identity/network/DNS/KMS access
  → verify storage/backup and artifact integrity
  → create/reconcile cluster + GPU nodes/operators
  → restore platform policies, secrets and monitoring
  → deploy serving models and validate warmup
  → test routing and canary traffic at SLO
  → promote traffic with rollback control
  → restore training from committed checkpoints
  → verify backlog, cost and business continuity
\`\`\`

Use GitOps and IaC with pinned versions and policy checks. Prevent accidental dual-active writers or duplicate batch outputs with fencing, idempotency keys and authoritative state transitions.

## 9. Recovery-time budget

\[
T_{\mathrm{recovery}}=T_{\mathrm{detect}}+T_{\mathrm{decide}}+T_{\mathrm{provision}}+T_{\mathrm{restore}}+T_{\mathrm{warm}}+T_{\mathrm{validate}}+T_{\mathrm{route}}
\]

This is a sequential planning upper-bound; measured stages may overlap. Report the **observed end-to-end** elapsed time from incident trigger to restored SLO, not only the storage restore duration.

### Synthetic example

| Stage | Minutes |
|---|---:|
| Detection and decision | 8 |
| GPU provisioning | 22 |
| Restore artifacts and platform | 15 |
| Warm models and validate | 12 |
| Traffic routing and verification | 8 |
| **Total** | **65** |

For a 60-minute RTO, the plan **fails** even if checkpoint integrity is perfect. Reduce critical-path duration, prewarm capacity or renegotiate the target; do not report a pass.

## 10. Security and compliance

- Use least-privilege break-glass roles with audited approvals and expiry.
- Replicate only data permitted by residency and contractual rules.
- Encrypt backups in transit/at rest; test independent recovery-key access.
- Separate immutable backup administration from production write privileges.
- Preserve artifact signatures, SBOM/provenance and admission policies.
- Audit restore actor, artifact hash, destination and traffic promotion.
- Redact sensitive model/customer identifiers from incident evidence.

## 11. DR test strategy

| Test level | Scope | What it proves |
|---|---|---|
| Static review | inventories, IaC, runbooks | documented dependencies |
| Offline tabletop | regional outage scenario | decision and escalation logic |
| Artifact restore | isolated storage and hash check | data integrity |
| Training restart | staging checkpoint resume | checkpoint compatibility and progress |
| Serving failover | synthetic traffic to recovery region | p99/errors and routing |
| Regional game day | authorized isolated full stack | end-to-end RTO/RPO and controls |

A successful tabletop is not equivalent to a completed live DR drill. Schedule recurring tests and track failed gates to closure.

## 12. Six production runbooks

**A — Regional inference failover:** declare incident, confirm recovery-region GPU capacity, validate artifacts/keys, warm replicas, canary SLO, shift traffic, monitor and retain rollback.

**B — Distributed training recovery:** fence old writers, select last committed checkpoint, verify all hashes/state, confirm world-size compatibility, resume and validate step/quality continuity.

**C — Corrupt checkpoint recovery:** quarantine invalid snapshot, select immutable known-good version, measure additional RPO, restore with checksum and open data-integrity incident.

**D — GPU capacity unavailable:** distinguish quota from stock, inspect reservations and compatible SKUs, protect P0 inference, escalate procurement/provider and activate approved fallback.

**E — IAM/KMS/registry outage:** use approved break-glass path, verify audit trail and key access, restore artifact mirror, revoke temporary access after recovery.

**F — Failback to primary:** confirm primary stability, fence dual writers, synchronize authoritative state, canary routing, verify cost and remove orphaned resources only after approval.

## 13. Sixteen hands-on exercises

1. Build primary/recovery-region dependency map.
2. Classify failure domains and recovery tiers.
3. Define inference and training RTO/RPO.
4. Inventory checkpoint and model artifacts.
5. Create committed checkpoint manifest with hashes.
6. Verify synthetic backup replication and retention.
7. Test immutable checkpoint restore in isolation.
8. Model distributed training resume and lost steps.
9. Audit recovery-region GPU SKU/quota/reservations.
10. Calculate end-to-end RTO critical path.
11. Design multi-region serving canary and rollback.
12. Validate model/secret/IAM and residency parity.
13. Simulate regional outage and incident command.
14. Simulate corruption and last-known-good recovery.
15. Plan failback, fencing and cost cleanup.
16. Score independent DR acceptance evidence.

## 14. Twelve troubleshooting scenarios

| Symptom | Diagnosis |
|---|---|
| Backup exists, no completion marker | incomplete checkpoint |
| Restore hash mismatch | corruption/tampering |
| Model weights restored, optimizer missing | invalid training resume |
| Different world-size fails | sharding compatibility |
| GPU quota approved, nodes unavailable | physical supply |
| Recovery GPU HBM too small | SKU mismatch |
| Serving p99 spikes after failover | cold start/capacity |
| Cross-region model artifact denied | IAM/KMS/residency |
| Old and new batch workers both active | missing fencing |
| Recovery cluster healthy, traffic fails | DNS/ingress/auth |
| DR run exceeds RTO | critical-path budget |
| Primary restored, failback duplicates jobs | idempotency/authority |

## 15. Acceptance gate (100 points)

| Gate | Points |
|---|---:|
| Failure domains, dependency map and DR contract | 15 |
| Backup/checkpoint integrity and training resume | 25 |
| GPU capacity and platform recovery feasibility | 20 |
| Serving failover SLO and end-to-end RTO/RPO | 20 |
| Security, fencing, rollback and failback | 10 |
| Evidence, runbooks and independent review | 10 |
| **Total** | **100** |

**Pass:** ≥85/100 and no critical checkpoint corruption, unapproved data movement, tenant security breach, unsupported recovery capacity, missed protected RTO/RPO or absent rollback. Publication alone is not recovery acceptance.

## 16. References

- [Kubernetes Disaster Recovery Considerations](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/)
- [Kubernetes Volumes](https://kubernetes.io/docs/concepts/storage/volumes/)
- [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/)
- [Kueue Documentation](https://kueue.sigs.k8s.io/)
- [AWS Disaster Recovery](https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/disaster-recovery-workloads-on-aws.html)
- [Google SRE — Managing Critical State](https://sre.google/sre-book/managing-critical-state/)

**Next:** Chapter 090 — Capacity and Reliability Acceptance Capstone.
