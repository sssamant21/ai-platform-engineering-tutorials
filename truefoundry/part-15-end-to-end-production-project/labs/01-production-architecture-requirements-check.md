# Chapter 15.1 — Hands-on Design Lab: Production Architecture and Requirements

**Part 15 — End-to-End Production Project | Staging planning / tabletop | Execution pending**

## Objective

Produce an actionable TrueFoundry GPU inference architecture package before cloud resources are provisioned: requirements, cloud/cluster selection, model sizing, threat model, SLOs, rollout/rollback, RTO/RPO, capacity and budget.

**Safety:** use sanitized synthetic workload assumptions, approved documentation and no production credentials. Do not provision paid GPU resources during this design-only lab.

## Lab 0 — Define service contract

Record business owner, platform owner, model owner, security reviewer, cloud/region/AZ, data classification, eligible requests, availability SLO, TTFT/TPOT percentiles, complete SSE stream, error budget, RTO/RPO and cost ceiling. Leave values as **TBD** rather than inventing business commitments.

## Lab 1 — Architecture diagram and ADRs

Draw control plane vs data plane, trust boundaries, TrueFoundry agent, Kubernetes GPU pool, vLLM, gateway, registry/artifact store, identity, secrets, metrics/logs/DCGM, synthetic probes and pager. Write ten ADRs with decision, alternatives, risk, owner and rollback.

## Lab 2 — Capacity worksheet

Fill:

```text
Cloud / region / AZs:
Model and tokenizer digest:
vLLM / CUDA / driver:
GPU SKU / VRAM / GPUs per replica:
Input/output token length p50/p95:
Arrival rate and measured tokens/sec:
Replicas / max concurrency / context:
GPU allocatable vs requested:
Rolling overlap / N-1 reserve:
GPU quota / autoscaler max:
GPU-hours / spend cap / cost per successful 1k output tokens:
```

Reject designs without measured model fit and capacity evidence; use staging benchmark placeholders until tested.

## Lab 3 — Security and release design

Map identity and RBAC, ingress TLS/auth, egress, secret manager, image/model provenance, network policy, audit, canary SLO gates, rollback compatibility and streaming drain. Record reviewer and open risks.

## Lab 4 — Four tabletop game days

1. GPU AZ shortage.
2. Control-plane reconciliation outage.
3. Cross-tenant/auth/secret threat.
4. Bad model release and SSE rollback.

For each record blast radius, mitigation, customer SLO, rollback, cost, owner, reviewer and outstanding action.

## Thirty-six failure scenarios

01 owner; 02 SLO; 03 region; 04 AZ; 05 quota; 06 N-1; 07 GPU SKU; 08 VRAM; 09 tokenizer; 10 CUDA; 11 plugin; 12 agent egress; 13 reconcile; 14 RBAC; 15 identity; 16 secret; 17 public auth; 18 tenant isolation; 19 egress; 20 image provenance; 21 mutable model; 22 staging; 23 rollback; 24 canary; 25 SSE; 26 probe; 27 TTFT; 28 TPOT; 29 DCGM; 30 stale metrics; 31 pager; 32 RTO/RPO; 33 DR; 34 budget; 35 teardown; 36 sign-off.

## Fourteen acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G1 | Business scope and owners | Pending |
| G2 | Architecture and trust diagram | Pending |
| G3 | Model/runtime immutable lineage | Pending |
| G4 | GPU sizing/N-1 | Pending |
| G5 | Cloud quota and AZ availability | Pending |
| G6 | TrueFoundry integration | Pending |
| G7 | Gateway/auth/identity | Pending |
| G8 | Network and secrets | Pending |
| G9 | Release/canary/rollback | Pending |
| G10 | Synthetic TTFT/TPOT/SSE SLO | Pending |
| G11 | Metrics/alerts/pager | Pending |
| G12 | RTO/RPO/DR | Pending |
| G13 | Cost/spend cap | Pending |
| G14 | ADR and reviewer sign-off | Pending |

## Evidence and handoff

```text
Project ID / design revision / UTC:
Cloud, cluster, region, AZ:
Business/data classification and owners:
Architecture diagram / ADR links:
Model/GPU capacity and N-1:
TrueFoundry integration and trust boundaries:
SLO/error budget, RTO/RPO:
Release, canary, rollback, streaming:
Observability/pager, security controls:
GPU cost ceiling and teardown:
Open risks, blockers, owner, due date:
Reviewer and decision (pending/pass/rework):
```

**Cleanup:** no resources should be created. Store sanitized design artifacts in approved repository and leave pending gates unmarked until review.

**Next:** Chapter 15.2 — Kubernetes and GPU Infrastructure Foundation.
