# Chapter 14.7 — Hands-on Lab: Post-Incident Validation and Reliability Acceptance

**Part 14 capstone | Approved staging/tabletop | Live execution pending**

## Goal

Execute a structured post-incident recovery assessment across customer inference, TrueFoundry reconciliation, Kubernetes, NVIDIA GPUs, vLLM, network, security, observability, FinOps and RCA/COA. **Publishing this lab does not certify execution.**

**Safety:** [SAFE-READ] authorized inspection; [LAB-MUTATE] disposable staging and synthetic traffic with spend cap; [CHANGE-CONTROL] production rollback, node operations, traffic, identities and alerts. No PHI, real prompts, credentials or private outputs in evidence.

## Lab 0 — Establish evidence and scope

```bash
kubectl config current-context
kubectl get nodes -L topology.kubernetes.io/zone -o wide
kubectl get deployments,pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -50
kubectl get svc,endpointslices -A
```

Record incident ID, UTC timeline, model/image/runtime revision, TrueFoundry agent, GPU SKU/zone, customer-path SLO, synthetic workload, stability window, rollback and reviewers.

## Lab 1 — Customer-path recovery

Validate authenticated synthetic requests and complete SSE streams. Capture eligible success, HTTP errors, TTFT, TPOT, queue, output token distribution, gateway status and agreed observation window. Reject HTTP-200-only recovery.

## Lab 2 — Control plane and Kubernetes

Compare TrueFoundry desired/observed state, agent health, harmless staging reconciliation, Kubernetes Pod/Deployment, EndpointSlice, events, quota and autoscaler. Record any drift or unresolved restart loop.

## Lab 3 — GPU and vLLM

Review GPU allocatable/requested slots, DCGM freshness, driver/device plugin, XID/ECC/thermal, N-1 headroom, vLLM KV cache, CUDA/host OOM, queue, TTFT/TPOT and model warmup. Calculate GPU-hours and cost per successfully served 1,000 output tokens.

## Lab 4 — Security, telemetry and closure

Verify approved RBAC, secrets references, TLS and network policy restored; confirm Prometheus, logs, DCGM, vLLM metrics, synthetic probes, alert routes and silences. Draft customer-safe recovery update and evidence-backed RCA/COA actions.

## Lab 5 — Six integrated game days

1. Bad model revision and SSE rollback.
2. Agent disconnected but existing inference healthy.
3. GPU XID and N-1 shortfall (tabletop).
4. KV cache/TTFT/TPOT regression (bounded staging).
5. TLS/ingress outage with paging and communications.
6. Compound GPU + telemetry + handoff and acceptance board.

Record executed/tabletop, UTC timeline, expected/actual, SLO, mitigation, rollback, evidence, residual risk and reviewer.

## Forty-eight failure scenarios

01 probe missing; 02 SSE truncated; 03 5xx; 04 429; 05 TTFT; 06 TPOT; 07 agent; 08 reconcile; 09 UI stale; 10 drift; 11 credential; 12 staging reconcile; 13 wrong model; 14 CrashLoop; 15 EndpointSlice; 16 Pending; 17 quota; 18 PDB; 19 allocatable; 20 plugin; 21 XID; 22 ECC; 23 thermal; 24 zone; 25 queue; 26 KV; 27 CUDA OOM; 28 host OOM; 29 cold start; 30 throughput; 31 TLS; 32 DNS; 33 proxy; 34 buffering; 35 retries; 36 tenant routing; 37 metrics; 38 silence; 39 paging; 40 DCGM; 41 RBAC; 42 secret; 43 rollback; 44 stability; 45 RCA; 46 COA; 47 cost; 48 closure.

## Fourteen acceptance gates — 100 points

| Gate | Evidence | Points | Status |
|---|---|---:|---|
| G1 | Scope/change/evidence | 5 | Pending |
| G2 | Customer availability/output | 10 | Pending |
| G3 | SSE/gateway | 10 | Pending |
| G4 | TTFT/TPOT/queue/error budget | 10 | Pending |
| G5 | Agent/reconciliation | 8 | Pending |
| G6 | Kubernetes | 7 | Pending |
| G7 | GPU/DCGM | 8 | Pending |
| G8 | N-1/quota/topology | 7 | Pending |
| G9 | vLLM/KV/OOM | 8 | Pending |
| G10 | Security | 7 | Pending |
| G11 | Telemetry/paging | 7 | Pending |
| G12 | Rollback/drift | 5 | Pending |
| G13 | Cost/comms/RCA owners | 4 | Pending |
| G14 | Stability/reviewer | 4 | Pending |
| | **Total** | **100** | **Not assessed** |

Recommended pass: at least 90 points **and every mandatory gate passed**. Security, customer streaming, paging and rollback gaps block acceptance regardless of score. Organization-approved acceptance policy supersedes this training rubric.

## Evidence template

```text
Incident / exercise / approval / UTC:
Customer impact and approved SLO window:
TrueFoundry desired/observed and agent:
Model/image/vLLM/CUDA/GPU revisions:
Synthetic availability, TTFT, TPOT, SSE:
Kubernetes/GPU/DCGM/queue/KV:
N-1, cloud quota, GPU-hours, cost/token:
Security, telemetry, paging, rollback:
Mitigation/recovery/stability timestamps:
RCA confidence, COA owner/due date:
Gate scores / blockers / residual risk:
Executed/tabletop / reviewer / decision:
```

## Cleanup and final decision

Restore disposable staging resources, alerts and traffic; verify no leaked privileges, no orphaned model routes, healthy synthetic streams and spend cap. Archive sanitized evidence. Record **PASS**, **CONDITIONAL WITH APPROVED RISK**, or **FAIL** only after actual review. Leave all gates pending until performed.

**Next:** Part 15 — End-to-End Production Project, Chapter 15.1 — Architecture and Requirements.
