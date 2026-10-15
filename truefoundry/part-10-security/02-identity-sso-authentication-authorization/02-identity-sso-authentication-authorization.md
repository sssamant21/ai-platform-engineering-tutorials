# Chapter 10.2 — Identity, SSO, Authentication and Authorization

**Part 10 — Security | Canonical production tutorial**

## Learning outcomes and safety

Design, inventory, verify and troubleshoot identity across the TrueFoundry management plane, CI/CD automation, Kubernetes and inference data plane. Understand human SSO, identity federation, machine credentials, authentication vs authorization, least-privilege role design, tenant boundaries, session/token lifecycle, break-glass access, audit and incident response.

**Safety labels:** [SAFE-READ] observation only; [LAB-MUTATE] disposable staging identities only; [CHANGE-CONTROL] approved production security change. TrueFoundry SSO providers, role names, SCIM support, API token formats and gateway authentication features vary by deployment/version: verify installed functionality and official configuration rather than assuming availability. Do not record real access tokens, cookies, credentials or customer data in lab evidence.

## 1. Identity architecture and planes

```text
Human → corporate IdP (SSO + MFA) → TrueFoundry management UI/API
CI pipeline → approved workload federation or scoped machine credential → platform API
TrueFoundry agent → Kubernetes API using ServiceAccount/RBAC
Model-serving workload → cloud storage via workload identity/IAM
Inference client → authenticated gateway/application endpoint → serving Pod
```

These are separate authorization systems. Successful login to TrueFoundry does not automatically grant Kubernetes cluster-admin, cloud storage access, or permission to invoke a specific inference endpoint. Likewise, a Kubernetes ServiceAccount is not a substitute for a human IdP identity.

## 2. Core identity concepts

| Concept | Meaning | Operational proof |
|---|---|---|
| Authentication (AuthN) | Verify principal identity | IdP sign-in, token validation, TLS identity |
| Authorization (AuthZ) | Decide permitted action on resource | role/policy evaluation and denied-action tests |
| SSO | Federated human sign-in | configured IdP, redirect/issuer, audit |
| OIDC | Identity protocol using ID tokens and OAuth 2.0 flows | issuer, audience, nonce/state, JWKS validation |
| SAML | Federated assertions for supported integrations | entity ID, ACS, signing certificate, attributes |
| OAuth 2.0 | Delegated authorization framework | scopes, token lifetime, audience |
| SCIM | Optional automated user/group provisioning | connector support, deprovision events |
| RBAC | Role-based permissions | role bindings and least privilege |
| ABAC/policy | Attribute- or condition-based permissions | effective policy and condition checks |
| Workload identity | Non-human federated service identity | OIDC trust, IAM role, token audience |
| Break-glass | Emergency privileged access | approval, expiry, logging and post-review |

OIDC ID tokens identify the authenticated user for the client; they are **not** generic bearer tokens for arbitrary APIs. APIs should validate access tokens as specified by their issuer and resource server. Never use unverified JWT claims as authorization decisions.

## 3. Map principals and permission boundaries

Create a principal inventory covering: employees, contractors, on-call operators, security reviewers, deployment bots, CI runners, platform agents, Kubernetes ServiceAccounts, inference clients and cloud workloads.

For each record identity source, owner, assigned role, scope, expiry, MFA/federation, secret storage, permitted actions, denied actions, audit stream and revocation path.

**Example role matrix — conceptual only; not TrueFoundry built-in role names:**

| Persona | Read deployments | Deploy staging | Deploy production | Change identity/policy |
|---|---|---|---|---|
| Viewer | Yes | No | No | No |
| Developer | Yes | Yes (approved) | No | No |
| Production deployer | Yes | Yes | Yes (approved) | No |
| Security administrator | Yes | No by default | No by default | Yes (approved) |
| Break-glass operator | Time-limited | Time-limited | Time-limited | Only if incident-authorized |

Enforce separation of duties and project/workspace/environment scope where the platform supports them. Do not assign administrative privileges merely to bypass an access error.

## 4. Human SSO configuration and validation

**Design review:**
1. Verify supported identity protocol (OIDC or SAML) and platform version.
2. Register the correct redirect URI/ACS and issuer/entity identifiers; use TLS.
3. Configure IdP signing keys/certificates and rotation ownership.
4. Map stable user IDs and groups; avoid authorization based on mutable email display strings where possible.
5. Require MFA and conditional access through the corporate IdP.
6. Test just-in-time provisioning and/or SCIM only if the integration supports them.
7. Test group removal, user disablement, active session expiry and token revocation behavior.
8. Retain emergency access with controlled approval, audit and expiry.

**Security review:** use PKCE for OAuth public clients where applicable, validate state/nonce, restrict redirect URIs, pin accepted issuer/audience and reject unsigned/untrusted token algorithms. Never expose IdP client secrets in Git.

## 5. CI/CD and machine-to-machine authentication

Prefer short-lived workload identity federation when supported, scoped to repository, branch/environment and intended audience. Where long-lived API tokens are unavoidable, store in an approved secret manager, rotate on schedule and restrict scope. Separate staging and production identities. Verify that a CI token cannot modify identity policy or deploy outside its environment.

GitLab CI/CD, GitHub Actions and cloud workload federation have different trust claims and role-assumption mechanics; verify actual provider implementation. Avoid sharing a human SSO token with automation.

## 6. Kubernetes and cloud identity boundaries [SAFE-READ]

```bash
kubectl config current-context
kubectl get serviceaccount -A
kubectl get rolebinding,clusterrolebinding -A
kubectl get pods -A -o custom-columns='NAMESPACE:.metadata.namespace,NAME:.metadata.name,SA:.spec.serviceAccountName'
kubectl auth can-i get pods -n YOUR_NAMESPACE
```

`kubectl auth can-i` checks the **current** user unless impersonation is explicitly authorized. A failure to impersonate another ServiceAccount is not evidence of that account's permissions. Review EKS IRSA/Pod Identity, AKS Workload Identity, cloud role trust, audience and permission boundaries using approved cloud read-only tools. Kubernetes RBAC and cloud IAM are independent.

## 7. Inference API authorization

Validate which component enforces inference authentication (gateway, service, mesh or application). Check missing/expired/wrong-audience tokens, wrong tenant/workspace/model, route bypass, alternate hostnames and streaming reconnection. Rate limits and quota are additional protections, not authorization. A `401` typically indicates missing/invalid authentication; `403` indicates authenticated but forbidden, though implementations vary. A `404` can intentionally conceal resource existence.

Test authorized and denied paths using synthetic payloads. Do not place bearer tokens in URL query strings, screenshots, shell history or shared curl logs. HTTP 200 with a truncated stream is not proof of completed authorized inference.

## 8. Session and token lifecycle

Document token issuance, intended audience/scope, expiration, refresh, key rotation, revocation, logout, session idle/absolute limits and incident invalidation. For JWTs, signature verification and issuer/audience/time checks are required, but short-lived tokens may remain valid until expiry unless the architecture supports revocation/introspection. Deprovisioning a user does not always instantly revoke every existing access token; verify actual behavior.

**Clock skew:** synchronize time across IdP, client, gateway and cluster; inspect `iat`, `nbf`, `exp` safely without leaking full token contents.

## 9. Identity observability and SLOs

Track SSO login success/failure, IdP outages, MFA challenge failure, token issuance/validation errors, 401/403 by route and principal class, unexpected privilege changes, service credential expiry, role binding drift, denied deployment attempts, break-glass activation and user deprovisioning latency. Redact token values and sensitive claims. Correlate identity events with GitOps changes and inference request IDs.

## 10. Twelve production troubleshooting runbooks

### RB-01 — SSO redirect loop
Validate redirect URI, issuer/entity ID, browser cookie SameSite/domain, TLS and IdP logs. Roll back misconfigured integration under change control; do not disable SSO broadly.

### RB-02 — SAML assertion rejected
Check assertion audience, ACS, clock skew, signing certificate rollover, NameID and required attributes. Validate trust chain and approved certificate rotation.

### RB-03 — OIDC token rejected
Inspect issuer, audience, expiration, JWKS cache, signature and scopes. Never disable signature validation as a workaround.

### RB-04 — MFA or conditional-access lockout
Review IdP policy and user/device context; follow approved recovery and break-glass workflow. Avoid blanket MFA removal.

### RB-05 — User authenticated but forbidden
Compare platform role, group sync, project/workspace scope and resource action. Restore minimum required role after approval.

### RB-06 — Group removal not reflected
Inspect provisioning/SCIM sync, cached group membership, active sessions and token lifetime. Verify deprovisioning end-to-end.

### RB-07 — CI token expired or revoked
Identify machine principal, token source, audience and expiry; rotate via secret manager or federation. Do not reuse personal tokens.

### RB-08 — CI can deploy production unexpectedly
Treat as privilege escalation risk. Restrict environment binding/role and review audit/deploy history; run negative tests.

### RB-09 — Inference returns 401
Inspect gateway auth mechanism, header forwarding, token issuer/audience, expiry and TLS. Do not log bearer values.

### RB-10 — Inference returns 403 or cross-tenant access
Inspect model/tenant authorization, route policies, service account and cache consistency. Cross-tenant success is a security incident.

### RB-11 — Workload identity fails cloud access
Check ServiceAccount annotations/bindings, projected token audience, cloud trust conditions and IAM. Restore narrowly scoped permissions.

### RB-12 — Compromised credential / break-glass event
Contain identity, revoke sessions/tokens as supported, rotate affected secrets, preserve audit, validate least privilege and review incident timeline.

## 11. Production change, rollback and acceptance

Record IdP/app registration, version, group/role mapping, allowed/denied test matrix, automation identities, rollback configuration, change approval, session impact, audit owner and user communication. Use staged changes and test both access grants and denials. Never remove the only working emergency access path without an approved replacement.

Complete the [companion lab](../labs/02-identity-sso-authentication-authorization-check.md): **26 failure scenarios and 10 acceptance gates**. Publication is not a claim of live execution, vendor certification or security sign-off.

**Next:** Chapter 10.3 — Kubernetes RBAC, ServiceAccounts and Workload Identity.
