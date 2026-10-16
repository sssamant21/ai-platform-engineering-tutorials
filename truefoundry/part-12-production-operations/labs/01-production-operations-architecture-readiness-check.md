# Chapter 12.1 — Hands-on Lab: Production Operations Architecture and Readiness

**Part 12 | Approved staging | Live execution pending**

## Goal and safeguards

Create a production-readiness dossier for a TrueFoundry inference deployment. Validate Kubernetes/GPU prerequisites, deployment and model identity, SLO ownership, release/rollback safety and incident contacts. Use only synthetic staging workloads. [SAFE-READ] inspect; [LAB-MUTATE] approved disposable resources; [CHANGE-CONTROL] production.

## Lab 0 — Preflight

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get deployments -A
kubectl get pdb,hpa -A
kubectl get events -A --sort-by=.lastTimestamp | tail -40
kubectl top nodes 2>/dev/null || true
```

Record cluster, namespace, TrueFoundry and vLLM versions, model/image digest, GPU node/driver, security boundary, owner and rollback contact. Do not export Secrets or sensitive inference payloads.

## Lab 1 — Readiness inventory

Build service map, RACI ownership, SLOs, alert routes, workload requests/limits, GPU capacity, topology, ingress/TLS, cloud identity and model-artifact dependencies. Identify critical missing evidence.

## Lab 2 — Capacity baseline

Using approved synthetic traffic, capture accepted RPS, input/output tokens/sec, TTFT, TPOT, error rate, queue, KV cache, GPU memory/utilization and cold-start time. Record sample size, model/GPU version and stop conditions. Mark unsupported runtime metrics as gaps.

## Lab 3 — Safe rollout tabletop

Design canary, smoke test, observation window, freeze trigger, streaming drain, immutable rollback and incident escalation. If permitted, demonstrate a disposable staging rollout and restore the prior version. Do not alter shared production routing.

## Lab 4 — Failure and recovery

Choose safe staging scenarios (bad readiness probe, missing model artifact, insufficient GPU request) or tabletop. Confirm detection, impact assessment, diagnosis, recovery, SLO verification and communications.

## Thirty failure scenarios

| # | Failure | Evidence to inspect |
|---|---|---|
| 01 | Missing service owner | RACI |
| 02 | No secondary on-call | Escalation |
| 03 | Unknown model digest | Supply chain |
| 04 | Unsupported vLLM/CUDA combination | Compatibility |
| 05 | GPU node NotReady | Node status |
| 06 | GPU allocatable missing | Device plugin |
| 07 | Pod Pending | Scheduler |
| 08 | Quota exhausted | ResourceQuota |
| 09 | Taint mismatch | Tolerations |
| 10 | Zone capacity imbalance | Topology |
| 11 | ImagePullBackOff | Registry/IAM |
| 12 | Model artifact access denied | Workload identity |
| 13 | Model loading timeout | Startup |
| 14 | Readiness probe too aggressive | Health |
| 15 | Liveness restart loop | Health |
| 16 | Container OOMKilled | Memory |
| 17 | CUDA OOM | GPU/KV |
| 18 | GPU runtime unavailable | Driver |
| 19 | Gateway TLS failure | Certificates |
| 20 | Gateway timeout | Streaming |
| 21 | SLO metric missing | Observability |
| 22 | Alert route broken | Incident |
| 23 | TTFT regression | Prefill/queue |
| 24 | Queue grows at peak | Capacity |
| 25 | Autoscaler too slow | Node provision |
| 26 | PDB blocks drain | Availability |
| 27 | Canary traffic misrouted | Release |
| 28 | Rollback revision unavailable | Recovery |
| 29 | GPU idle cost unbounded | FinOps |
| 30 | Security exception undocumented | Governance |

For each record expected/actual, UTC time, sanitized evidence, owner, mitigation, rollback and reviewer. Unsafe scenarios are tabletop.

## Twelve acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G1 | Scope, safety and owner approved | Pending |
| G2 | Architecture and dependencies documented | Pending |
| G3 | Immutable model/image revisions verified | Pending |
| G4 | Identity/network/security controls reviewed | Pending |
| G5 | Kubernetes and GPU placement verified | Pending |
| G6 | Health probes and drain policy reviewed | Pending |
| G7 | Load baseline and capacity headroom measured | Pending |
| G8 | SLO/alert/dashboard coverage validated | Pending |
| G9 | Release/canary/rollback rehearsed | Pending |
| G10 | Incident ownership/escalation tested | Pending |
| G11 | Cost, resilience and risk register reviewed | Pending |
| G12 | SRE/security/service-owner sign-off | Pending |

**Pass** requires executed evidence and reviewer approval. Pending or tabletop-only recovery cannot be marked as a successful live test.

**Next:** Chapter 12.2 — GPU Capacity Planning, Quotas and Production Scaling.
