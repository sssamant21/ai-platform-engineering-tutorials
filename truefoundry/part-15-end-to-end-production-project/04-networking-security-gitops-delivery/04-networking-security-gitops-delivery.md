# Chapter 15.4 — Networking, Security and GitOps Delivery

**Part 15 — End-to-End Production Project | Capstone 4 of 7**

## Mission

Deliver a secure, reproducible and auditable production inference release path for the TrueFoundry-managed vLLM service from Chapters 15.1–15.3. Integrate private networking, DNS, TLS, gateway authentication, tenant isolation, Kubernetes policies, workload identity, GitOps/IaC review, software supply-chain controls, progressive rollout and rollback.

**[SAFE-READ]** approved cluster and repository inspection. **[LAB-MUTATE]** staging-only configuration with approved change and GPU spend cap. **[CHANGE-CONTROL]** production network, DNS, certificates, IAM, cluster agent, model rollout and traffic. No production credentials, prompts, PHI, secrets or private model data in Git or incident records.

## 1. Target delivery and trust architecture

```text
Developer PR → CI lint/tests/security/policy → signed/pinned artifacts
       │                │
       │           reviewed IaC plan
       ▼                ▼
 GitOps desired state → approved TrueFoundry deployment workflow
       │                │
       └──────────┬─────┘
                  ▼
      Kubernetes GPU namespace
       ├─ admission, RBAC, workload identity
       ├─ NetworkPolicy and cloud firewall/egress
       ├─ vLLM model + approved secret references
       └─ Service → authenticated gateway → TLS/DNS
                          │
               authorized synthetic clients
                          │
       metrics / logs / traces / audit / SLO / pager
```

**One authoritative deployment owner:** define whether GitOps manages platform declarations, Kubernetes resources or both, and where TrueFoundry remains the supported controller. Do not have Argo CD/Flux and TrueFoundry continuously overwrite the same fields. Identify ownership, drift resolution, rollback and change audit for every resource.

## 2. Network and endpoint design

- **Private routing:** VPC/VNet, subnets, route tables, DNS resolvers, NAT/private endpoints, network egress and cloud load balancer.
- **Ingress:** choose approved Ingress or Gateway API controller, TLS termination, identity-aware gateway, rate limits, timeout policy and streaming behavior.
- **Egress:** explicitly allow only required TrueFoundry control plane, registries, model artifact storage, observability and identity endpoints; discover current provider-specific destinations.
- **DNS:** validate record ownership, TTL, split-horizon/private DNS, resolution from workload namespace and client path.
- **TLS:** certificates, trust chains, SNI, rotation, renewal alerting, mTLS where required, no `--insecure` bypasses.
- **Streaming:** SSE buffering, idle/read timeouts, connection drain, retry semantics and end-of-stream validation.

Cloud security groups/firewalls and Kubernetes NetworkPolicy both matter; a NetworkPolicy manifest is ineffective if the installed CNI does not enforce it.

## 3. Trust boundary and authorization matrix

| Actor | Identity | Minimum access |
|---|---|---|
| CI runner | short-lived federated identity | build/push approved artifact; no cluster-admin |
| IaC pipeline | scoped cloud role | plan/apply approved environment resources |
| GitOps controller | namespace-scoped service account | approved Kubernetes resource kinds |
| TrueFoundry agent | vendor-supported scoped permissions | reconcile authorized platform resources |
| Model Pod | workload identity | read approved model prefix/secrets |
| Inference client | gateway identity/token | approved model route and tenant |
| SRE on-call | audited human role | read diagnostics; elevated actions by approval |
| Security reviewer | audit/read role | policy, secrets metadata, provenance |

Document separation of duties, break-glass expiry and audit evidence. Never use a cluster-admin token for normal CI/CD.

## 4. GitOps repository layout

```text
platform/
  infrastructure/
    modules/network/
    modules/cluster/
    modules/gpu-pool/
    environments/staging/
    environments/production/
  kubernetes/
    base/
      namespaces/
      networkpolicy/
      serviceaccounts/
      gateway/
      monitoring/
    overlays/staging/
    overlays/production/
  truefoundry/
    models/
    environments/staging/
    environments/production/
  policies/
    admission/
    ci/
  docs/
    adr/
    runbooks/
    evidence/
```

Pin module and image versions. Store only encrypted/approved secret references. Production promotion must be a reviewed immutable artifact reference change, not rebuilding an untraceable mutable tag.

## 5. NetworkPolicy example (review CNI and DNS first)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: inference-default-deny
  namespace: model-staging
spec:
  podSelector:
    matchLabels:
      app: vllm
  policyTypes: [Ingress, Egress]
```

This is a **deny-all starting point** for selected Pods and will break DNS, model downloads, telemetry, gateway traffic and control-plane dependencies until approved allow rules exist. Do not apply to production without staged connectivity tests, verified CNI enforcement, exception inventory and rollback. Ingress and egress policies are additive; review other policies that may already allow traffic.

## 6. CI/CD and supply-chain gates

1. Static YAML/Kustomize/Helm and IaC validation.
2. Policy-as-code for public ingress, privileged Pods, RBAC, egress, resource quotas and image digests.
3. Secret scanning and dependency/vulnerability checks with approved severity policy.
4. SBOM, artifact signing/attestation and provenance verification.
5. Staging deployment and synthetic inference/SSE tests.
6. Model/runtime compatibility and GPU fit checks.
7. SLO/error-budget and cost gates.
8. Manual/authorized production approval, immutable promotion and audit.
9. Canary/blue-green/rolling release as supported; customer-path verification.
10. Automatic abort/rollback policy **only if implemented and tested**, otherwise explicit on-call runbook.

## 7. GitOps versus TrueFoundry reconciliation

Build a resource ownership table: namespace/RBAC/NetworkPolicy/Gateway usually managed by platform GitOps; TrueFoundry model deployment specifications and controller-managed children remain under their authoritative workflow. Verify actual platform integration and documented supported options. Never force-sync, prune or manually patch TrueFoundry-managed Deployments to bypass reconciliation.

Drift triage:
```text
Unexpected state?
 ├─ Desired Git differs → review PR and promotion provenance
 ├─ TrueFoundry desired differs → inspect approved deployment revision
 ├─ Kubernetes differs → controller reconciliation, admission, RBAC
 ├─ GitOps/TrueFoundry conflict → stop competing writers, clarify ownership
 └─ Emergency patch → record change, restore authoritative source
```

## 8. Secure gateway and SSE acceptance

Test unauthorized request rejection, authorized synthetic request success, tenant routing, TLS certificate validity, rate limits, request size/context guardrails, full SSE termination, proxy timeout, client cancellation and graceful rollout drain. Measure end-to-end p95 TTFT/TPOT, 429/5xx and completion. An initial HTTP 200 is insufficient.

## 9. Secrets and certificate rotation

Use cloud workload identity and approved external secrets provider. Validate rotation in staging with overlapping validity windows, revocation, rollback and audit. Do not expose secrets in pipeline output, GitOps diffs, `kubectl describe`, trace attributes or vendor support bundles. Define certificate renewal owner, alerts and emergency recovery.

## 10. Progressive release and rollback

Record image digest, model checksum, tokenizer, vLLM/CUDA compatibility, Git commit, TrueFoundry deployment revision, gateway route and GPU capacity. Ensure old/new replica overlap fits GPU quota and zone placement. Abort on authentication failures, model contract drift, SSE truncation, TTFT/TPOT regression, GPU OOM, cost breach or telemetry gaps. Roll back through authoritative desired state and verify full customer-path SLO and restored security policy.

## 11. Troubleshooting decision tree

```text
Release or inference access failure?
 ├─ CI policy denied → inspect evidence, do not bypass gate
 ├─ GitOps drift → ownership and sync/prune policy
 ├─ Agent reconcile denied → TrueFoundry role/TLS/egress
 ├─ DNS/TLS fail → resolver, SNI, cert, trust chain
 ├─ 401/403 → gateway identity, tenant policy, token expiry
 ├─ 502/503/504 → ingress, Service/EndpointSlice, readiness
 ├─ Model download fail → egress/identity/artifact ACL
 ├─ SSE truncated → proxy buffering, timeout, drain
 └─ Canary SLO fail → halt promotion, compatible rollback
```

## 12. Eighteen operational runbooks

1. Network inventory and trust boundaries.
2. DNS/private routing.
3. TLS/certificate lifecycle.
4. Gateway auth/rate limits.
5. SSE proxy and drain.
6. CNI NetworkPolicy enforcement.
7. Model artifact egress.
8. Workload identity and RBAC.
9. Secret references/rotation.
10. CI security gates.
11. Image/model provenance.
12. GitOps ownership and drift.
13. TrueFoundry reconciliation conflict.
14. Staging promotion.
15. Production approval.
16. Canary and rollback.
17. SLO/security/FinOps validation.
18. Audit and incident handoff.

## 13. Forty-eight failure scenarios

| Domain | Scenarios |
|---|---|
| Network | 01 DNS fail; 02 private route fail; 03 CNI policy unenforced; 04 deny-all outage; 05 model egress blocked; 06 control-plane egress blocked |
| TLS/gateway | 07 cert expired; 08 wrong SNI; 09 trust chain; 10 gateway 401; 11 gateway 403; 12 tenant route leak |
| Streaming | 13 proxy buffering; 14 SSE truncation; 15 read timeout; 16 drain interrupt; 17 retry amplification; 18 context size bypass |
| Identity | 19 CI role overprivileged; 20 agent RBAC denied; 21 workload identity denied; 22 secret leaked; 23 rotation outage; 24 break-glass not revoked |
| GitOps | 25 competing controllers; 26 forced prune; 27 sync loop; 28 unmanaged drift; 29 wrong overlay; 30 unreviewed prod promotion |
| Supply chain | 31 mutable image; 32 unsigned artifact; 33 untrusted model; 34 SBOM absent; 35 secret scanner fail; 36 vulnerability gate bypass |
| Release | 37 GPU surge quota; 38 canary auth regression; 39 TTFT regression; 40 TPOT regression; 41 rollback unavailable; 42 incompatible tokenizer |
| Operations | 43 metrics stale; 44 alerts muted; 45 audit gap; 46 cost spike; 47 no change approval; 48 no security sign-off |

## 14. Six staging game days

1. Deny-all policy breaks model egress; recover with reviewed allow rules.
2. TLS certificate expiry/rotation tabletop.
3. Gateway auth and cross-tenant isolation negative tests.
4. Competing GitOps and TrueFoundry reconciliation.
5. Canary SSE/TTFT regression and rollback.
6. Supply-chain policy rejection and emergency release governance.

Record UTC, approved scope, expected/actual, customer SLO, security evidence, mitigation, rollback, GPU-hours and reviewer.

## 15. Fourteen acceptance gates

G1 network trust diagram; G2 private DNS/egress; G3 CNI policy enforcement; G4 TLS renewal; G5 gateway auth/tenant isolation; G6 SSE; G7 least-privilege identity; G8 secrets/rotation; G9 CI policy/provenance; G10 GitOps/TrueFoundry ownership; G11 staging-to-prod approval; G12 canary/rollback; G13 SLO/telemetry/cost; G14 security/SRE sign-off.

See [companion lab](../labs/04-networking-security-gitops-delivery-check.md). Gates remain pending until executed and independently reviewed.

**Next:** Chapter 15.5 — Observability, SLOs and Load/Capacity Validation.
