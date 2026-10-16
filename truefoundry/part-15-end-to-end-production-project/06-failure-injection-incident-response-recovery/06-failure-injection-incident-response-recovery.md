# Chapter 15.6 — Failure Injection, Incident Response and Recovery

**Part 15 — End-to-End Production Project | Capstone 6 of 7**

## Objective

Prove that a TrueFoundry-managed Kubernetes GPU/vLLM inference platform can detect, contain, mitigate and recover from realistic failures while protecting customers, security and data. Integrate the architecture, GPU foundation, serving, GitOps and SLO work from Chapters 15.1–15.5.

**Publication is not a completed game day or production resilience certification.** This chapter defines exercises and evidence; live execution, independent review and sign-off remain pending.

## 1. Safety and blast-radius contract

- **[SAFE-READ]** inspect approved cluster, alerts, telemetry and change history.
- **[TABLETOP]** model destructive or high-risk faults without touching infrastructure.
- **[LAB-MUTATE]** bounded staging-only faults with isolated synthetic traffic, approved scope, rollback owner, abort thresholds and GPU-hour budget.
- **[CHANGE-CONTROL]** production traffic, GPU node drains, network disruption, IAM/secret rotation, DNS/TLS, TrueFoundry agent, autoscaling and rollback.

**Never** inject faults into customer-facing inference, production GPU nodes, shared ingress, cloud identities, or monitoring without explicit change authorization. Do not log credentials, real prompts, PHI, model output or private customer identifiers.

### Required approval envelope

```text
Exercise ID / owner / change ticket:
Cluster/context/namespace/region/zone:
Approved start/end UTC:
Synthetic route and workload:
Permitted fault and blast radius:
Forbidden resources and actions:
SLO baseline and abort thresholds:
Incident commander / safety observer:
Rollback artifact and operator:
GPU-hour cap and cloud spend ceiling:
Pager/communications plan:
Evidence location and retention:
```

## 2. Failure domains and dependencies

```text
TrueFoundry control plane → agent reconciliation → desired deployment
                                  │
GitOps/CI → immutable model revision → Kubernetes GPU workload
                                  │
Cloud GPU pool → driver/plugin → CUDA/vLLM → gateway/SSE → client
                                  │
            Prometheus/logs/traces/DCGM/synthetic probes
                                  │
                      alert → incident commander
                                  │
                mitigate → rollback → verify → RCA/COA
```

A control-plane outage may block *new* releases without stopping *existing* inference. Distinguish serving SLO from reconciliation SLO and avoid unnecessary rollback during healthy serving.

## 3. Recovery objectives and evidence

Record approved availability SLO, eligible requests, TTFT, TPOT, complete SSE stream rate, 429/5xx, queue, model correctness, GPU capacity and error budget. Define scenario-specific **RTO** (restore customer service) and **RPO** (loss tolerance for stateful components, if any). Stateless inference can still depend on model artifacts, configuration, logs, audit and other stateful services. Do not invent contractual RTO/RPO.

Collect UTC detection, declaration, mitigation, recovered, stable and closed times. Keep immutable image/model/tokenizer/vLLM revision, agent desired/observed, GPU SKU/driver, Kubernetes events, gateway logs, alerts and customer synthetic evidence. Sanitize everything.

## 4. Preflight and rollback readiness

```bash
kubectl config current-context
kubectl get nodes -L topology.kubernetes.io/zone -o wide
kubectl get deployments,pods -A -o wide
kubectl get svc,endpointslices -A
kubectl get hpa,pdb,resourcequota -A
kubectl get events -A --sort-by=.lastTimestamp | tail -80
```

Before any fault, confirm cluster context, dedicated staging namespace, approved test route, baseline synthetic SSE, telemetry freshness, on-call ownership, rollback through **authoritative TrueFoundry desired state**, immutable artifacts and spare GPU capacity. Do not rely on `kubectl rollout undo` for TrueFoundry-managed workloads unless vendor-supported and reconciled.

## 5. Incident command and communications

Roles: incident commander (decision and severity), operations lead (diagnosis/mitigation), safety officer (stop authority), communications lead (stakeholder updates), scribe (UTC timeline), security lead (suspected exposure) and cloud/vendor liaison. Small teams may combine roles but maintain explicit accountability.

Severity is determined by customer impact, scope, duration, security/data risk and SLO burn. Use approved internal SEV definitions. Communications should distinguish confirmed facts from hypotheses, avoid speculative root causes, identify mitigation status and provide next update time. Escalate early when GPU quota, cloud capacity, vendor control plane or customer security is involved.

## 6. Fault injection method

1. State hypothesis, expected blast radius and observable customer symptom.
2. Verify synthetic-only route, fault boundary and change approval.
3. Record baseline over an agreed window.
4. Apply a **single reversible staging fault** or use tabletop for destructive cases.
5. Detect via independent customer-path probe and infrastructure signals.
6. Declare incident, assign roles, execute diagnostic runbook.
7. Contain/mitigate, restore through approved rollback and remove fault.
8. Verify complete SSE, TTFT/TPOT, errors, model correctness, GPU and agent state.
9. Observe approved stability window, restore paging and security controls.
10. Document timeline, MTTA/MTTR definitions, error budget, RCA/COA and sign-off.

**Stop** on unintended production impact, missing rollback, XID/ECC hardware faults, secret exposure, uncontrolled scaling, budget breach, absent telemetry or safety observer request.

## 7. Scenario A — TrueFoundry agent reconciliation failure

**Hypothesis:** existing serving remains healthy while deployment changes cannot reconcile. **Preferred exercise:** tabletop or benign staging reconciliation validation; do not intentionally disable a shared production agent.

Evidence: agent status and sanitized logs, desired/observed revision, TLS/DNS/identity, Kubernetes workload health, synthetic SSE. Mitigation: hold releases, restore supported agent connectivity/permissions, revalidate drift and staging reconciliation. Declare customer impact only if actual inference is affected.

## 8. Scenario B — GPU node/zone failure and N-1

**Hypothesis:** remaining schedulable GPUs absorb the failed node/zone without breaching approved customer SLO, or documented degraded-mode threshold applies. Prefer tabletop for cloud AZ loss and production node failures.

Check node readiness, GPU allocatable, device plugin, DCGM/XID/ECC, PDB, Pod anti-affinity, autoscaler max, quota, model warmup, VRAM and rolling surge. If safe staging node removal is explicitly approved, require independent observer, spare capacity and rollback. Do not force-drain GPU nodes or delete critical Pods without authorization.

## 9. Scenario C — vLLM queue/KV cache and CUDA OOM

Use bounded synthetic prefill/decode cohorts to observe TTFT/TPOT, queue, KV cache, VRAM, tokens/sec and streaming completion. Avoid deliberately exhausting shared GPU memory. If limits approach thresholds, abort load and reduce concurrency/context under approved model settings. Recover by reverting compatible release and verifying warmup.

## 10. Scenario D — Gateway, DNS, TLS and SSE

Exercise tabletop certificate expiry, DNS/egress failure, gateway auth policy or staging-only timeout change. Validate 401/403 for unauthorized synthetic clients, valid TLS chain, Service/EndpointSlice, proxy buffering, stream completion and graceful drain. A 200 followed by truncated SSE is an outage symptom.

## 11. Scenario E — GitOps/TrueFoundry release regression

Introduce a reversible staging-only invalid model revision or harmless configuration regression through approved deployment workflow. Confirm CI policy and provenance, canary SLO abort, authoritative rollback, old artifact accessibility and no controller ownership conflict. Measure detection and recovery; do not bypass admission/security policy.

## 12. Scenario F — Observability outage and blind spots

Tabletop or isolated staging target scrape failure; ensure customer synthetic probes, alert routing and fallback signals remain available. If both telemetry and customer-path checks are unavailable, **do not** declare full recovery. Restore scrapes, remove authorized incident silences and prove paging.

## 13. Scenario G — Compound failure

Combine tabletop GPU shortage, delayed TrueFoundry reconciliation, gateway SSE truncation and stale DCGM metrics. Evaluate prioritization, customer impact, error budget, cloud/vendor escalation, rollback and shift handoff. Run only tabletop unless a narrowly approved staging plan explicitly authorizes each dependency.

## 14. Recovery verification matrix

| Gate | Evidence | Failure disposition |
|---|---|---|
| Customer path | authenticated synthetic inference, output contract | block closure |
| Streaming | complete SSE with protocol termination | block closure |
| Latency | TTFT/TPOT by approved token cohort | degraded until restored |
| Errors | eligible 429/5xx and error budget | degraded or incident |
| Control plane | agent and desired/observed reconciliation | deployment risk |
| Kubernetes | Pods, nodes, Service/EndpointSlice, quota | infrastructure risk |
| GPU | driver/plugin, DCGM, XID/ECC, capacity | hardware risk |
| vLLM | queue, KV cache, CUDA/host OOM, warmup | serving risk |
| Security | TLS, auth, tenant isolation, secrets/RBAC | automatic blocker |
| Observability | metrics freshness, synthetic, alerts/pager | automatic blocker |
| Rollback | immutable artifact and authoritative revision | automatic blocker |
| Stability | policy-defined observation window | no closure before proof |

Recovery is not established by Pod Ready, UI Connected or a single HTTP 200.

## 15. Twenty operational runbooks

1. Exercise approval and abort.
2. Baseline SLO/evidence capture.
3. Incident declaration and SEV.
4. Roles and stakeholder communications.
5. Agent/reconciliation outage.
6. Kubernetes scheduling and Pending.
7. GPU node/zone loss.
8. Driver/plugin/XID/ECC.
9. N-1 and quota shortage.
10. vLLM queue and KV cache.
11. CUDA/host OOM.
12. Gateway/DNS/TLS.
13. SSE truncation and drain.
14. Auth/tenant isolation.
15. GitOps/TrueFoundry drift.
16. Canary failure and rollback.
17. Telemetry/paging outage.
18. Vendor/cloud escalation.
19. Recovery/stability verification.
20. RCA/COA and closure.

## 16. Fifty failure scenarios

| Domain | Scenarios |
|---|---|
| Control plane | 01 agent disconnected; 02 DNS; 03 TLS; 04 token expired; 05 reconciliation drift |
| Kubernetes | 06 Pending; 07 CrashLoop; 08 EndpointSlice empty; 09 PDB blocked; 10 quota exceeded |
| GPU | 11 node NotReady; 12 plugin absent; 13 CUDA mismatch; 14 XID; 15 ECC |
| Capacity | 16 N-1 shortfall; 17 zone shortage; 18 autoscaler max; 19 surge unavailable; 20 warmup delayed |
| vLLM | 21 TTFT high; 22 TPOT high; 23 queue saturation; 24 KV pressure; 25 CUDA OOM |
| Gateway | 26 DNS failure; 27 TLS expiry; 28 401/403 regression; 29 502/503; 30 SSE truncation |
| Release | 31 bad model digest; 32 tokenizer mismatch; 33 GitOps conflict; 34 canary abort; 35 rollback artifact missing |
| Telemetry | 36 Prometheus down; 37 DCGM stale; 38 synthetic missing; 39 pager muted; 40 log correlation gap |
| Security | 41 secret exposed; 42 excessive RBAC; 43 tenant routing leak; 44 unapproved egress; 45 audit gap |
| Process | 46 missing incident owner; 47 unclear SEV; 48 budget overrun; 49 premature closure; 50 unowned COA |

## 17. Eight integrated game days

1. Agent disconnect/reconciliation lag.
2. GPU node loss with N-1 check.
3. vLLM queue/KV cache pressure.
4. Gateway SSE timeout.
5. Canary regression and rollback.
6. Prometheus/DCGM blind spot and pager.
7. Cloud GPU quota/zone exhaustion tabletop.
8. Compound failure with incident command, communications and RCA.

For each: approval, UTC baseline, injected/tabletop fault, expected/actual SLO, time-to-detect, time-to-mitigate, rollback, GPU-hours, security, evidence and reviewer.

## 18. Sixteen acceptance gates

G1 scope/change/safety; G2 baseline and synthetic; G3 incident command; G4 agent/reconciliation; G5 Kubernetes; G6 GPU hardware/DCGM; G7 N-1/quota; G8 vLLM/TTFT/TPOT; G9 SSE/gateway; G10 security; G11 release/rollback; G12 telemetry/paging; G13 recovery stability; G14 error budget/RTO; G15 RCA/COA/cost; G16 independent sign-off.

See [companion failure-injection lab](../labs/06-failure-injection-incident-response-recovery-check.md). All gates pending until actual execution and formal review.

**Next:** Chapter 15.7 — Production Readiness and Final Enterprise Acceptance.
