# Chapter 15.6 — Hands-on Lab: Failure Injection, Incident Response and Recovery

**Part 15 — End-to-End Production Project | Staging/tabletop | Live execution pending**

## Goal

Demonstrate bounded failure detection, incident command, mitigation, authoritative rollback, customer-path recovery, telemetry restoration and evidence-backed RCA/COA for a TrueFoundry-managed Kubernetes GPU/vLLM service.

**Safety:** only approved staging synthetic traffic may be faulted. Production GPU nodes, shared agents, cloud IAM, DNS, TLS and monitoring require separate change control. Use tabletop for destructive scenarios. Stop on unexpected customer impact, missing rollback, telemetry loss, XID/ECC, security exposure or GPU budget breach.

## Lab 0 — Approval and baseline

```bash
kubectl config current-context
kubectl get nodes -L topology.kubernetes.io/zone -o wide
kubectl get deployments,pods -A -o wide
kubectl get svc,endpointslices -A
kubectl get hpa,pdb,resourcequota -A
kubectl get events -A --sort-by=.lastTimestamp | tail -80
```

Record approved cluster/namespace, staging synthetic endpoint, model/image/tokenizer revision, TrueFoundry agent, GPU quota/N-1, SLO, TTFT/TPOT, SSE completion, incident roles, pager, rollback and cost cap.

## Lab 1 — Incident command tabletop

Assign incident commander, operations lead, safety observer, communications lead and scribe. Practice SEV declaration, UTC timeline, first customer-safe update, escalation, decision log and shift handoff. Distinguish facts, hypotheses and RCA.

## Lab 2 — Control plane and GPU scenarios

Tabletop agent disconnection and GPU AZ loss. If explicitly approved, exercise only benign staging reconciliation or bounded GPU scheduling faults. Validate desired/observed revision, node allocatable, device plugin, DCGM, PDB, quota, autoscaler and model warmup.

## Lab 3 — vLLM and gateway faults

Use bounded synthetic cohorts for TTFT/TPOT/queue/KV cache, then staging-only reversible gateway SSE timeout or a tabletop. Verify complete streams, error budget, TLS/auth, 429/5xx, CUDA/host OOM signals and abort triggers.

## Lab 4 — Canary rollback and telemetry

Stage a reversible approved model/config regression, observe canary gate and restore immutable compatible bundle through authoritative TrueFoundry desired state. Validate alert/pager freshness, customer synthetic probes, agent reconciliation and GPU cost.

## Lab 5 — Eight game days

1. Agent disconnected.
2. GPU node loss and N-1.
3. vLLM KV/queue.
4. SSE timeout.
5. Bad canary and rollback.
6. Prometheus/DCGM scrape and pager.
7. Cloud GPU quota/AZ exhaustion tabletop.
8. Compound incident with handoff and RCA.

Each record tabletop/executed, approval, UTC, expected/actual, SLO, MTTA/MTTR definitions, mitigation, rollback, spend, reviewer.

## Fifty failure scenarios

01 agent; 02 DNS; 03 TLS; 04 token; 05 drift; 06 Pending; 07 CrashLoop; 08 EndpointSlice; 09 PDB; 10 quota; 11 NotReady; 12 plugin; 13 CUDA; 14 XID; 15 ECC; 16 N-1; 17 zone; 18 autoscaler; 19 surge; 20 warmup; 21 TTFT; 22 TPOT; 23 queue; 24 KV; 25 OOM; 26 DNS; 27 TLS; 28 auth; 29 502/503; 30 SSE; 31 digest; 32 tokenizer; 33 GitOps; 34 canary; 35 rollback; 36 Prometheus; 37 DCGM; 38 synthetic; 39 pager; 40 logs; 41 secret; 42 RBAC; 43 tenant; 44 egress; 45 audit; 46 owner; 47 SEV; 48 budget; 49 closure; 50 COA.

## Sixteen acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G1 | Scope/change/safety approval | Pending |
| G2 | Baseline synthetic SLO | Pending |
| G3 | Incident roles/communications | Pending |
| G4 | Agent/reconciliation | Pending |
| G5 | Kubernetes recovery | Pending |
| G6 | GPU/DCGM hardware | Pending |
| G7 | N-1/quota | Pending |
| G8 | vLLM/TTFT/TPOT | Pending |
| G9 | SSE/gateway | Pending |
| G10 | Security/tenant isolation | Pending |
| G11 | Canary/rollback | Pending |
| G12 | Telemetry/pager | Pending |
| G13 | Stability window | Pending |
| G14 | Error budget/RTO | Pending |
| G15 | RCA/COA/GPU cost | Pending |
| G16 | Independent sign-off | Pending |

## Evidence template

```text
Exercise ID / change approval / UTC:
Cluster/namespace/TrueFoundry revision:
Model/image/tokenizer/vLLM/GPU:
Fault hypothesis and blast radius:
Executed or tabletop / safety observer:
Baseline SLO, TTFT, TPOT, SSE:
Detection/SEV/communications timeline:
GPU, Kubernetes, agent, gateway findings:
Mitigation and rollback proof:
Recovered/stable timestamps and window:
Alerts, security and cost restored:
RCA confidence, COA owner/due date:
Gate blockers, reviewer and decision:
```

## Cleanup

Remove only staging faults, restore authoritative deployment and network policies, verify synthetic SSE, telemetry/pager, agent state and GPU cost cap, archive sanitized evidence. Do not mark gates passed without executed evidence and reviewer approval.

**Next:** Chapter 15.7 — Production Readiness and Final Enterprise Acceptance.
