# Chapter 15.7 — Final Hands-on Lab: Production Readiness and Enterprise Acceptance

**Part 15 — End-to-End Production Project | Final Capstone | Live execution pending**

## Objective

Perform a formal evidence-backed production readiness review (PRR) of the TrueFoundry GPU inference platform. Review Chapters 15.1–15.6, execute safe staging checks, score mandatory gates and record a defensible GO / CONDITIONAL GO / NO-GO / DEFERRED decision by authorized reviewers.

**Safety:** read-only inventory unless staging change approval explicitly permits synthetic load, GPU tests, canary and rollback. Never test against production or customer data without separate change control. Keep secrets/PHI/private prompts out of evidence.

## Lab 0 — PRR scope and baseline

Record change ticket, cloud/cluster/region, TrueFoundry workspace and agent, model/image/tokenizer/vLLM/CUDA revision, GPU SKU, approved SLOs, RTO/RPO, staging synthetic endpoint, reviewers, budget, evidence location and exception policy.

```bash
kubectl config current-context
kubectl get nodes -L topology.kubernetes.io/zone -o wide
kubectl get deployments,pods -A -o wide
kubectl get resourcequota,pdb,hpa -A
kubectl get svc,endpointslices -A
kubectl get networkpolicies -A
kubectl get events -A --sort-by=.lastTimestamp | tail -80
```

## Lab 1 — Architecture, IaC and security review

Inspect Chapter 15.1 ADRs, Chapter 15.2 IaC/GPU foundation and Chapter 15.4 network/GitOps policy. Verify cloud identity, namespace/RBAC, secret references, private DNS/TLS, tenant isolation, provenance, signed immutable artifacts, policy gates and rollback ownership. Collect independent security review evidence.

## Lab 2 — TrueFoundry and model serving

Verify Chapter 15.3 agent desired/observed reconciliation, approved model artifact checksum, image digest, tokenizer/vLLM/CUDA matrix, GPU scheduling, startup/warmup, readiness and authorized synthetic inference with complete SSE. Negative-test unauthorized and cross-tenant access.

## Lab 3 — Observability, load and capacity

Using Chapter 15.5 approved staging profiles, verify representative TTFT/TPOT, 429/5xx, queue, KV cache, output correctness, GPU/DCGM freshness, alert routing and synthetic availability. Validate safe measured throughput, GPU N-1, rolling surge, cloud quota, warmup, cost per 1,000 successful output tokens and budget ceiling. Abort unsafe tests.

## Lab 4 — Recovery and incident readiness

Review Chapter 15.6 executed/tabletop game days, incident command, customer communications, RTO/RPO, security incidents, canary rollback, alert restoration, RCA/COA and unresolved risk. Live DR restore must be separately evidenced if policy requires it.

## Lab 5 — Ten acceptance game days

1. GPU N-1.
2. Agent reconciliation.
3. CUDA startup.
4. Artifact identity.
5. TLS/auth/tenant.
6. SSE drain.
7. TTFT/TPOT.
8. Quota/autoscaler.
9. Telemetry/pager.
10. Canary rollback.

Each: approval, UTC, expected/actual, synthetic impact, mitigation, security, cost, evidence and reviewer. Tabletop does not count as a live test.

## Sixty-scenario audit

01 ADR; 02 region; 03 dependency; 04 SLO; 05 ownership; 06 threat; 07 quota; 08 AZ; 09 IP; 10 node; 11 plugin; 12 CUDA; 13 agent; 14 RBAC; 15 reconcile; 16 control plane; 17 version; 18 conflict; 19 artifact; 20 mutable; 21 tokenizer; 22 crash; 23 OOM; 24 readiness; 25 TLS; 26 DNS; 27 route; 28 tenant; 29 502/503; 30 SSE; 31 secret; 32 IAM; 33 egress; 34 admission; 35 provenance; 36 audit; 37 TTFT; 38 TPOT; 39 queue; 40 KV; 41 cold start; 42 contract; 43 scrape; 44 DCGM; 45 alert; 46 pager; 47 data logs; 48 synthetic; 49 N-1; 50 autoscaler; 51 rollback; 52 DR; 53 incident; 54 RTO; 55 approval; 56 cost; 57 budget; 58 exception; 59 security; 60 premature GO.

## Twenty final gates

| Gate | Evidence required | Status |
|---|---|---|
| G01 | Architecture/ADRs | Pending |
| G02 | Cloud/IaC/RBAC | Pending |
| G03 | GPU/CUDA/device plugin | Pending |
| G04 | GPU N-1/quota | Pending |
| G05 | TrueFoundry reconciliation | Pending |
| G06 | Immutable release bundle | Pending |
| G07 | vLLM warmup/readiness | Pending |
| G08 | TLS/auth/tenant | Pending |
| G09 | Network/secrets | Pending |
| G10 | GitOps/CI/provenance | Pending |
| G11 | Synthetic correctness | Pending |
| G12 | Complete SSE | Pending |
| G13 | TTFT/TPOT/queue | Pending |
| G14 | Load/soak/capacity | Pending |
| G15 | DCGM/metrics | Pending |
| G16 | Alerts/pager/on-call | Pending |
| G17 | Canary/rollback | Pending |
| G18 | Incident/DR/RTO | Pending |
| G19 | FinOps/cost | Pending |
| G20 | Security/change sign-off | Pending |

## Evidence and decision template

```text
PRR ID / UTC / change approval:
Cloud/cluster/TrueFoundry/model release:
Technical, security, SRE and service reviewers:
G01–G20 PASS/FAIL/BLOCKED/PENDING:
Executed vs tabletop game days:
Customer-path SSE, correctness and SLO:
GPU compatibility, N-1, capacity, cost:
Security/tenant/secrets/provenance:
Observability, alerts and paging:
Canary, rollback, incident and DR:
Residual risks, owners, deadlines, exception expiry:
Decision: GO / CONDITIONAL GO / NO-GO / DEFERRED
Authorized sign-off:
Handoff/on-call and next review:
```

## Exit criteria

The **written curriculum** is complete when the final chapter and lab are published and indexed. **Operational acceptance** requires current evidence, independent technical/security review, required live lab execution, signed decision and handoff. If any mandatory gate remains pending, record **DEFERRED / NO-GO according to policy**, never automatic GO.

**Next phase:** Execute and evidence all chapters' labs, close RCA/COA and security findings, perform independent PRR and formally authorize production only after acceptance.
