# Chapter 022 — vLLM Security, Authentication and Network Isolation

**Part 04 — vLLM Administration and Operations | Chapter 022 of 100**

> **Status:** Written tutorial published. Live labs, technical/security review and operational acceptance pending. Verify security features, CLI flags and endpoint behavior against the pinned vLLM release.

## Learning objectives

Build a threat model for production LLM inference; design authentication, authorization, tenant quotas and network isolation; secure model artifacts, workloads, secrets and distributed workers; prevent unsafe exposure of API/debug/control-plane endpoints; implement auditability without prompt leakage; investigate abuse and recover safely.

## 1. Trust boundaries and threat model

```text
Untrusted clients / tenants
          │
     TLS + WAF + API gateway
     ├─ identity/authentication
     ├─ tenant authorization
     ├─ rate + token quotas
     ├─ request size / context limits
     └─ audit metadata (no raw prompts)
          │ restricted ingress
          ▼
  Kubernetes Service → vLLM API server
                         ├─ model/tokenizer runtime
                         ├─ GPU engine and workers
                         └─ cache / inference telemetry
          │
   workload identity + approved egress
          ▼
  model registry / artifact store
          │
  internal-only coordinator / NCCL / Ray if used
```

Treat the API gateway, vLLM runtime, model artifact store, Kubernetes control plane and distributed coordination network as separate trust zones. A private subnet is not sufficient authorization.

## 2. Security requirements

| Layer | Required controls | Validation |
|---|---|---|
| Client edge | TLS, authenticated identity, bounded requests | unauthorized access denied |
| Authorization | tenant/model/route permissions | cross-tenant access rejected |
| Abuse protection | rate, token, concurrency and cost quotas | bounded 429 behavior |
| Kubernetes | least privilege, restricted Pod security | admission/policy evidence |
| Network | default-deny with scoped allow rules | explicit connectivity tests |
| Secrets | approved secret manager and rotation | no secrets in manifests/logs |
| Model supply chain | immutable revision, checksum, license | provenance and approvals |
| Runtime | pinned image, no unapproved remote code | security review |
| Distributed workers | private authenticated membership | no public control ports |
| Observability | sanitized metrics/traces/audit | no prompt or token leakage |
| Recovery | rotation, containment, rollback | rehearsed tabletop |

## 3. Authentication versus authorization

**Authentication** establishes client identity. **Authorization** decides which tenant, model, route and quota that identity may use. A bearer API key accepted by a serving process is not a replacement for enterprise gateway identity, scoped permissions, rate limits and audit.

A typical gateway policy maps validated identities to tenant/model entitlements. Default-deny unknown models and cross-tenant access. For internal service-to-service calls, prefer approved workload identity and mTLS where supported by the mesh/gateway architecture.

Never place static bearer tokens in Git, image layers, command history, dashboards or URL query parameters.

## 4. vLLM API-key support and limitations

Some vLLM releases provide an API-key option for their HTTP serving interface. Confirm the exact option and endpoint coverage for your pinned version. Do not assume it protects health/metrics endpoints, distributed workers, debug interfaces or the model registry. Put enterprise authentication, authorization and abuse controls at an approved gateway; keep the backend unreachable except through intended service paths.

## 5. Kubernetes network isolation

An illustrative **default-deny ingress and egress** NetworkPolicy for an isolated serving namespace:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: APPROVED_NAMESPACE
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

This policy **will break traffic** until specific allow policies are applied. It is a design example, not an instruction to apply to production. NetworkPolicy enforcement requires a supporting CNI. Plan explicit DNS, approved gateway-to-serving ingress, observability scraping, model artifact access and distributed worker communication. For multi-node NCCL/Ray, validate required dynamic ports and network policies with the pinned architecture. Avoid blanket all-namespace egress allowances.

## 6. Example scoped ingress policy

Illustrative only; replace selectors with approved labels and verify namespace selectors match the actual gateway namespace:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-gateway-to-vllm
  namespace: APPROVED_NAMESPACE
spec:
  podSelector:
    matchLabels:
      app: vllm
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: APPROVED_GATEWAY_NAMESPACE
          podSelector:
            matchLabels:
              app: APPROVED_GATEWAY_POD_LABEL
      ports:
        - protocol: TCP
          port: 8000
```

Labels, namespace identity, Pod port and CNI behavior must be verified. This policy does not establish application-level authentication or TLS by itself.

## 7. Pod and workload hardening

Review non-root execution where supported by the serving image, read-only root filesystem where compatible with caches, dropped Linux capabilities, seccomp, controlled writable volumes, no hostPath/privileged Pods, dedicated ServiceAccount, explicit resource limits and GPU device-plugin integration. Validate GPU runtime requirements before enforcing policies; do not weaken namespace-wide security just to make a model start.

Avoid mounting cloud credentials as long-lived files. Use scoped workload identity and short-lived credentials with audit logs.

## 8. Model and tokenizer supply chain

Require approved model license, artifact checksum, pinned weights/tokenizer/template revisions, model card review, immutable image digest, vulnerability scanning, restricted model registry and safe artifact handling. Remote custom model code can execute inside the inference runtime: enabling it is a security exception requiring independent review, not a routine loader fix.

Treat model artifacts and prompt/response logs as potentially sensitive. Maintain artifact lineage and documented retention/deletion policies.

## 9. Multi-tenancy and cache isolation

Tenant boundaries should cover:
- model/route entitlement;
- prompt and response data;
- GPU admission and quota;
- prefix/KV cache sharing behavior;
- tracing/logging;
- per-tenant SLO and cost attribution.

Prefix caching may share computation across requests. Confirm cache-key semantics and isolation guarantees in the pinned runtime before enabling reuse across tenants. Do not infer isolation from model alias alone.

## 10. Abuse and denial-of-wallet

LLM inference is vulnerable to high-token prompts, long outputs, high concurrency, retries and persistent SSE connections. Apply layered limits:

1. authenticated per-tenant requests per interval;
2. maximum prompt and output tokens;
3. context and body-size limits;
4. concurrent in-flight requests and queue deadlines;
5. bounded retries with jitter and cancellation;
6. GPU-hour or spend ceilings and alerts;
7. overload 429 behavior that does not leak internal details.

Rate limits alone do not bound token consumption. Track cost per **valid completed** request, not just raw requests.

## 11. Privacy-preserving observability

Log metadata such as request ID, tenant class (pseudonymized), model alias, token counts, latency buckets, status/error category, cancellation and completion. Avoid raw prompts, generated responses, bearer tokens, cookies, model weights and unbounded user identifiers.

Audit security decisions (allowed/denied) with retention and access control. Scrub sensitive exception strings and tracing attributes; avoid enabling verbose request-body logging during incidents.

## 12. Read-only security checks

**[SAFE-READ]** with authorized Kubernetes access:

```bash
kubectl config current-context
kubectl get ns --show-labels
kubectl get networkpolicy -A
kubectl get serviceaccount -A
kubectl get ingress,svc -A -o wide
kubectl get pods -A -o wide
```

For an approved namespace:

```bash
kubectl -n MODEL_NAMESPACE describe networkpolicy
kubectl -n MODEL_NAMESPACE describe deployment MODEL_DEPLOYMENT
```

Do not export Kubernetes Secrets or dump environment variables containing credentials. Review gateway configuration through approved read-only change-management paths.

## 13. Incident response decision tree

```text
Potential vLLM security incident?
 ├─ unauthenticated access → gateway policy / exposed service
 ├─ cross-tenant model access → authorization rules
 ├─ public debug/control port → network containment
 ├─ leaked API token → rotate, revoke, audit
 ├─ prompt/response in logs → contain, purge per policy
 ├─ unusual token burn → quotas / retry storm
 ├─ unapproved model revision → supply chain freeze
 ├─ remote code execution risk → isolate workload
 ├─ suspicious egress → identity / network policy
 ├─ cache isolation concern → disable sharing by approval
 ├─ stale auth policy after rollout → GitOps drift
 └─ security telemetry missing → restore audit visibility
```

For suspected compromise, preserve evidence under incident policy and coordinate with security. Do not blindly delete resources or logs.

## 14. Security change and rollback

1. Identify trust boundaries and asset owners.
2. Record current gateway, workload, NetworkPolicy, ServiceAccount and model artifact provenance.
3. Obtain change approval and an explicit emergency recovery path.
4. Validate intended allowed and denied paths in isolated staging.
5. Verify health/metrics/control endpoints are not externally reachable.
6. Test tenant/model authorization, quota, cancellation and logging redaction.
7. Canary gateway/network policy changes with synthetic requests.
8. Abort on legitimate traffic outage, unauthorized access, missing audit or SLO burn.
9. Revert through the authoritative GitOps/security controller.
10. Verify both functional recovery and deny-by-default security posture.

## 15. Twenty-four production failure scenarios

| ID | Failure | First evidence |
|---|---|---|
| F01 | public unauthenticated API | exposure scan |
| F02 | API key bypasses gateway | path inventory |
| F03 | cross-tenant model access | authorization test |
| F04 | stale revoked token accepted | rotation test |
| F05 | bearer token in logs | privacy audit |
| F06 | secrets committed to Git | repository scan |
| F07 | broad ServiceAccount privileges | RBAC review |
| F08 | privileged GPU Pod | Pod security |
| F09 | unapproved hostPath mount | Pod spec |
| F10 | unrestricted egress | policy inventory |
| F11 | default-deny breaks model downloads | connectivity |
| F12 | DNS blocked by policy | resolution |
| F13 | gateway ingress selector mismatch | CNI policy |
| F14 | public Ray/coordinator port | exposure scan |
| F15 | NCCL ports overexposed | network review |
| F16 | mutable model artifact drift | checksum |
| F17 | remote model code execution risk | loader config |
| F18 | malicious tokenizer/template | artifact review |
| F19 | prompt/response captured in traces | trace audit |
| F20 | prefix cache isolation concern | cache review |
| F21 | tenant token-budget abuse | quota metrics |
| F22 | retry amplification/denial-of-wallet | GPU-hours |
| F23 | missing security audit events | log freshness |
| F24 | policy rollout causes outage | complete SSE SLI |

## 16. Sixteen operational runbooks

1. Validate exposed API and ingress routes.
2. Investigate authentication bypass.
3. Investigate tenant/model authorization.
4. Rotate and revoke compromised credentials.
5. Triage sensitive log/trace leakage.
6. Audit Kubernetes RBAC and ServiceAccounts.
7. Audit Pod security and GPU privileges.
8. Investigate NetworkPolicy deny/allow failures.
9. Restore DNS and approved model egress.
10. Secure distributed coordination ports.
11. Validate model artifact provenance.
12. Investigate remote code and tokenizer risk.
13. Audit prefix cache tenant isolation.
14. Contain quota abuse and GPU cost.
15. Execute approved security rollback.
16. Validate restored security posture and RCA.

## 17. Eight tabletop game days

1. Public ingress bypasses API authentication.
2. Tenant accesses another tenant's model alias.
3. API token leaks into a diagnostic log.
4. Default-deny policy blocks model artifact access.
5. Distributed coordinator port is exposed.
6. Unapproved model revision requests remote code.
7. Token-heavy retries trigger denial-of-wallet.
8. Security policy rollout passes health checks but breaks complete SSE.

Default **tabletop**; penetration tests and live fault injection require explicit security authorization.

## 18. Twenty acceptance gates

G01 threat model; G02 TLS/ingress; G03 authentication; G04 tenant authorization; G05 rate/token quotas; G06 Kubernetes RBAC; G07 Pod security; G08 NetworkPolicy enforcement; G09 DNS/approved egress; G10 distributed port isolation; G11 secret rotation; G12 model provenance; G13 remote code controls; G14 cache isolation; G15 privacy-preserving logs; G16 security audit; G17 SLO and FinOps; G18 safe change; G19 rollback; G20 independent security review.

See [companion lab](../labs/022-vllm-security-authentication-network-isolation-check.md). All gates remain pending until independently evidenced.

**Next: Chapter 023 — vLLM Metrics, Logging and Distributed Tracing.**
