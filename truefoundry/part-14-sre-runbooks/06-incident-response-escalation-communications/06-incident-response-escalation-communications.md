# Chapter 14.6 — Incident Response, Escalation and Communications Runbook

**Part 14 — TrueFoundry SRE Runbooks | Enterprise on-call operations**

## Mission

Provide a practical command-and-control procedure for production TrueFoundry, Kubernetes, GPU and vLLM incidents: detect, declare, coordinate, escalate, communicate, mitigate, recover, hand off and learn. The goal is rapid, safe restoration of customer-facing inference—not merely healthy infrastructure dashboards.

**[SAFE-READ]** approved telemetry and sanitized evidence. **[LAB-MUTATE]** isolated staging drills. **[CHANGE-CONTROL]** production traffic changes, node operations, credentials, model rollback and vendor escalations involving sensitive data. Never put real prompts, PHI, API keys, customer outputs or credentials in incident chat, tickets or external escalation.

## 1. Incident command roles and accountability

| Role | Responsibility | Handoff artifact |
|---|---|---|
| Incident commander (IC) | declares severity, priorities, decisions, cadence | incident timeline and decision log |
| Operations lead | technical diagnosis, mitigation, rollback | command/evidence log |
| Communications lead | customer/internal updates, stakeholder cadence | approved status messages |
| Scribe | UTC chronology, hypotheses and actions | sanitized incident record |
| Security/privacy lead | data exposure, identity and containment | security escalation record |
| Change approver | approves production mutations | change ID and rollback approval |
| Cloud/GPU specialist | GPU supply, driver/device, quota, cloud escalation | capacity/health evidence |
| Vendor liaison | TrueFoundry/cloud/NVIDIA support escalation | case IDs, sanitized bundle |

One person may hold multiple roles in a small incident, but IC and change approval must remain clear. Identify backup/on-call coverage and incident ownership before a major event.

## 2. Severity and declaration

| Level | Illustrative impact | Initial response |
|---|---|---|
| SEV1 | widespread inference outage, security compromise or rapid severe SLO burn | immediate page, IC, coordinated bridge, executive/customer comms |
| SEV2 | material model/region degradation or sustained customer-facing SLO violation | page owners, IC, time-boxed mitigation, scheduled updates |
| SEV3 | bounded or low-impact issue with workaround | ticket or on-call workflow per policy |

Use the organization's approved thresholds; these are examples, not vendor or contractual definitions. Escalate severity based on blast radius, duration, tenant sensitivity, security and error budget, not simply number of failing Pods.

## 3. First 15 minutes: incident launch

**Minute 0–5:** acknowledge page; establish UTC start, incident ID, synthetic inference and SSE completion; identify model, tenant, cluster, region and suspected blast radius; declare severity and IC.

**Minute 5–10:** freeze unrelated changes, identify recent releases, compare TrueFoundry desired vs Kubernetes observed, check gateway, Pod Events, GPU/DCGM and vLLM TTFT/TPOT/queue. Separate observed facts from hypotheses.

**Minute 10–15:** select safest reversible mitigation, assign operator and change approver, confirm rollback and next status time; page additional experts and open vendor case when needed.

Times are an illustrative training target, not a guarantee or replacement for internal paging SLAs.

## 4. Read-only evidence pack

```bash
kubectl config current-context
kubectl get nodes -L topology.kubernetes.io/zone -o wide
kubectl get deployments,pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -80
kubectl get svc,endpointslices -A
kubectl get resourcequota,pdb,hpa -A
kubectl -n MODEL_NAMESPACE describe pod MODEL_POD
kubectl -n MODEL_NAMESPACE logs MODEL_POD --since=20m --tail=150
```

Restrict to authorized namespaces when cluster-wide visibility is unavailable. Sanitize logs and preserve source, UTC timestamp, query window, image/model revision and observation owner.

## 5. Cross-layer triage tree

```text
Customer-path synthetic SLO degraded?
 ├─ No → verify observability, platform control plane, pending deployment risk
 └─ Yes
     ├─ 401/403/429 → auth, rate limiting, policy, retries
     ├─ 502/503/504 → gateway, Service, EndpointSlice, readiness
     ├─ Pod absent/Pending → TrueFoundry reconciliation, GPU quota, scheduler
     ├─ Pod failing → image, init, CrashLoop, model load, CUDA
     ├─ Pod serving but slow → vLLM queue, TTFT, TPOT, KV cache, GPU
     └─ stream truncated → proxy timeout, drain, client, network
```

Correlate changes and telemetry; do not declare root cause solely from temporal association. See Part 13 for detailed fault diagnosis and Part 14.1–14.5 for specific runbooks.

## 6. Communications cadence and templates

Agree cadence in the incident bridge; example: SEV1 every 15–30 minutes, SEV2 every 30–60 minutes, subject to company policy and contractual commitments. Every update must state what is known, customer impact, mitigation progress, next update time and what remains unconfirmed. Avoid unapproved root-cause speculation and exposure of tenant identities.

### Internal update

```text
[UTC TIME] [INCIDENT ID] [SEVERITY]
Impact: affected model/region/tenant scope and measured SLO.
Current state: confirmed observations; unknowns explicitly identified.
Actions: approved mitigation, owner, rollback readiness.
Next update: UTC timestamp.
Escalations: teams/vendor case and blockers.
```

### External/customer update

```text
We are investigating degraded model inference in [approved scope]
since [UTC]. Customers may experience [verified symptom].
Our team is working on mitigation. Next update by [UTC].
We will provide additional details after verification.
```

### Recovery notice

```text
Service recovery was verified at [UTC] using customer-path synthetic
requests, streaming completion and agreed SLO windows.
We continue enhanced monitoring and will provide follow-up findings.
```

Do not claim recovery based only on Pod Ready or a successful rollback command.

## 7. Escalation triggers

Escalate to platform/vendor when agent registration/reconciliation is stale beyond policy; to Kubernetes/cloud for unschedulable GPU and quota; to NVIDIA/hardware for XID/ECC; to security for possible exposure/credential compromise; to network for TLS/ingress; to application/model owner for inference correctness and vLLM degradation. Include sanitized UTC timeline, revision, logs, Events, synthetic impact, attempted mitigations, region/GPU SKU and case priority. Never share full secrets or unredacted customer data.

## 8. Mitigation, change control and rollback

Prefer reversible, low-blast-radius actions: stop retry amplification, shift to approved healthy capacity, restore compatible prior model/image/config via authoritative TrueFoundry desired state, scale within quota, then perform infrastructure changes only under approval. Every mutation needs owner, expected outcome, abort condition, rollback and post-change customer-path validation. Do not mass-drain GPU nodes or delete controller-managed resources as an ad hoc incident shortcut.

## 9. Recovery and incident closure

Validate independently: eligible request availability, TTFT/TPOT, SSE completion, gateway errors, queue, GPU/DCGM health, TrueFoundry reconciliation, Kubernetes readiness, telemetry freshness and paging re-arm. Observe for a policy-defined stability window, record remaining risks, communicate recovery, transfer to problem management and assign RCA/COA owners.

## 10. Shift handoff and evidence integrity

Handoff must include severity, current customer impact, active mitigation, known-good revision, pending changes, escalation/case IDs, next update time, on-call contacts, UTC timeline, rollback plan and open risks. Mark evidence as fact, hypothesis or conclusion. Use approved retention and access controls.

## 11. Eighteen operational runbooks

1. Alert acknowledgment and SEV declaration.
2. Incident bridge and role assignment.
3. Customer-path synthetic impact assessment.
4. UTC timeline and change correlation.
5. TrueFoundry control-plane/agent escalation.
6. GPU scheduling and cloud quota escalation.
7. Kubernetes startup and Pod failure escalation.
8. NVIDIA XID/ECC hardware escalation.
9. vLLM TTFT/TPOT/queue performance escalation.
10. Gateway, TLS and streaming escalation.
11. Security/privacy incident escalation.
12. Change freeze and approval.
13. Reversible mitigation and traffic routing.
14. Model rollback and compatibility.
15. Vendor support evidence bundle.
16. Internal and external updates.
17. Shift handoff and incident ownership.
18. Recovery verification and postmortem initiation.

## 12. Forty-eight failure scenarios

| Domain | Scenarios |
|---|---|
| Detection | 01 no page; 02 noisy alert storm; 03 stale dashboard; 04 synthetic probe missing; 05 low-traffic SLO misread; 06 incident unacknowledged |
| Command | 07 no IC; 08 conflicting operators; 09 unclear severity; 10 no change approver; 11 decision log missing; 12 handoff gap |
| Platform | 13 agent disconnected; 14 reconciliation stalled; 15 UI status stale; 16 registry auth; 17 bad model revision; 18 rollback artifact unavailable |
| GPU | 19 GPU quota; 20 autoscaler max; 21 Pending Pod; 22 device plugin; 23 XID/ECC; 24 zone loss |
| Inference | 25 CUDA OOM; 26 host OOM; 27 KV cache; 28 TTFT; 29 TPOT; 30 queue |
| Network | 31 502; 32 503; 33 504; 34 TLS expiry; 35 SSE truncation; 36 retry storm |
| Communications | 37 late update; 38 unsupported RCA claim; 39 tenant disclosure; 40 wrong status audience; 41 missed next update; 42 vendor evidence unredacted |
| Recovery | 43 no rollback plan; 44 mitigation worsens SLO; 45 Pod-only verification; 46 alerts left muted; 47 no stability window; 48 no RCA owner |

## 13. Six integrated game days

1. SEV1 gateway and streaming outage with customer communications.
2. Agent reconciliation outage while serving data plane remains healthy.
3. GPU capacity shortage with cloud/vendor escalation.
4. vLLM latency/CUDA OOM regression after model deployment.
5. Security/identity alert requiring privacy-aware escalation.
6. Compound outage across GPU, gateway and missing telemetry with shift handoff.

Use approved staging faults or tabletop simulation for unsafe cases. Score detection, IC assignment, evidence, escalation, update cadence, rollback and synthetic SLO recovery.

## 14. Fourteen acceptance gates

G1 approved incident scope; G2 SEV and roles; G3 customer SLO; G4 UTC evidence; G5 cross-layer diagnosis; G6 escalation contacts; G7 security/privacy; G8 change control; G9 rollback; G10 internal updates; G11 external messaging approval; G12 handoff; G13 end-to-end recovery and alerts; G14 reviewer sign-off.

See the [companion lab](../labs/06-incident-response-escalation-communications-check.md). Formal technical/security review and live execution pending.

**Next:** Chapter 14.7 — Post-Incident Validation and Reliability Acceptance Runbook, the Part 14 capstone.
