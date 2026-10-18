# Chapter 085 — Security, Governance and FinOps Acceptance Lab

**Part 10 — AI Security, Governance and FinOps · Chapter 085 of 100 · Capstone**  
**Status:** Chapter and executable offline acceptance harness published. Live staging execution, independent security/privacy/FinOps review and production sign-off pending.

## Mission
Demonstrate end-to-end LLM serving governance across Chapters 076–084. An enterprise inference release is accepted only when identity, retrieval and tool authorization, prompt-injection containment, workload secrets, artifact provenance, telemetry privacy, gateway budgets, GPU economics, policy changes and auditability are verified together.

## 1. Capstone architecture
```text
client -> OIDC/JWT gateway -> admission & tenant budgets -> model router
                 |                  |                   |
            policy PDP         usage ledger      signed model registry
                 |                                      |
             retrieval ACL -> vector DB          GPU inference
                 |                                      |
             tool broker -------------------------------+
                 |
        privacy-safe telemetry -> secure audit / FinOps / SLO dashboard
                 |
          GitOps policy + change approval + rollback
```
**Trust boundaries:** clients cannot set verified tenant identity; model output cannot grant tool access; cross-tenant retrieval is denied at resource access; deployment uses signed pinned artifacts; sensitive telemetry is filtered before export; gateway admission and budgets cannot be bypassed.

## 2. Prerequisite chapter controls
| Chapter | Acceptance evidence |
|---|---|
| 076 — AI Security Foundations | threat model, assets, abuse cases |
| 077 — Inference Authentication, Authorization and Tenant Isolation | verified identity, deny cross-tenant |
| 078 — Prompt Injection, Data Leakage and Model Abuse | tool authorization, leakage drills |
| 079 — Secrets, Workload Identity and Network Policies | federation, secret lifecycle, enforced segmentation |
| 080 — Model Artifact Supply Chain, SBOM and Provenance | trusted digest, signatures, SBOM and license |
| 081 — Privacy-Safe Telemetry and Data Retention | safe schema, TTL and deletion workflow |
| 082 — AI Gateway Rate Limits and Denial-of-Wallet Protection | atomic admission, token budgets, ledger |
| 083 — GPU FinOps, Budget Alerts and Rightsizing | reconciled spend, rightsizing guardrails |
| 084 — Policy-as-Code, Change Management and Audit | versioned policy, approval, tamper-evident audit |

## 3. Staging acceptance plan
### Phase A — Environment and evidence
Use a disposable nonproduction environment, synthetic prompts and tenants, approved identity provider, model registry, secret manager, GPU or representative inference pool, network policy enforcement and controlled cost ledger. Record cluster, model and policy versions. Avoid real credentials in test output.

### Phase B — Identity and data boundaries
- Authenticate verified users; reject forged tenant headers, missing scopes and expired tokens.
- Enforce retrieval ACL at vector/database boundary and tool scopes at broker.
- Inject synthetic prompt-injection instructions; verify they cannot change resource authorization.
- Test tenant isolation in caches, telemetry and error responses.

### Phase C — Infrastructure and artifact trust
- Validate workload federation subject/audience, secret access paths and credential rotation.
- Apply staged default-deny network policies with minimal DNS, gateway, retrieval and telemetry allowlists.
- Verify model/tokenizer/image digests, signed provenance, SBOM coverage and revocation handling.

### Phase D — Privacy, budget and reliability
- Run synthetic canary leakage tests across logs, traces, metrics and exporters.
- Verify retention, deletion propagation and role-limited audit access.
- Stress gateway RPS, token/concurrency limits and atomic budget reservations across replicas.
- Reconcile provider/GPU metering, test alert thresholds and validate N-1 rightsizing SLOs.

### Phase E — Policy change and recovery
- Make an approved policy change via PR with independent review, signed bundle and canary.
- Inject denial/regression, restore prior trusted policy digest and confirm audit continuity.
- Capture incident timeline, escalation, remediation, rollback and ownership.

## 4. Failure injection matrix
| ID | Scenario | Required result |
|---|---|---|
| F01 | forged tenant identity | deny |
| F02 | cross-tenant retrieval | deny |
| F03 | model-generated privileged tool instruction | deny |
| F04 | expired workload token | deny |
| F05 | unapproved network egress | block |
| F06 | model/tokenizer digest mismatch | block rollout |
| F07 | missing provenance signature | block rollout |
| F08 | raw prompt in telemetry | block export |
| F09 | budget exhausted | reject admission |
| F10 | concurrent retry storm | bounded spend and queues |
| F11 | GPU rightsizing harms p95/SLO | rollback |
| F12 | unapproved policy change | reject promotion |
| F13 | audit record tampering | detect and investigate |
| F14 | secret leak in logging | contain, rotate, notify |
| F15 | model artifact revoked | quarantine and rollback |

## 5. Operational scorecard (100 points)
| Domain | Points |
|---|---:|
| Threat model, authentication and tenant isolation | 15 |
| Prompt-injection, tool and retrieval security | 15 |
| Secrets, workload identity and network isolation | 15 |
| Artifact supply chain, signatures and provenance | 10 |
| Privacy, retention and telemetry | 10 |
| Gateway quotas, budget ledger and FinOps | 15 |
| Policy-as-code, audit, change control and rollback | 10 |
| Incident runbooks, SLO and evidence | 10 |
| **Total** | **100** |

**Pass threshold:** >=90/100 **and** all critical security/privacy/tenant/cost gates passing. A numerical score cannot override a critical gate failure. The offline lab is a teaching simulator and cannot certify production.

## 6. Production runbooks

### RB-085-A — Unauthorized cross-tenant access
Disable affected path, revoke credentials if compromised, preserve restricted sanitized evidence, identify affected resources, fix authorization at data/tool boundary and obtain security approval before restoring service.

### RB-085-B — Untrusted artifact or secret compromise
Block promotion, quarantine affected model/image, revoke leaked secrets/signing identities, verify trusted provenance and restore a known-good signed release.

### RB-085-C — Privacy disclosure
Stop telemetry exporter, restrict access, engage privacy/security incident process, rotate exposed credentials, determine retention/deletion obligations and validate clean export with synthetic canaries.

### RB-085-D — Budget burn or GPU saturation
Apply gateway admission caps, investigate retries/long contexts, reconcile ledger and GPU billing, preserve availability/SLO guardrails and roll back unsafe rightsizing.

### RB-085-E — Policy rollout regression
Stop promotion, compare approved vs deployed policy digests, revert through GitOps, test tenant isolation and maintain tamper-evident audit evidence.

## 7. Required evidence package
- Signed-off threat model, risk register and policy versions.
- Test inventory and reproducible commands; real staging pass/fail evidence.
- Sanitized audit, denial, latency, throughput, budget and alert records.
- Artifact digest, SBOM, verified provenance and model license approval.
- Secret rotation, NetworkPolicy enforcement and recovery drills.
- Change ticket, independent reviewers, rollback outcome and exception expiry.
- Explicit residual risk and accountable acceptance owner.

## 8. Companion offline acceptance harness
[Chapter 085 — Security, Governance and FinOps Acceptance Lab](../labs/085-security-governance-finops-acceptance-lab-check.md) contains a runnable Python cross-domain gate evaluator and pytest suite. It deliberately does **not** perform real JWT, cryptographic signature, CNI, telemetry deletion or cloud billing validation.

## References
- [NIST AI RMF](https://www.nist.gov/itl/ai-risk-management-framework)
- [OWASP LLM Top 10](https://genai.owasp.org/llm-top-10/)
- [Open Policy Agent](https://www.openpolicyagent.org/)
- [SLSA](https://slsa.dev/)
- [OpenTelemetry](https://opentelemetry.io/)
- [FinOps Foundation](https://www.finops.org/framework/)

**Part 10 complete on publication (10/10 chapters). Next:** Chapter 086 — LLM Incident Command, SEV and Escalation, beginning Part 11 — Incident Response, HA and Disaster Recovery.
