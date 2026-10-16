# Chapter 14.7 — Post-Incident Validation and Reliability Acceptance Runbook

**Part 14 — TrueFoundry SRE Runbooks | Final capstone**

## Mission

Prove that a TrueFoundry GPU inference service has recovered after an incident, not merely that a deployment is green. Integrate customer-path SLOs, Kubernetes/agent reconciliation, GPU health, vLLM inference, streaming, security, observability, cost, incident communications and evidence-based operational acceptance.

**[SAFE-READ]** approved inspections and synthetic requests. **[LAB-MUTATE]** isolated staging drills. **[CHANGE-CONTROL]** production traffic, rollback, capacity, identity, alert routing and node operations. Formal technical/security review and live exercises remain pending until independently performed and signed off.

## 1. Recovery versus closure

- **Mitigated:** customer impact reduced; root cause or risk may persist.
- **Recovered:** customer-path inference and agreed SLOs restored for the approved observation window.
- **Stable:** alerts, telemetry, capacity and platform reconciliation remain healthy; no repeated failure pattern.
- **Resolved:** incident commander confirms closure, stakeholder update and handoff.
- **Accepted:** all mandatory gates, evidence and accountable reviewer approvals complete.
- **Prevented:** corrective/preventive actions verified in follow-up; not implied by closure.

Never declare full recovery from Pod Ready, TrueFoundry UI Connected, a rollback command, or one HTTP 200 alone.

## 2. End-to-end acceptance architecture

```text
Customer synthetic inference and complete SSE stream
        │ availability / TTFT / TPOT / token mix / errors
        ▼
Gateway / DNS / TLS / Service / EndpointSlice
        ▼
vLLM / queue / KV cache / model artifact and runtime
        ▼
Kubernetes Pods / scheduler / autoscaler / GPU nodes
        ▼
NVIDIA driver / device plugin / DCGM / quota / N-1
        ▲
TrueFoundry desired ↔ observed / tfy-agent reconciliation
        │
Metrics, logs, traces, alerts, security, FinOps and audit
```

Evaluate both existing serving models and a safe staging deployment change to prove the control plane can still reconcile.

## 3. Recovery SLO and evidence windows

Define approved per-model/tenant/region eligible requests, error-budget policy, p95 TTFT, p95 TPOT, stream completion, gateway errors, queue wait, capacity and telemetry freshness. Use stable workload token distributions, request-volume confidence and multiple independent signals. The stability window is policy-defined; examples are 30–60 minutes for some incidents, but longer windows may be required for infrequent traffic, failover, periodic jobs or cloud capacity recovery. Do not turn example thresholds into contractual SLOs.

## 4. Read-only evidence pack

```bash
kubectl config current-context
kubectl get nodes -L topology.kubernetes.io/zone -o wide
kubectl get deployments,pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -80
kubectl get svc,endpointslices -A
kubectl get resourcequota,pdb,hpa -A
kubectl -n MODEL_NAMESPACE describe deployment MODEL_DEPLOYMENT
kubectl -n MODEL_NAMESPACE describe pod MODEL_POD
kubectl -n MODEL_NAMESPACE logs MODEL_POD --since=30m --tail=150
```

Namespace-scope commands where RBAC requires. Redact logs and operational metadata. Inspect actual TrueFoundry agent namespace, platform status and reconciliation signals using installed-version-supported interfaces; do not assume a universal agent API or metric name.

## 5. Customer-path acceptance

Validate through approved customer-equivalent gateway route:
1. Authenticated synthetic request, correct HTTP status and output contract.
2. TTFT, TPOT and end-to-end latency at representative prompt/output token lengths.
3. Complete SSE stream with expected protocol termination; a 200 followed by truncation fails.
4. 429/5xx, retry behavior, request success and error budget.
5. At least one cold/warm model case where applicable.
6. Correct tenant/region isolation and approved routing.
7. Repeat over approved stability window with alerting enabled.

Synthetic data only; no customer prompts or PHI in acceptance evidence.

## 6. Platform and infrastructure acceptance

**TrueFoundry:** desired/observed revisions, agent health, new safe staging reconciliation, deployment lineage and no recurring 401/403/429/TLS errors.

**Kubernetes:** Deployment/Pod readiness, no recurring restart loops, Service/EndpointSlice correctness, healthy nodes, quotas, PDB and autoscaler state.

**GPU:** allocatable/requested resources, driver/device plugin, DCGM freshness, XID/ECC/thermal, GPU memory, N-1 or approved degraded-mode capacity, cloud quota and zone availability.

**vLLM:** model/runtime digest, startup, queue, KV cache, TTFT/TPOT, tokens/sec, CUDA vs host OOM, streaming and graceful drain.

**Security:** no temporary excessive RBAC, exposed secrets, bypassed TLS, disabled policies or lingering emergency credentials; involve security for suspected exposure.

**FinOps:** record GPU-hours, warm capacity, retries and cost per successfully served output tokens; assess impact of temporary mitigation.

## 7. Alerts and monitoring restoration

Confirm synthetic probes, Prometheus scrapes, log ingestion, tracing where installed, DCGM/vLLM exporter freshness, paging routes, alert silences, inhibition and dashboards. Remove only incident-specific temporary silences after authorization. A healthy service with muted alerts is not fully accepted.

## 8. Root cause, RCA/COA and prevention

Separate trigger, contributing factors, detection gaps and impact amplification. Build a UTC timeline from verifiable events and immutable release IDs. RCA must cite evidence and confidence, not speculation. Assign corrective and preventive actions with owner, due date, test, risk and acceptance evidence. Distinguish remediation deployed from remediation validated; postmortem approval is separate from incident closure.

## 9. Recovery decision matrix

| Evidence | Condition | Decision |
|---|---|---|
| Customer SLO recovered, all mandatory gates pass | verified stability window | eligible for closure review |
| Inference healthy, agent reconciliation broken | control-plane deployment risk persists | conditional recovery; remain open or documented degraded mode |
| Pods healthy, streams truncate | customer-path defect | not recovered |
| GPU errors persist, spare capacity healthy | risk not eliminated | mitigation only; escalation and replacement plan |
| Telemetry stale or paging disabled | cannot trust recovery signals | no full acceptance |
| Security controls weakened | unacceptable residual risk | block acceptance |
| SLO restored, quota/N-1 below policy | resilience deficit | conditional sign-off only under explicit risk acceptance |
| No rollback artifact | future recovery risk | block change readiness, assign urgent COA |

## 10. Sixteen operational runbooks

1. Incident evidence and UTC chronology.
2. Customer synthetic availability.
3. SSE completion and gateway.
4. TTFT/TPOT and workload normalization.
5. Error-budget and stability window.
6. TrueFoundry agent and reconciliation.
7. Kubernetes Pod/EndpointSlice.
8. GPU driver/device plugin/DCGM.
9. GPU N-1/quota/zone capacity.
10. vLLM queue/KV cache/memory.
11. Security/identity/policy restoration.
12. Observability freshness and paging.
13. Rollback provenance and drift.
14. FinOps and cost regression.
15. Incident closure and stakeholder notice.
16. RCA/COA and reliability acceptance board.

Each runbook includes owner, trigger, evidence, acceptance criterion, failure disposition and escalation.

## 11. Forty-eight acceptance failure scenarios

| Domain | Scenarios |
|---|---|
| Customer path | 01 synthetic missing; 02 HTTP 200 but truncated SSE; 03 5xx persists; 04 429 surge; 05 TTFT p95 fails; 06 TPOT p95 fails |
| TrueFoundry | 07 agent disconnected; 08 reconciliation stale; 09 UI status stale; 10 desired/observed drift; 11 credential error; 12 staging reconcile fails |
| Kubernetes | 13 Pod Ready but wrong model; 14 CrashLoop recurs; 15 EndpointSlice empty; 16 Pod Pending; 17 quota exhausted; 18 PDB violated |
| GPU | 19 allocatable zero; 20 plugin stale; 21 XID; 22 ECC; 23 thermal throttle; 24 zone capacity shortage |
| vLLM | 25 queue spike; 26 KV cache pressure; 27 CUDA OOM; 28 host OOM; 29 model cold start; 30 throughput collapse |
| Network | 31 TLS expiry; 32 DNS failure; 33 proxy timeout; 34 stream buffering; 35 retry storm; 36 tenant routing error |
| Observability/security | 37 metrics stale; 38 alert silenced; 39 paging misrouted; 40 DCGM absent; 41 excess RBAC; 42 exposed secret |
| Process/FinOps | 43 rollback unavailable; 44 no stability window; 45 unverified RCA; 46 missing COA owner; 47 cost/token spike; 48 premature closure |

## 12. Six integrated acceptance game days

1. Bad model revision with SSE truncation and rollback.
2. Agent reconciliation failure while inference continues.
3. GPU XID/device-plugin issue plus N-1 capacity deficit.
4. vLLM KV cache and TTFT/TPOT regression under bounded staging load.
5. Network/TLS failure with paging and customer communications.
6. Compound failure: GPU shortage, stale telemetry, incident handoff and postmortem.

Use tabletop for destructive or risky cases. Score detection, evidence, mitigation, customer-path recovery, rollback, observability, risk and owner sign-off.

## 13. Fourteen mandatory acceptance gates

| Gate | Requirement | Weight |
|---|---|---:|
| G1 | Approved scope, change and evidence custody | 5 |
| G2 | Customer-path availability and output contract | 10 |
| G3 | Complete SSE streaming and gateway | 10 |
| G4 | TTFT/TPOT, queue and error budget | 10 |
| G5 | TrueFoundry agent/reconciliation | 8 |
| G6 | Kubernetes Pod/Service/node health | 7 |
| G7 | GPU/DCGM hardware and schedulable resources | 8 |
| G8 | N-1/cloud quota/topology | 7 |
| G9 | vLLM KV/cache/OOM/performance | 8 |
| G10 | Security/identity/policy restored | 7 |
| G11 | Telemetry, alerts and paging re-armed | 7 |
| G12 | Rollback lineage and drift-free desired state | 5 |
| G13 | Cost, communications and RCA/COA owners | 4 |
| G14 | Stability window and accountable sign-off | 4 |
| | **Total** | **100** |

**Scoring:** record earned points with direct evidence; every mandatory gate must pass and total must be at least 90/100 for recommended acceptance. Security, customer-path streaming, paging and rollback failures are automatic blockers regardless of score. Actual organization policy and risk authority supersede this illustrative rubric. Never mark gates passed solely because the Markdown file was published.

## 14. Audit and final handoff

Record incident ID, UTC start/mitigation/recovery, customer impact, SLO evidence, TrueFoundry/model/image revisions, cluster/GPU, change approvals, rollback proof, stability window, security sign-off, alerts restored, vendor cases, residual risks, RCA/COA owners, reviewer and acceptance decision. Archive sanitized evidence under approved retention and access controls.

See the [companion acceptance lab](../labs/07-post-incident-validation-reliability-acceptance-check.md).

**Part 14 publication complete: seven canonical runbooks and seven companion labs.** Formal technical/security review, live execution and operational acceptance pending.

**Next:** Part 15 — End-to-End Production Project, beginning with architecture and requirements.
