# Chapter 002 — Hands-on Lab: Production LLM Service Architecture and SRE Responsibilities

**Part 01 | Chapter 002 | Read-only discovery + tabletop | Execution pending**

## Goal

Create an evidence-backed production inference architecture and ownership map, including GPU/serving dependencies, SLI eligibility, security boundaries, rollback and incident escalation.

## Guardrails

**[SAFE-READ]** approved inventory. **[LAB-MUTATE]** synthetic-only staging tests with change approval. **[CHANGE-CONTROL]** production GPU, DNS, IAM, rollout or failure injection. Do not include credentials, PHI, customer prompts or private model artifacts.

## Lab 0 — Context and approval

Document approved cluster/context, namespace, cloud, zone, gateway, model, inference runtime, GPU SKU, monitoring, service owner, on-call, reviewer, UTC window and evidence location. Record that commands may expose sensitive metadata; sanitize before sharing.

```bash
kubectl config current-context
kubectl get nodes -L topology.kubernetes.io/zone -o wide
kubectl get namespaces
kubectl get deploy,sts,ds,pods -A -o wide
kubectl get svc,endpointslices,ingress -A
kubectl get networkpolicies -A
kubectl get resourcequota,hpa,pdb -A
kubectl get events -A --sort-by=.lastTimestamp | tail -80
```

Expected: read-only inventory and scheduling/event baseline. No changes.

## Lab 1 — Trace serving and control planes

Draw synthetic client → DNS/TLS/auth gateway → Service/EndpointSlice → inference Pod → GPU → model artifact. Draw GitOps/CI, TrueFoundry/platform reconciliation and Kubernetes API separately. Annotate trust boundaries and dependencies.

## Lab 2 — Record GPU and runtime compatibility

From approved deployment and dashboards, record immutable image digest, model checksum, tokenizer, vLLM/Triton/Ray/KServe version, CUDA/driver, GPU SKU, device plugin, allocatable GPU, VRAM, topology and node quota.

```bash
kubectl -n MODEL_NAMESPACE describe deployment MODEL_DEPLOYMENT
kubectl -n MODEL_NAMESPACE get pods -o wide
kubectl -n MODEL_NAMESPACE logs deployment/MODEL_DEPLOYMENT --since=15m --tail=100
```

Do not run privileged diagnostics or disclose model secrets.

## Lab 3 — Build RACI and escalation

Assign service owner, model owner, GPU SRE, platform SRE, security, release owner, FinOps, on-call and incident commander. Record who can approve rollback, emergency GPU scaling and customer communications.

## Lab 4 — Design customer-centered SLIs

Document eligible requests, successful complete SSE, 429 treatment, TTFT/TPOT cohort, error budget, synthetic route, monitoring freshness and alert owner. Do not use illustrative values as approved production SLOs.

## Lab 5 — Six architecture game days

01 gateway 503; 02 GPU N-1 failure; 03 model artifact outage; 04 control-plane reconciliation; 05 SSE truncation; 06 telemetry blind spot. For each, record tabletop/executed, expected signal, first evidence, escalation, mitigation, rollback and recovery criteria.

## Eighteen-scenario coverage

F01 DNS; F02 TLS; F03 auth; F04 429; F05 EndpointSlice; F06 readiness; F07 Pending; F08 plugin; F09 CUDA; F10 OOM; F11 artifact; F12 tokenizer; F13 TTFT; F14 TPOT; F15 SSE; F16 reconciliation; F17 telemetry; F18 GPU quota.

## Twelve acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | Control/data plane diagram | Pending |
| G02 | Dependency and trust boundaries | Pending |
| G03 | RACI, on-call, escalation | Pending |
| G04 | Eligible SLI and SLO worksheet | Pending |
| G05 | Complete SSE success criteria | Pending |
| G06 | Immutable release manifest | Pending |
| G07 | GPU N-1 and quota worksheet | Pending |
| G08 | Identity/tenant/network controls | Pending |
| G09 | GitOps/controller ownership | Pending |
| G10 | Synthetic/telemetry and pager | Pending |
| G11 | Rollback and recovery runbooks | Pending |
| G12 | Independent reviewer approval | Pending |

## Evidence template

```text
Lab ID / UTC / approval:
Cloud/cluster/namespace:
Data-plane diagram / control-plane diagram:
Model/image/tokenizer/runtime/GPU:
Dependency and failure domains:
RACI, escalation, pager:
SLI eligibility, SSE, TTFT, TPOT:
N-1, cloud quota and GPU budget:
Trust boundaries, IAM and tenant policy:
Game-day observations (tabletop vs executed):
Findings, rollback, reviewer and status:
```

## Cleanup and exit

Read-only checks should leave no resources. Stop separately approved synthetic traffic, sanitize evidence, and leave gates pending until independently reviewed.

**Next: Chapter 003 — SLIs, SLOs, Error Budgets and Inference User Journeys.**
