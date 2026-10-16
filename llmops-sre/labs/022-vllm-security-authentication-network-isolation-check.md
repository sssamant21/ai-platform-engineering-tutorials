# Chapter 022 — Hands-On Lab: vLLM Security, Authentication and Network Isolation

**Part 04 | Chapter 022 | Published lab plan; live execution pending**

## Goal

Produce a production inference threat model, verify gateway auth and tenant isolation, review Kubernetes network policies and workload identity, secure model supply chain and distributed coordination, and validate security controls without exposing secrets or customer data.

## Safety

**[SAFE-READ]** approved Kubernetes metadata and sanitized configuration. **[LAB-MUTATE]** isolated staging tests with synthetic identities, models and traffic. **[CHANGE-CONTROL]** production gateway, network policy, secrets, model artifact, RBAC or distributed ports. No unauthorized penetration tests, PHI, customer prompts or secret dumps.

## Lab 0 — Authorization and context

Record UTC, cluster/context, namespace, gateway, model alias, vLLM image/version, identity owner, CNI, artifact store, GPU SKU, approval, rollback and security reviewer.

```bash
kubectl config current-context
kubectl get ns --show-labels
kubectl get networkpolicy -A
kubectl get serviceaccount -A
kubectl get ingress,svc -A -o wide
```

## Lab 1 — Trust boundary diagram [DESIGN]

Map client → TLS gateway → tenant authorization → Service → vLLM → GPU workers → model artifact store → internal distributed coordinator. Annotate identities, network paths, data classes, logging and control ownership.

## Lab 2 — Authentication and authorization [DESIGN]

Create a synthetic test matrix:

| Identity | Model | Expected |
|---|---|---|
| no token | permitted alias | denied |
| invalid token | permitted alias | denied |
| tenant A | tenant A alias | allowed |
| tenant A | tenant B alias | denied |
| tenant A | unsupported alias | denied |
| revoked token | tenant A alias | denied |

Do not run against production without authorization. Do not expose credentials in screenshots or logs.

## Lab 3 — Kubernetes NetworkPolicy [DESIGN]

Review the chapter's illustrative default-deny and gateway-ingress examples. Document necessary DNS, model registry egress, metrics scraping, gateway-to-serving ingress and distributed worker flows. Confirm the CNI enforces policies; do not apply a default-deny policy without staged allow rules and recovery.

## Lab 4 — Workload hardening [SAFE-READ]

For an approved namespace/deployment:

```bash
kubectl -n MODEL_NAMESPACE describe deployment MODEL_DEPLOYMENT
kubectl -n MODEL_NAMESPACE describe networkpolicy
```

Review Pod security, ServiceAccount, privilege, volumes, image digest, workload identity and resource limits. Do not print Kubernetes Secret values or container environment.

## Lab 5 — Model supply chain and privacy [DESIGN]

Record model license, revision, checksum, tokenizer/chat template, artifact access, approved remote-code policy, image signature and log/tracing redaction. Verify that prompt text, generated responses and bearer tokens are excluded from standard telemetry.

## Lab 6 — Abuse and denial-of-wallet [DESIGN]

Define per-tenant request, prompt-token, output-token, concurrency, queue and GPU cost ceilings. Simulate (tabletop) a retry storm with long output requests. Confirm 429 policy, bounded retries and complete-stream success accounting.

## Lab 7 — Eight tabletop game days

01 auth bypass; 02 cross-tenant access; 03 token log leak; 04 model egress blocked; 05 coordinator exposure; 06 remote code risk; 07 denial-of-wallet; 08 policy rollout outage.

## Twenty-four failure checklist

F01 public API; F02 gateway bypass; F03 cross-tenant; F04 revoked token; F05 token logs; F06 Git secret; F07 RBAC; F08 privileged Pod; F09 hostPath; F10 egress; F11 model download; F12 DNS; F13 gateway selector; F14 coordinator; F15 NCCL ports; F16 model drift; F17 remote code; F18 tokenizer; F19 trace leak; F20 cache isolation; F21 token abuse; F22 retry amplification; F23 audit; F24 policy outage.

## Twenty acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | Threat model | Pending |
| G02 | TLS and ingress | Pending |
| G03 | Authentication | Pending |
| G04 | Tenant authorization | Pending |
| G05 | Request/token quotas | Pending |
| G06 | Kubernetes RBAC | Pending |
| G07 | Pod security | Pending |
| G08 | NetworkPolicy enforcement | Pending |
| G09 | DNS/approved egress | Pending |
| G10 | Distributed port isolation | Pending |
| G11 | Secret rotation | Pending |
| G12 | Model provenance | Pending |
| G13 | Remote code controls | Pending |
| G14 | Cache isolation | Pending |
| G15 | Privacy-safe telemetry | Pending |
| G16 | Audit evidence | Pending |
| G17 | SLO and GPU FinOps | Pending |
| G18 | Safe change plan | Pending |
| G19 | Rollback | Pending |
| G20 | Independent security review | Pending |

## Evidence template

```text
UTC/context/approval:
Gateway/TLS/auth:
Tenant authorization matrix:
Kubernetes RBAC/Pod security:
CNI/NetworkPolicy/egress:
Distributed coordinator isolation:
Model/image/tokenizer provenance:
Secrets/privacy/audit:
Quotas/GPU cost/complete SSE:
Game-day mode/findings:
Rollback/reviewer/gates:
```

## Cleanup

Read-only/tabletop work creates no resources. For approved staging, revoke synthetic tokens, stop synthetic traffic, remove only owned test policies/resources through the authoritative controller and reconcile GPU-hours. All gates remain pending until independently reviewed.

**Next: Chapter 023 — vLLM Metrics, Logging and Distributed Tracing.**
