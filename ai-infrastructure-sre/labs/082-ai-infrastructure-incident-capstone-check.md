# Lab 082 — AI Infrastructure Incident Capstone Acceptance Gate

**Chapter:** [082 — AI Infrastructure Incident Capstone](../part-10-incident-response-and-troubleshooting/082-ai-infrastructure-incident-capstone.md)

**Status:** NOT EXECUTED. Documentation published; offline tabletop, staging game day, independent review and acceptance are pending.

## 1. Safety and scope

Use synthetic incident fixtures offline and dedicated, approved staging GPUs for bounded runtime checks. No production GPU reset, driver reload, forced rank kill, network fault injection, storage corruption, checkpoint deletion or node drain. Require IC/ML owner/network/storage approvals for changes and protect last verified checkpoint.

## 2. Twenty-exercise execution checklist

| ID | Exercise | Required evidence | Status |
|---|---|---|---|
| E01 | Cluster/GPU inventory | node/pod/GPU UUID matrix | Pending |
| E02 | SLO, severity and roles | incident charter | Pending |
| E03 | Kubernetes synthetic failure | pod/node event timeline | Pending |
| E04 | Rank timeline | first error by rank | Pending |
| E05 | CUDA/driver classification | compatibility evidence | Pending |
| E06 | GPU vs host OOM | CUDA error vs OOMKilled | Pending |
| E07 | NCCL collective analysis | rank sequence and watchdog | Pending |
| E08 | RDMA/RoCE counter analysis | deltas, errors, timing | Pending |
| E09 | Storage/checkpoint diagnosis | 503/retry/latency | Pending |
| E10 | Inference SLO impact | separate/shared hypothesis | Pending |
| E11 | Competing hypotheses | confidence and missing evidence | Pending |
| E12 | Topology map | rank/GPU/NIC/storage | Pending |
| E13 | Checkpoint integrity | manifest and staged restore | Pending |
| E14 | CUDA smoke | dedicated staging result | Pending |
| E15 | NCCL smoke | approved two-GPU result | Pending |
| E16 | Cordon/drain tabletop | owner, abort and rollback | Pending |
| E17 | Eight runbooks | peer-reviewed runbook links | Pending |
| E18 | RCA | causal timeline, unknowns | Pending |
| E19 | Six COAs | owner, due date, test | Pending |
| E20 | Acceptance review | score, reviewer and risks | Pending |

## 3. Synthetic incident fixture

\`\`\`text
14:00:00Z node image rollout
14:01:10Z GPU plugin registration warning node-3
14:01:35Z rank-5 CUDA initialization error
14:02:10Z NCCL progress stops on peers
14:03:20Z RoCE retry counters rise
14:04:00Z checkpoint PUT 503 with retries
14:05:30Z training progress alert
14:06:15Z inference p99 increases
14:08:00Z incident declared
14:12:00Z last complete checkpoint identified
\`\`\`

**Questions:** What is the earliest observed fault? What evidence supports or contradicts driver incompatibility, fabric congestion, storage throttling and independent inference regression? What should be preserved before any restart? What is the lowest-risk reversible mitigation? Which actions need owner approval?

## 4. Read-only diagnostic command set

\`\`\`bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -100
kubectl -n <namespace> get pods -o wide -l <job-label-selector>
kubectl -n <namespace> logs <rank-pod> --timestamps=true --since=2h
kubectl -n <namespace> describe pod <rank-pod>
\`\`\`

Approved GPU host:

\`\`\`bash
nvidia-smi -L
nvidia-smi --query-gpu=uuid,name,driver_version --format=csv
journalctl -k --since '-2 hours' | grep -Ei 'NVRM|Xid|AER|nvidia|rdma'
ip -s link
\`\`\`

Record unsupported tools and missing telemetry explicitly. Do not invent a successful smoke test when staging hardware is unavailable.

## 5. Staging game-day plan

**Preflight:** staging namespace, isolated GPUs, compatible PyTorch/CUDA/NCCL image, disposable storage, approved owners, checkpoint fixture, max duration and abort threshold.

**Test A:** bounded CUDA device enumeration and tensor allocation; capture exit status and GPU UUID.

**Test B:** two-GPU \`torchrun\` all-reduce smoke from Chapter 078; capture per-rank outputs and version.

**Test C:** synthetic checkpoint write with commit marker, checksum and readback; verify a restart/restore path.

**Test D:** simulate a failure through **fixture-based logs or application-level controlled exit** in staging; verify alert delivery and decision path. Do not inject Xid/ECC or live network faults.

**Abort:** unexpected impact on shared tenants, GPU disappearance, checkpoint corruption, missing owner, SLO regression beyond agreed limit, or test duration exceeded.

## 6. Acceptance evidence template

\`\`\`yaml
chapter: "082"
part: "10"
environment: "OFFLINE_OR_STAGING"
execution_date_utc: null
incident_commander: null
technical_reviewer: null
security_reviewer: null
training_slo: null
inference_slo: null
checkpoint_id: null
checkpoint_restore: "NOT_RUN"
cuda_smoke: "NOT_RUN"
nccl_smoke: "NOT_RUN"
rank_timeline: "NOT_COMPLETED"
fabric_analysis: "NOT_COMPLETED"
storage_analysis: "NOT_COMPLETED"
runbooks_reviewed: 0
coa_verified: 0
synthetic_cases_completed: 0
change_approval: "PENDING"
rollback_test: "NOT_RUN"
score: null
acceptance: "NOT_EVALUATED"
evidence_links: []
open_risks: []
\`\`\`

## 7. Scoring

| Gate | Points |
|---|---:|
| Incident command, safety, severity | 10 |
| Cross-layer evidence and timeline | 20 |
| Fault isolation and hypotheses | 15 |
| Eight runbooks and rollback | 20 |
| Checkpoint and SLO recovery | 15 |
| RCA/COA governance | 10 |
| Staging drills and independent review | 10 |
| **Total** | **100** |

**Pass:** ≥85 and zero critical privacy, data-integrity, unsafe-change or unapproved-production-impact failures. If staging drills or review are not run, record **PENDING** rather than claiming acceptance.

**Part 10:** 8/8 chapters documented, validation pending.

**Next:** Chapter 083 — GPU Demand Forecasting.
