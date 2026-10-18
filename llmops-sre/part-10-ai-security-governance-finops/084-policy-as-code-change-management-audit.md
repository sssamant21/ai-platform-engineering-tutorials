# Chapter 084 — Policy-as-Code, Change Management and Audit

**Part 10 — AI Security, Governance and FinOps · Chapter 084 of 100**  
**Status:** Chapter and offline lab published; live integration, independent technical/security review and production approval pending.

## Objectives
Establish version-controlled, testable policy for LLM inference, retrieval, tools, model supply chain, privacy and GPU spending. Design change approvals, separation of duties, policy rollout/rollback, exception expiry and tamper-evident audit trails.

## 1. Reference architecture
```text
policy Git repo -> PR checks -> unit/adversarial tests -> review approvals
                        |
                 signed release artifact
                        |
               policy distribution / GitOps
                        |
gateway PEP -> PDP <- retrieval PEP / tool broker / deployment admission
                        |
                 sanitized decision audit
                        |
                immutable log storage + SIEM
```
PEP = policy enforcement point; PDP = policy decision point. **The LLM is not a PDP.** Its prompts and generated responses are untrusted for authorization. Apply enforcement at each protected resource boundary.

## 2. Policy domains
| Domain | Example rule | Release-blocking regression |
|---|---|---|
| Identity | verified principal and tenant membership | forged tenant accepted |
| Inference | approved model and token ceilings | unauthenticated GPU admission |
| Retrieval | document ACL and tenant scope | cross-tenant document |
| Tools | explicit action/resource scope and approval | privileged tool bypass |
| Privacy | allowlisted telemetry attributes | raw prompt or secret export |
| Supply chain | trusted signer, digest, SBOM | unverified model deployed |
| FinOps | per-tenant budget and concurrency | unbounded spend |
| Infrastructure | workload identity, network egress | unrestricted secret access |

Policy rules should be declarative, independently testable and fail closed for high-risk actions. Do not use a regex prompt filter as a substitute for resource authorization.

## 3. Policy repository layout
```text
llm-policy/
  policies/
    inference.rego
    retrieval.rego
    tools.rego
    deployment.rego
  data/
    model-allowlist.json
    tenant-entitlements.json
  tests/
    inference_test.rego
    adversarial-cases.json
  change/
    change-template.md
    exception-register.md
  ci/
    validate.sh
```
Pin OPA version and trusted policy bundle digest. Protect branches, require code-owner review, use least-privilege CI identities, and sign approved bundles where supported.

## 4. Illustrative OPA policy
```rego
package llm.inference

import rego.v1

default allow := false

allow if {
  input.identity.verified == true
  input.identity.tenant == input.request.tenant
  input.request.model in data.approved_models
  input.request.input_tokens <= 8192
  input.request.output_tokens <= 1024
  input.request.input_tokens >= 0
  input.request.output_tokens >= 0
}
```
The gateway must construct `identity.verified` from a vetted identity verifier; clients must not supply it directly. For production, add strict schema/type validation, action/resource scopes, signed bundle trust, policy data integrity and deny-on-error behavior.

## 5. Change management workflow
1. Open a change request with purpose, affected tenants, environments and risk classification.
2. Record baseline policy version, owner, approvals, blast radius and rollback digest.
3. Run format/lint, unit, adversarial and schema tests in CI.
4. Evaluate privacy, security, reliability, cost and legal/license impact as applicable.
5. Require independent approval for high-risk policies; separate author, reviewer and deployer.
6. Produce a signed/immutable policy artifact and capture commit, digest and test evidence.
7. Roll out via staging, shadow evaluation (when safe), canary and gradual promotion.
8. Compare allow/deny changes, 401/403/429 rates, latency, tenant isolation and cost.
9. Revert to last approved digest on critical regression.
10. Close with sanitized audit evidence, exception expiry and post-change review.

## 6. Exceptions and break-glass
Exceptions must have owner, scope, business justification, compensating controls, approval, start/end time and review schedule. A break-glass action must be separately authenticated, time-bound, logged and reviewed. Never silently change `default deny` to `allow` during outages.

## 7. Audit evidence model
Record decision ID, UTC timestamp, policy bundle digest, action, resource reference, sanitized principal/tenant reference, decision, reason, change ticket and deployment revision. Do **not** log raw prompts, completions, bearer tokens or secrets. Restrict audit read access, enforce retention, clock synchronization and tamper-evident storage. Hash chaining can detect local record modification but does not alone prove trusted origin, completeness or secure anchoring.

## 8. CI/CD and test matrix
| Test | Expected |
|---|---|
| Unknown principal or model | deny |
| Cross-tenant resource | deny |
| Privileged tool without approval | deny |
| Missing/expired exception | deny |
| Unverified model provenance | deny |
| Budget exhausted | deny |
| Safe authorized inference | allow |
| PDP unavailable | fail closed or explicitly approved degraded policy |
| Malformed policy input | deny with sanitized error |
| Policy rollback | restore approved bundle and verify audit continuity |

## 9. Troubleshooting
| Symptom | Check | Response |
|---|---|---|
| sudden 403 surge | policy diff, identity claims, bundle version | canary rollback if regression |
| unexpected allow | data freshness, PEP bypass, default behavior | isolate route and escalate |
| policy rollout inconsistent | replicas, cache TTL, signed bundle digest | stop promotion, converge versions |
| audit gaps | exporter outage, queue and retention | protect evidence, restore audit path |
| expired exception persists | expiry enforcement, timezone | revoke exception, review decisions |
| OPA latency spike | policy complexity, data size, network | profile, optimize and protect SLO |

## 10. Production runbooks

### RB-084-A — Unauthorized policy decision
Contain affected PEP, preserve sanitized policy/input metadata, revoke inappropriate grants, identify affected tenants and restore approved policy. Security owner signs off before re-enable.

### RB-084-B — Failed policy rollout
Stop promotion, compare expected/observed policy digest, revert to approved bundle, run tenant isolation and safe-path smoke tests, and record change evidence.

### RB-084-C — Policy engine outage
Deny high-risk operations; use only approved, pre-tested fallback policy for low-risk traffic. Do not bypass authorization for availability.

### RB-084-D — Audit pipeline failure
Protect pending records, restore secure exporter, reconcile decision gaps, review retention and notify governance owner.

## 11. Acceptance rubric (100 points)
| Area | Points |
|---|---:|
| Policy inventory, least privilege and fail-closed rules | 20 |
| CI tests, adversarial cases and signed promotion | 20 |
| Change control, separation of duties and exceptions | 20 |
| Audit integrity, retention and access controls | 20 |
| Canary, rollback, incident runbooks and evidence | 20 |
| **Total** | **100** |

Critical release blockers: cross-tenant access, unapproved privileged tool action, unsigned/untrusted policy promotion, secret leakage or bypassable enforcement. Educational target >=90/100 with no critical failures.

## 12. Companion lab
[Chapter 084 — Executable Policy-as-Code, Change Management and Audit Lab](../labs/084-policy-as-code-change-management-audit-check.md). The offline Python evaluator exercises deterministic decisions, expiring exceptions and audit chaining. It is **not** a real OPA engine, cryptographically signed policy bundle or immutable audit service.

## References
- [Open Policy Agent](https://www.openpolicyagent.org/docs/latest/)
- [NIST SP 800-53](https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final)
- [SLSA Framework](https://slsa.dev/)
- [Kubernetes Admission Controllers](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)
- [OWASP API Security](https://owasp.org/API-Security/)

**Next:** Chapter 085 — Security, Governance and FinOps Acceptance Lab.
