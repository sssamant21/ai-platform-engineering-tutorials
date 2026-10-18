# Chapter 086 — LLM Incident Command, SEV and Escalation

**Part 11 — Incident Response, HA and Disaster Recovery · Chapter 086 of 100**  
**Status:** Published tutorial and offline incident-command lab; live game day, operational validation and independent review pending.

## Objectives
Operate a consistent LLM incident command system spanning AI gateways, GPU serving, retrieval/vector stores, tool brokers, model releases, data security and cloud infrastructure. Classify incidents, assign roles, communicate impact, make safe mitigation decisions and hand off into RCA/COA and reliability improvements.

## 1. Reference incident architecture
```text
signals: SLO burn / 429 / 5xx / TTFT / TPOT / GPU OOM / RAG / safety
                              |
                  alert routing + deduplication
                              |
                     incident commander (IC)
                              |
          +-------------------+-------------------+
          |                   |                   |
      ops lead           comms lead         security/privacy
          |                   |                   |
  gateway / model / GPU   stakeholders      breach assessment
  RAG / tools / platform  customer updates   evidence controls
          |
  mitigation -> validation -> recovery -> retrospective
```
A human IC coordinates decisions and accountability; automation may collect evidence and execute preapproved low-risk actions but must not silently authorize high-impact failover or data exposure.

## 2. Severity model
| SEV | Typical LLM service impact | Response expectation |
|---|---|---|
| SEV1 | widespread inference outage, confirmed cross-tenant data disclosure, uncontrolled high-impact spend or critical safety failure | immediate IC, security escalation when relevant, executive/customer communications |
| SEV2 | major tenant/service degradation, sustained SLO burn, material model output regression, GPU pool outage with limited fallback | dedicated IC, operations and stakeholder coordination |
| SEV3 | partial regional degradation, intermittent 429/5xx, noncritical retrieval failure or contained budget anomaly | on-call-led incident with time-boxed escalation |
| SEV4 | low-impact defect, alert noise or nonurgent follow-up | ticket and planned remediation |

These are example categories, not universal thresholds. Adopt organizational policy and regulatory notification requirements. A security incident may require separate handling even when availability is unaffected.

## 3. Detection and first five minutes
1. Confirm the alert using independently sourced telemetry and synthetic probes.
2. Establish affected region, model revision, tenants, request path and onset.
3. Check blast radius and error-budget consumption; classify provisional SEV.
4. Declare incident, assign IC and technical/comms leads, and start timeline.
5. Freeze risky changes, identify last known-good deployment and current rollback capability.
6. Check for sensitive-content exposure before copying traces, prompts or logs.
7. Establish update cadence and next decision checkpoint.
8. Begin low-risk containment only under approved runbooks.

## 4. LLM-specific incident dimensions
| Symptom | Evidence | Immediate safe options |
|---|---|---|
| 429 surge | quota reason, tenant fairness, retries | isolate abusive workload; validate budgets |
| 5xx/timeout | gateway errors, queue age, GPU saturation | shed load, revert recent change |
| streaming/SSE stalls | first-token latency, disconnects, proxies | inspect stream lifecycle and timeouts |
| GPU OOM | KV cache, concurrency, model size | reduce admission, restore known-good profile |
| degraded answer quality | offline/online eval, model/version, retrieval freshness | stop rollout, revert approved model |
| cross-tenant RAG leak | ACL evidence, cache scope, vector filters | disable path, security incident |
| tool abuse | action authorization, broker audit | revoke tool scope, quarantine workflow |
| spend spike | reservations, token usage, retry storms | enforce budget caps and reconcile |

## 5. Roles and RACI
| Responsibility | IC | Ops lead | Comms lead | Security/privacy |
|---|---|---|---|---|
| Severity and priorities | Accountable | Consulted | Informed | Consulted |
| Technical mitigation | Approves | Responsible | Informed | Consulted |
| Stakeholder updates | Accountable | Consulted | Responsible | Consulted |
| Data exposure response | Coordinates | Supports | Approved messaging | Responsible |
| Incident closure | Accountable | Validates | Notifies | Signs off when required |

Avoid role overload: for major incidents, the IC should not simultaneously troubleshoot GPU kernels, manage customer messaging and run the bridge.

## 6. Communications
A good status update states: incident ID, severity, affected capabilities, start time (UTC), customer-visible symptoms, known/unknown scope, mitigation in progress, next update time and owner. Do not speculate on root cause or share sensitive prompts, credentials, tenant IDs or patient data. Use a consistent cadence (e.g. every 15–30 minutes for high severity, adjusted by policy).

### Sample update
```text
INC-2026-086 | SEV2 | 14:10 UTC
Impact: elevated inference errors and time-to-first-token in one serving region.
Mitigation: gateway admission tightened; previous model revision being restored.
Next update: 14:25 UTC. Root cause remains under investigation.
```

## 7. Incident timeline and evidence
Record timestamps in UTC, alert source, declarations, role assignments, policy/model/infra versions, hypothesis, decisions, approvals, mitigation, customer updates, recovery verification and follow-up owners. Restrict evidence access; avoid raw user content. Preserve clock synchronization and correlation IDs.

## 8. Escalation policy
- Page model-serving/GPU on-call for OOM, queue or runtime faults.
- Page gateway/API on-call for admission, SSE, routing and 429/5xx.
- Page DBRE/search owner for vector retrieval, Kafka, Redis, PostgreSQL, Snowflake or Elasticsearch/OpenSearch dependencies.
- Engage security/privacy for suspicious access, prompt leakage, secret exposure or model supply-chain compromise.
- Engage FinOps for unbounded inference cost and billing anomalies.
- Engage vendor/cloud support for capacity or managed-service outages; retain internal IC ownership.

Escalate when a mitigation exceeds its timebox, SLO burn accelerates, blast radius expands or the required authority is missing.

## 9. Recovery criteria
Declare recovery only after sustained synthetic and real-user SLO checks: request success, p95/p99 latency, TTFT, TPOT, queue depth, streaming completion, model quality guardrails, GPU memory and cost burn. Validate both primary and fallback paths. Do not close a security/privacy incident without required specialist sign-off.

## 10. Production runbooks

### RB-086-A — Declare and establish command
Open incident bridge, assign IC/ops/comms, record UTC onset, classify SEV, page relevant owners and publish first update.

### RB-086-B — Mitigation and controlled rollback
Confirm change history, choose approved rollback target, validate tenant isolation and model policy, execute canary rollback and monitor SLOs.

### RB-086-C — Escalate unresolved incident
At checkpoint, summarize known facts, impact, failed mitigations, decision required and next owner. Escalate without losing timeline or command ownership.

### RB-086-D — Close, RCA and COA
Verify recovery, confirm stakeholder notifications, assemble sanitized timeline, distinguish contributing factors from root cause, assign corrective actions with owners and deadlines, and schedule follow-up drills.

## 11. Game-day scenarios
1. 429 storm from recursive agents and shared tenant quota.
2. 5xx and streaming stalls after model router change.
3. GPU OOM and KV cache pressure during burst traffic.
4. RAG cross-tenant exposure requiring security escalation.
5. Cost spike caused by retries and large output token limits.
6. Model quality regression after a seemingly healthy deployment.
7. Observability exporter failure while incident is active.
8. Regional serving outage requiring capacity-aware failover.

## 12. 100-point acceptance
| Domain | Points |
|---|---:|
| SEV classification and impact assessment | 20 |
| IC roles, escalation and communications | 20 |
| Evidence, timelines and privacy-safe handling | 20 |
| Mitigation, rollback and recovery verification | 20 |
| Game day, RCA/COA and follow-up ownership | 20 |
| **Total** | **100** |

Educational target >=90/100 with no critical omission in data exposure escalation or unsafe recovery. Live incident exercises and organizational approval remain pending.

## 13. Companion lab
[Chapter 086 — Executable LLM Incident Command, SEV and Escalation Lab](../labs/086-llm-incident-command-sev-escalation-check.md). Offline simulator tests severity assignment, escalation, timelines and closure gates; it is not a paging or incident-management integration.

## References
- [Google SRE Workbook — Incident Response](https://sre.google/workbook/incident-response/)
- [Google SRE Book — Managing Incidents](https://sre.google/sre-book/managing-incidents/)
- [NIST SP 800-61](https://csrc.nist.gov/pubs/sp/800/61/r3/final)
- [OpenTelemetry](https://opentelemetry.io/)

**Next:** Chapter 087 — Inference 429, 5xx, Timeout and SSE Incident Runbooks.
