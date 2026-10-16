# Chapter 13.7 — Enterprise Troubleshooting and Incident Recovery Capstone: Acceptance Lab

**Part 13 | Approved staging | Live execution pending**

## Objective

Run an integrated, evidence-based TrueFoundry GPU inference incident recovery campaign across control plane, Kubernetes, image/model startup, network, CUDA/vLLM, streaming and on-call operations. Publish an auditable GO / CONDITIONAL GO / NO-GO decision **only after** real execution and review.

## Guardrails

**[SAFE-READ]** approved diagnostics. **[LAB-MUTATE]** disposable staging with explicit blast radius and spend cap. **[CHANGE-CONTROL]** production fault injection, routing, identity, GPU reset, drain and rollback. Use synthetic requests, redact logs, never collect real prompts/PHI or secrets. Stop drills if cost/SLO/tenant-isolation guardrails are exceeded.

## Lab 0 — Baseline inventory

```bash
kubectl config current-context
kubectl get nodes -L topology.kubernetes.io/zone -o wide
kubectl get deployments,pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -40
kubectl get svc,endpointslices -A
kubectl get resourcequota,pdb,hpa -A
```

Record TrueFoundry/agent, Kubernetes, CNI/ingress, cloud region, GPU SKU/driver, vLLM, model/image digests, autoscaler, quotas, SLOs, TTFT/TPOT, stream completion, owner, incident commander, change ticket and cost budget.

## Lab 1 — Six game days

1. Agent/control-plane degraded; independently test serving inference.
2. GPU Pod Pending with quota/autoscaler/driver triage.
3. Bad disposable staging image/model rollout and supported rollback.
4. Gateway/DNS/TLS/EndpointSlice and SSE streaming interruption.
5. Bounded CUDA OOM/KV cache/queue/TTFT/TPOT diagnosis (tabletop if unsafe).
6. Compound multi-layer incident with broken telemetry and on-call escalation.

For each: UTC start/end, authorized fault, expected vs actual, synthetic traffic mix, detection, SEV, blast radius, hypotheses, sanitized evidence, mitigation, rollback, recovery SLO, cost and reviewer.

## Lab 2 — Forty-eight scenarios

| Domain | IDs and checks |
|---|---|
| Control plane | 01 agent unavailable; 02 stale registration; 03 expired identity; 04 RBAC; 05 reconcile queue; 06 stale UI |
| Scheduling | 07 GPU quota; 08 taint; 09 affinity; 10 topology; 11 device plugin; 12 cloud SKU |
| Deployment | 13 image digest; 14 registry auth; 15 init; 16 config; 17 startup probe; 18 crash loop |
| GPU/runtime | 19 driver mismatch; 20 device injection; 21 XID/ECC; 22 host OOM; 23 CUDA OOM; 24 NCCL |
| Model | 25 artifact IAM; 26 model load; 27 tokenizer; 28 KV cache; 29 cold start; 30 tensor parallel |
| Performance | 31 TTFT; 32 TPOT; 33 queue; 34 retry storm; 35 streaming; 36 slow autoscale |
| Network | 37 DNS; 38 TLS; 39 502; 40 503; 41 504; 42 EndpointSlice |
| Operations | 43 zone loss; 44 N-1 shortfall; 45 missing metrics; 46 paging; 47 rollback artifact; 48 handoff |

Label each as **executed**, **tabletop**, **not tested** or **blocked**, and include evidence link, owner, expected/actual, mitigation, rollback and review status.

## Lab 3 — Fourteen acceptance gates

| Gate | Required evidence | Status |
|---|---|---|
| G1 | Authorized environment and cost cap | Pending |
| G2 | SEV, commander and escalation | Pending |
| G3 | UTC incident timeline and sanitized evidence | Pending |
| G4 | TrueFoundry agent/control-plane diagnosis | Pending |
| G5 | Scheduler, GPU quota and autoscaler diagnosis | Pending |
| G6 | Deployment, image and startup diagnosis | Pending |
| G7 | DNS, TLS, gateway and streaming diagnosis | Pending |
| G8 | NVIDIA/CUDA/vLLM diagnosis | Pending |
| G9 | Independent synthetic SLO recovery | Pending |
| G10 | Metrics, traces and paging validation | Pending |
| G11 | Safe rollback verified | Pending |
| G12 | N-1 redundancy and GPU cost | Pending |
| G13 | RCA, action owners and due dates | Pending |
| G14 | Reviewed GO/NO-GO decision | Pending |

## Lab 4 — 100-point scorecard

| Area | Maximum | Earned |
|---|---:|---|
| Incident command and timeline | 10 | Pending |
| Agent/reconciliation | 10 | Pending |
| Scheduling/GPU quota/autoscaler | 15 | Pending |
| Deployment/model startup | 15 | Pending |
| Network/TLS/streaming | 15 | Pending |
| CUDA/vLLM performance | 15 | Pending |
| Observability/evidence | 10 | Pending |
| Rollback/recovery/postmortem | 10 | Pending |
| **Total** | **100** | **Not assessed** |

Training thresholds: GO >=90 without critical failure; CONDITIONAL GO 80–89 with accepted risk; NO-GO <80 or critical failure. Never record GO until execution and review.

## Evidence template

```text
Exercise ID / incident ID:
Change approval / environment / region:
Start UTC / end UTC:
TrueFoundry, Kubernetes, vLLM, driver, GPU:
Image/model digests and deployment revision:
Customer SLO proxy and synthetic traffic profile:
Gateway, Service/EndpointSlice, Pod Events:
Agent/reconciliation and cloud quota evidence:
CUDA/GPU/queue/TTFT/TPOT/streaming evidence:
Timeline, hypotheses, root cause confidence:
Mitigation, rollback and SLO recovery:
N-1 capacity and cost impact:
Executed / tabletop / blocked:
RCA, owners, reviewer and decision:
```

## Cleanup and decision

Restore approved staging baseline, remove disposable faults, confirm healthy routing, model version, GPU capacity, alerts, streaming completion and spend. Collect platform, SRE, security, application and FinOps review. Publication is not acceptance.

**Next:** Part 14 — SRE Runbooks.
