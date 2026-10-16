# Chapter 15.4 — Hands-on Lab: Networking, Security and GitOps Delivery

**Part 15 — End-to-End Production Project | Staging-only | Live execution pending**

## Goal

Validate private networking, DNS/TLS, gateway authentication, tenant isolation, Kubernetes NetworkPolicy, secrets, GitOps/TrueFoundry ownership, CI policy gates, progressive delivery and rollback for a GPU vLLM inference service.

**Safety:** [SAFE-READ] authorized inspection. [LAB-MUTATE] staging with approval and rollback. [CHANGE-CONTROL] production network/identity/policy/traffic. Never expose credentials, PHI, private prompts or model data.

## Lab 0 — Inventory and ownership

```bash
kubectl config current-context
kubectl get namespaces
kubectl get networkpolicies -A
kubectl get ingress,svc,endpointslices -A
kubectl get serviceaccounts,roles,rolebindings -A
kubectl get events -A --sort-by=.lastTimestamp
```

Record approved cluster/namespace, TrueFoundry agent, gateway controller, CNI NetworkPolicy enforcement, GitOps ownership, DNS/TLS, workload identity, cost cap and reviewers.

## Lab 1 — Network and TLS tests

From an approved staging diagnostic client, test DNS resolution, TLS certificate chain/hostname, allowed gateway route, blocked unapproved egress and required artifact/control-plane endpoints. Confirm ingress and egress policies actually enforce; do not apply a default-deny policy before reviewing allow rules.

## Lab 2 — Identity and tenant isolation

Use synthetic authorized/unauthorized requests to verify 401/403, tenant routing, service account scope, workload identity, model artifact ACL and secret references. Redact headers/tokens from output.

## Lab 3 — CI and GitOps release

Review a staging PR with pinned image/model digest, policy-as-code, secret scan, provenance, IaC plan and approval. Validate GitOps ownership versus TrueFoundry controller and prevent competing writers. Record expected and observed revision.

## Lab 4 — Streaming and rollback

Exercise synthetic SSE completion, gateway timeout, graceful drain and canary SLO. Trigger a safe staging-only bad revision or tabletop scenario, then roll back the compatible model/image/tokenizer through authoritative TrueFoundry desired state.

## Lab 5 — Six game days

1. Deny-all policy and blocked model egress.
2. TLS rotation/expiry tabletop.
3. Unauthorized inference and tenant isolation.
4. GitOps/TrueFoundry ownership conflict.
5. Canary TTFT/SSE regression and rollback.
6. CI supply-chain rejection with emergency approval policy.

Capture UTC timeline, approval, security evidence, customer-path SLO, mitigation, rollback, GPU-hours, executed/tabletop and reviewer.

## Forty-eight scenario checklist

01 DNS; 02 route; 03 CNI; 04 deny-all; 05 artifact egress; 06 agent egress; 07 cert; 08 SNI; 09 trust; 10 401; 11 403; 12 tenant; 13 buffering; 14 SSE; 15 timeout; 16 drain; 17 retries; 18 context; 19 CI role; 20 agent RBAC; 21 workload identity; 22 secret; 23 rotation; 24 break-glass; 25 controllers; 26 prune; 27 sync loop; 28 drift; 29 overlay; 30 promotion; 31 mutable image; 32 unsigned; 33 model; 34 SBOM; 35 scanner; 36 gate bypass; 37 GPU surge; 38 canary auth; 39 TTFT; 40 TPOT; 41 rollback; 42 tokenizer; 43 metrics; 44 alerts; 45 audit; 46 cost; 47 approval; 48 sign-off.

## Fourteen acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G1 | Network/trust diagram | Pending |
| G2 | DNS/private egress | Pending |
| G3 | CNI policy enforcement | Pending |
| G4 | TLS and renewal | Pending |
| G5 | Gateway auth/tenant isolation | Pending |
| G6 | Complete SSE streaming | Pending |
| G7 | Least-privilege IAM/RBAC | Pending |
| G8 | Secrets and rotation | Pending |
| G9 | CI provenance/policy gates | Pending |
| G10 | GitOps/TrueFoundry ownership | Pending |
| G11 | Staging promotion and approval | Pending |
| G12 | Canary and rollback | Pending |
| G13 | SLO/telemetry/GPU cost | Pending |
| G14 | Security/SRE sign-off | Pending |

## Evidence template

```text
Lab ID / UTC / change approval:
Cluster/namespace/CNI/gateway:
DNS/TLS/egress results:
Auth, tenant isolation, secret references:
CI commit, policy, SBOM/provenance:
GitOps owner vs TrueFoundry owner:
Model/image/tokenizer revision:
Synthetic SSE/TTFT/TPOT/HTTP:
Canary, rollback and restored SLO:
GPU-hours, cleanup, reviewer:
```

## Cleanup

Restore approved staging network policies and routes, remove disposable synthetic clients, reconcile authoritative desired state, confirm alerts and budgets, and revoke temporary privileges. Leave gates pending until actual review.

**Next:** Chapter 15.5 — Observability, SLOs and Load/Capacity Validation.
