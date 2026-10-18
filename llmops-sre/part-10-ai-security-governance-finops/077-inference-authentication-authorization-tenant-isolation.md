# Chapter 077 — Inference Authentication, Authorization and Tenant Isolation

**Part 10 — AI Security, Governance and FinOps · Chapter 077 of 100**  
**Status:** Chapter and companion lab published. Live execution and independent technical/security reviews pending.

## Objectives
Implement defense-in-depth identity verification, authorization, tenant-scoped inference admission, retrieval isolation, tool permissions, quotas, caching and audit controls for an enterprise LLM service. Validate that authenticated identity—not user-supplied prompt text or model output—determines access.

## 1. Architecture
```text
client identity (OIDC JWT / workload mTLS)
                 |
          API gateway / TLS
                 |
     token validation + key rotation
                 |
    policy decision point (PDP)
    principal + tenant + action + resource
                 |
    policy enforcement point (PEP)
                 |
   request admission / tenant quotas
                 |
     prompt assembly + RAG connector
           |             |
       model GPU     ACL-filtered retrieval
           |             |
     tool broker   tenant-scoped cache
           |
  scoped backend credentials / audited tool actions
```

**Trust boundaries:** network ingress, identity assertion, tenant context, retrieval authorization, model-generated tool requests, shared cache and observability export. The model is never an identity provider or a policy authority.

## 2. Authentication
- Human/user traffic: OIDC access tokens validated for **issuer, audience, signature, expiration, not-before and algorithm allowlist**. Validate key ID and rotate keys through a trusted JWKS endpoint; reject unsigned tokens and algorithm confusion.
- Service-to-service: mTLS and short-lived workload identity where available. Separate machine roles from user impersonation.
- API keys: only if approved; scope and rotate, hash at rest, never expose in logs, and avoid treating an API key as sufficient tenant authorization.
- Never accept an arbitrary `X-Tenant-ID` header as authority without binding it to verified identity and an explicit tenant membership decision.
- Enforce token revocation or sufficiently short lifetimes according to the service threat model.

## 3. Authorization
Use deny-by-default policy at every resource boundary:
```text
allow if:
  verified_principal
  AND principal belongs to requested tenant
  AND requested action is permitted by role/scope
  AND resource belongs to authorized tenant
  AND any required human approval is valid
  AND quota/admission policy permits request
otherwise deny and audit
```
RBAC controls coarse roles; ABAC adds tenant, environment, resource sensitivity and operation attributes. Sensitive tool actions should require explicit approval independent of the model. A system prompt is **not** an access-control mechanism.

## 4. Tenant isolation planes
| Plane | Isolation requirement | Example regression |
|---|---|---|
| Identity | tenant membership bound to principal | spoofed tenant header |
| Inference API | tenant-scoped quotas, routing and admission | noisy neighbor saturates queue |
| Retrieval | ACL filter enforced before retrieval results enter prompt | cross-tenant document returned |
| Prefix/KV cache | isolation or approved safe sharing policy | cached tenant content reused |
| Tool broker | per-action/resource authorization | model invokes privileged write |
| Storage/logs | tenant access controls, encryption, retention | PII leaked via traces |
| Model registry | signed model and runtime artifacts | unauthorized model revision loaded |

For strict isolation requirements, use dedicated workers, storage, network policies or accounts; logical tenant tags alone may be insufficient. Assess shared GPU memory reuse, batching, caches, metrics cardinality and timing side channels according to the data classification and threat model.

## 5. Quotas, fairness and availability
Enforce per-tenant request rate, input/output token budgets, concurrency, queue size and GPU admission. Token-based quotas are important because two requests with identical RPS can have vastly different prefill/decode cost. Return bounded `429`/retry guidance when admission fails. Use fair scheduling to protect smaller tenants from noisy neighbors. Monitor attempted/admitted/completed requests, token counts, rejected calls and p95 latency by low-cardinality tenant class without exporting sensitive IDs to general metrics.

## 6. Practical deployment pattern
1. Terminate TLS at approved gateway; authenticate request using trusted identity provider.
2. Validate token issuer/audience/signature/time and derive principal and allowed tenant memberships.
3. Verify requested tenant and resource authorization at API gateway and retrieval/tool layers.
4. Enforce quotas before GPU admission; apply bounded deadlines and queue caps.
5. Resolve retrieval documents with server-side tenant ACL filtering.
6. Validate every model-requested tool action in a deterministic broker; use scoped backend credentials.
7. Partition or disable sensitive caches; never use untrusted user-supplied cache namespace as authorization.
8. Emit sanitized allow/deny decisions with request correlation ID and policy version.
9. Test cross-tenant and noisy-neighbor cases under authorized staging conditions.
10. Rotate credentials, revoke access and audit policy changes under change control.

## 7. Example policy (illustrative)
```yaml
identity:
  issuer: https://idp.example.invalid/
  audience: llm-inference-api
  allowed_algorithms: [RS256]
authorization:
  default: deny
  tenant_source: validated_identity_membership
  enforce_retrieval_acl: true
  tool_broker_authorization: required
admission:
  max_input_tokens_per_request: 8192
  max_output_tokens_per_request: 1024
  max_concurrent_requests_per_tenant: 8
audit:
  log_raw_prompts: false
  log_credentials: false
```
This is a policy design sketch, **not** a directly deployable configuration. Thresholds are illustrative and must be selected from measured workloads.

## 8. Test plan
- Reject unsigned/expired/wrong-audience tokens, unknown issuer and revoked credentials.
- Deny a valid principal attempting to switch to a tenant without membership.
- Deny cross-tenant document retrieval and cache access.
- Deny unauthorized tool calls even when model output asks for them.
- Confirm quota isolation during synthetic noisy-neighbor load.
- Verify audit fields are present and sensitive content is absent.
- Test key rotation and failure mode when identity provider is unavailable.
- Validate session revocation, rollback and incident containment.

## 9. Troubleshooting
| Symptom | Diagnose | Response |
|---|---|---|
| All requests return 401 | issuer, JWKS, clock skew, audience | verify identity configuration; do not disable validation |
| Valid user gets 403 | membership, scope, resource ACL | inspect policy decision evidence |
| Cross-tenant RAG result | connector ACL, index filtering, cache key | isolate connector/cache; escalate |
| 429 only for one tenant | quota and token use | tune approved quota, investigate abuse |
| Tool action bypass | broker PEP, scope binding | disable tool route, rotate credentials |
| Latency spike for all tenants | shared queue, fair scheduling, GPU capacity | activate admission/fairness safeguards |
| Sensitive data in logs | exporter/sampling config | stop export, restrict evidence, incident response |

## 10. Production runbooks

### RB-077-A — Authentication outage
Inspect trusted issuer/JWKS reachability, key rotation, token validation errors and clock skew. **Fail closed** on unverifiable identity; use approved fallback only if independently authenticated. Restore and verify authorized and denied requests.

### RB-077-B — Cross-tenant isolation alert
Disable affected retrieval route or cache under incident authority, preserve sanitized evidence, evaluate exposure scope, rotate credentials if necessary and require security sign-off before re-enable.

### RB-077-C — Noisy-neighbor overload
Inspect per-tenant admission, token budgets, queue age, GPU saturation and rejected requests. Apply approved fairness/quota policies; do not bypass isolation to improve aggregate throughput.

### RB-077-D — Privileged tool-call violation
Freeze tool broker action, revoke scoped credentials, review model-generated request versus policy decision and verify all privileged writes require independent authorization.

## 11. Acceptance rubric (100 points)
| Domain | Points |
|---|---:|
| Token/workload identity validation | 20 |
| RBAC/ABAC and resource authorization | 20 |
| Retrieval, cache and tool isolation | 20 |
| Quotas, fairness and observability | 20 |
| Adversarial tests, runbooks and audit | 20 |
| **Total** | **100** |

Any cross-tenant access, privileged tool bypass, secret exposure or unauthenticated inference access blocks release regardless of score. Record security review and live staging results separately.

## 12. Companion lab
[Chapter 077 — Executable Inference Authentication, Authorization and Tenant Isolation Lab](../labs/077-inference-authentication-authorization-tenant-isolation-check.md).

## References
- [OWASP API Security Top 10](https://owasp.org/API-Security/)
- [OAuth 2.0 Security Best Current Practice (RFC 9700)](https://www.rfc-editor.org/rfc/rfc9700.html)
- [JWT Best Current Practices (RFC 8725)](https://www.rfc-editor.org/rfc/rfc8725.html)
- [NIST SP 800-207 Zero Trust Architecture](https://csrc.nist.gov/pubs/sp/800/207/final)
- [Open Policy Agent](https://www.openpolicyagent.org/docs/latest/)

**Next:** Chapter 078 — Prompt Injection, Data Leakage and Model Abuse.
