# Chapter 004 — AI Infrastructure Service Ownership and Support Tiers

**Part 01 — Foundations and Architecture · Chapter 004 of 100**  
**Publication:** Complete. **Offline lab execution, staging exercises, operational approval and independent review:** Pending.

## Learning outcomes
Create a supportable AI infrastructure service catalog, define clear ownership across GPU hardware, Kubernetes, network, storage, training and inference, implement service tiers and SLO-based escalation, route incidents using evidence, establish incident command and change approvals, and validate ownership contracts with an executable Python lab.

## 1. Why ownership is an SRE control
AI incidents cross multiple teams: a GPU worker can disappear because of a host Xid error, driver incompatibility, Kubernetes device plugin, quota exhaustion, RDMA instability, model loading, checkpoint storage or IAM. A generic “GPU problem” ticket creates delayed escalation and unclear rollback authority. A production-ready AI platform therefore needs explicit **service ownership**, **operational boundaries**, **dependency mapping**, **escalation paths**, **security responsibility** and **measured support commitments**.

### Enterprise service map
```text
AI consumer / model owner
        |
  API / training job
        |
ML platform owner ----- AI inference / training SRE
        |                          |
   Kubernetes platform SRE ---- GPU host / cloud provider
        |                          |
   network / storage / DBRE ---- security / IAM / FinOps
        |
    incident commander + communications lead
```
The incident commander coordinates response but does not override change controls, privacy policy or data ownership. Service owners maintain the service catalog, SLO and on-call coverage. Dependency teams own their component recovery. Security can stop unsafe activity.

## 2. Production service catalog
| Service | Primary owner | Secondary escalation | Evidence / monitoring |
|---|---|---|---|
| GPU hardware and firmware | Infrastructure SRE / cloud provider | hardware vendor | Xid/ECC, node events, GPU inventory |
| Linux GPU drivers and CUDA | GPU platform SRE | vendor / host team | driver/kernel matrix, nvidia-smi |
| GPU Operator / device plugin | Kubernetes platform SRE | GPU platform SRE | DaemonSet, allocatable GPU, logs |
| EKS/AKS GPU capacity and quotas | Cloud platform SRE | cloud provider support | quota, allocation, node groups |
| RDMA/EFA/InfiniBand/NCCL fabric | Network/HPC SRE | GPU platform SRE | link errors, latency, rank traces |
| Dataset/checkpoint storage | Storage SRE / DBRE | cloud storage support | throughput, RPO, restore evidence |
| Slurm/Ray/Kubernetes job scheduling | ML platform SRE | Kubernetes/HPC SRE | queue age, job state, placement |
| Inference gateway and serving | LLMOps / inference SRE | application/model owner | TTFT/TPOT, stream completion |
| Model registry and artifact signatures | ML platform / security | storage/platform | digest, provenance, IAM |
| Identity, secrets and tenant boundaries | Security/IAM | platform SRE | access decisions, audit |
| Observability and incident paging | Observability SRE | incident commander | telemetry health, alert delivery |
| GPU cost and budget | Platform FinOps | service owner | GPU-hours, allocation, idle spend |

This is an illustrative model. Assign actual named teams, paging rotations and backup contacts before production rollout.

## 3. Support tiers and service criticality
Support **tiers** describe response and escalation obligations; **service criticality** describes impact. Do not conflate them.

| Tier | Scope | Illustrative coverage | Escalation |
|---|---|---|---|
| Tier 0 | documented self-service, diagnostics, known issues | automated 24/7 documentation | page Tier 1 for uncontained impact |
| Tier 1 | initial triage, incident intake, basic containment | on-call according to service contract | GPU/platform owner |
| Tier 2 | Kubernetes/GPU/network/storage specialist | production on-call | Tier 3/vendor |
| Tier 3 | engineering/vendor, complex recovery | escalation agreement | executive/major incident coordination |

**Criticality examples:** P0/SEV1 = broad production outage, data exposure, cross-tenant violation or unrecoverable corruption; P1/SEV2 = major service degradation or missed critical workload deadline; P2/SEV3 = localized degradation with workaround; P3/SEV4 = low-impact defect or request. Security and integrity violations may warrant immediate high severity even with few affected requests.

**Illustrative targets, not promises:** SEV1 acknowledge within 5 minutes, assign incident commander within 10 minutes, stakeholder update every 15 minutes; SEV2 acknowledge within 15 minutes, update every 30 minutes. Approve targets based on staffed coverage, monitoring capability, customer contracts and actual operational evidence.

## 4. RACI and decision rights
| Activity | Incident commander | Platform SRE | ML/Model owner | Security | Storage/Network |
|---|---|---|---|---|---|
| Declare severity and coordinate | A/R | C | C | C | C |
| Diagnose GPU/driver failure | I | A/R | C | C | C |
| Verify model quality after rollback | I | C | A/R | C | I |
| Investigate tenant/security breach | C | R | C | A | C |
| Restore checkpoint consistency | I | C | A | C | R |
| Approve production change | C | R | C | C | C |

The change-approval authority must be explicitly identified in each organization's governance process; this table does not itself grant permission. R = responsible, A = accountable, C = consulted, I = informed. Avoid multiple accountable owners for the same decision.

## 5. SLOs, SLIs and ownership contracts
A service catalog entry should specify: service ID, environment, business criticality, primary and backup owner, support window, SLI/SLO, alert thresholds, paging route, dependencies, error-budget policy, RPO/RTO, runbook, rollback owner, security classification, quota and spend ceiling, audit retention and last recovery test.

**Example service-level indicators:**
- GPU node readiness: schedulable healthy GPU capacity / intended GPU capacity.
- GPU allocation success: accepted requests scheduled within target / accepted requests.
- Training reliability: completed eligible jobs / eligible jobs; include checkpoint recovery time.
- Inference reliability: correctly completed eligible streams / eligible streams, p95 TTFT and TPOT.
- Checkpoint recoverability: successfully restored checkpoints / sampled restore attempts.
- Alerting: actionable critical incidents paged and acknowledged within contract.

An SLO is not an alerting rule by itself. Multi-window burn-rate alerts should be chosen according to error-budget policy and available telemetry.

## 6. Escalation decision tree
```text
Incident or alert
    |
security/privacy/tenant breach? --- yes --> security incident, stop unsafe work
    |
    no
    |
service impact and severity classification
    |
collect UTC timeline + workload/model/cluster IDs
    |
GPU node/driver? --> GPU platform SRE
scheduler/quota? --> Kubernetes/cloud SRE
NCCL/network? --> HPC/network SRE
checkpoint/data? --> storage/DBRE
model/TTFT/TPOT? --> inference/ML SRE
    |
incident commander tracks owner + acknowledgement
    |
containment --> recovery --> quality/integrity verification
    |
RCA/COA + evidence + follow-up
```
If multiple domains fail, maintain a single incident command channel with multiple workstreams. Escalate unknown-owner incidents to the designated platform incident commander, not to a randomly selected team.

## 7. [SAFE-READ] Operational discovery lab
Only run in an authorized environment:
```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get nodes -L topology.kubernetes.io/zone
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp
kubectl get daemonsets -A
kubectl get resourcequota -A
nvidia-smi -L
nvidia-smi --query-gpu=index,name,driver_version,temperature.gpu --format=csv
```
The commands do not modify infrastructure, but may expose sensitive metadata. `nvidia-smi` is unavailable without compatible hardware/driver. Capture evidence privately and avoid placing tokens or customer data in tickets.

## 8. Production incident simulations
**Scenario A — GPU unavailable:** A node is Ready but advertises zero GPU resources. Tier 1 gathers Kubernetes events and node resource inventory; GPU platform SRE checks driver/device plugin; vendor escalated for Xid/ECC; rollback owner authorizes approved node remediation.

**Scenario B — Training NCCL hang:** ML platform collects rank logs and job state; network/HPC checks RDMA and fabric; storage validates checkpoint; incident commander decides bounded recovery only after optimizer/data cursor consistency.

**Scenario C — Inference TTFT spike:** Inference SRE checks queue, prefill, decode, KV pressure and gateway throttling; cloud/GPU team validates N-1 capacity; security reviews any unexpected tenant traffic.

**Scenario D — RAG stale data:** Data/DBRE checks source checkpoint and index freshness, ML platform gates stale retrieval, security validates ACLs, application owner verifies answer quality before reopen.

**Scenario E — Regional GPU capacity shortage:** Cloud SRE checks quotas and reservations, platform SRE enforces safe admission, FinOps approves budget and incident commander coordinates customer communications.

## 9. Troubleshooting matrix
| Failure | Likely ownership | Immediate safe action |
|---|---|---|
| no allocatable GPU | GPU/Kubernetes platform | inspect driver and device-plugin health |
| CUDA init failure | GPU platform + ML runtime | verify compatibility matrix |
| NCCL collective timeout | HPC network + ML platform | correlate ranks and fabric counters |
| training checkpoint corrupted | storage/DBRE + ML owner | stop replay and verify restore |
| TTFT high | inference SRE + GPU platform | trace queue and GPU pressure |
| cross-tenant data exposure | security + incident command | stop affected path, preserve evidence |
| quota or GPU shortage | cloud platform + FinOps | enforce admission and escalate capacity |
| alert delivery failure | observability SRE | independent probe and paging fallback |

## 10. Runbooks
**RB-004-A — Incident intake:** establish severity, affected tenants, service ID, primary owner, fallback owner, UTC timeline, customer impact and first safe containment.

**RB-004-B — Escalation and handoff:** include current state, symptom, diagnostics, actions attempted, risks, next action, named owner and time of next update. Do not transfer an incident without acknowledgement.

**RB-004-C — GPU/platform failure:** correlate node/GPU health, driver, operator, scheduler, topology and quota; avoid unapproved production node restart.

**RB-004-D — Security override:** immediately pause unsafe action, preserve evidence, invoke security incident process and block change until authorized.

**RB-004-E — Post-incident closure:** verify recovery SLOs, checkpoint or stream correctness, customer communications, RCA/COA owner, deadline and follow-up retest.

## 11. Acceptance criteria and 100-point assessment
| Domain | Points |
|---|---:|
| Service catalog completeness | 20 |
| Tier definitions and severity contracts | 20 |
| RACI, escalation and security controls | 20 |
| SLO, runbook and recovery ownership | 20 |
| Executable lab and incident simulations | 20 |
| **Total** | **100** |

Pass >=90/100 with no unowned critical service, security bypass or ambiguous recovery approval. The offline lab uses synthetic service metadata and cannot verify actual on-call staffing or paging.

## 12. Executable lab
[Chapter 004 — Ownership and Escalation Acceptance Lab](../labs/004-ai-infrastructure-service-ownership-check.md) contains a Python 3.11+ fail-closed service contract validator and pytest tests.

## References
- [Google SRE Workbook — Incident Response](https://sre.google/workbook/incident-response/)
- [Google SRE Book — Effective Troubleshooting](https://sre.google/sre-book/effective-troubleshooting/)
- [Kubernetes Device Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)
- [NVIDIA Data Center Documentation](https://docs.nvidia.com/datacenter/)

**Next:** Chapter 005 — Linux, Kubernetes and Cloud Dependency Mapping.
