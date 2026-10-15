# Chapter 10.2 — Hands-on Lab: Identity, SSO, Authentication and Authorization

**Part 10 | Authorized staging only | Formal execution pending**

## Objectives and prerequisites

Map identity across the IdP, TrueFoundry management plane, CI/CD, Kubernetes, cloud IAM and inference endpoints. Validate authentication vs authorization, least-privilege role assignments, token/session lifecycle, machine identity, negative tests, break-glass and rollback.

Prerequisites: approved non-production IdP application, disposable users/groups, staging TrueFoundry workspace, read-only cluster/cloud access, synthetic inference endpoint and security reviewer. **[SAFE-READ]** inspect; **[LAB-MUTATE]** staging-only identity changes; **[CHANGE-CONTROL]** production requires approval. Never share token strings, cookies, passwords or sensitive JWT claims.

## Lab 0 — Identity inventory [SAFE-READ]

```bash
kubectl config current-context
kubectl get sa -A
kubectl get rolebinding,clusterrolebinding -A
kubectl get pods -A -o custom-columns='NS:.metadata.namespace,NAME:.metadata.name,SA:.spec.serviceAccountName'
kubectl auth can-i get pods -n YOUR_NAMESPACE
```

Document actual IdP protocol, issuer, app registration, redirect URI/ACS, platform role scopes, automation identity, agent ServiceAccount, inference auth layer, token lifetime, logging and credential owner. Stop if an integration's behavior is unknown.

## Lab 1 — Human SSO [SAFE-READ / LAB-MUTATE]

Using disposable staging accounts: test valid sign-in, MFA requirement, rejected user, group-based access, sign-out and session expiry. If supported, test group provisioning/deprovisioning. Do not alter production IdP app or company-wide conditional-access rules.

## Lab 2 — Permission matrix [LAB-MUTATE]

Create scoped staging personas (viewer, developer, deployer, security reviewer) according to actual platform roles. Attempt authorized read/deploy operations and forbidden environment/identity changes. Capture expected vs actual decisions without tokens. Restore baseline role assignments after test.

## Lab 3 — CI and workload identity [SAFE-READ]

Review non-human credential provenance, federation trust, token audience, repository/environment restriction, cloud role and rotation. Test staging-only access using approved automation; confirm the same principal cannot deploy production or administer identity policy.

## Lab 4 — Inference authentication [SAFE-READ]

Send synthetic requests with valid, missing, expired and wrong-scope credentials through the actual staging inference route. Check 401/403/404 semantics, alternate hostnames, tenant boundaries and streaming reconnect behavior. Do not print bearer tokens in commands/logs. Confirm authorization at the actual enforcement layer, not just gateway reachability.

## Lab 5 — Revocation and incident tabletop [LAB-MUTATE]

Disable a disposable user or rotate a disposable machine credential. Measure when new logins and existing sessions/tokens stop working; note any non-instant revocation. Rehearse compromised credential, emergency access, IdP outage and rollback with incident owner.

## Lab 6 — Cleanup and evidence

Remove only lab-owned identities, groups and temporary credentials. Restore known-good staging mappings, confirm negative tests remain denied and attach sanitized evidence and reviewer notes.

## Failure scenarios — 26

| # | Scenario | Diagnostic focus |
|---|---|---|
| 01 | Wrong OIDC issuer | Issuer configuration |
| 02 | Wrong OIDC audience | Token/resource audience |
| 03 | OIDC JWKS rotation lag | Key cache and signature |
| 04 | Expired access token | Clock and expiry |
| 05 | Token not yet valid | Clock skew / nbf |
| 06 | Redirect URI mismatch | IdP app registration |
| 07 | SSO redirect loop | Cookies and callback |
| 08 | SAML signing cert rotated | Trust metadata |
| 09 | SAML ACS mismatch | Entity and ACS |
| 10 | MFA challenge blocked | Conditional access |
| 11 | User group missing | Provisioning/group claims |
| 12 | Group removal not reflected | Sync/session lifetime |
| 13 | Role assigned too broadly | Workspace/environment scope |
| 14 | Viewer can deploy | Privilege boundary |
| 15 | Developer can change identity policy | Separation of duties |
| 16 | CI token expired | Credential lifecycle |
| 17 | CI identity crosses environment | Federation trust |
| 18 | Personal token used in automation | Credential ownership |
| 19 | Inference without auth succeeds | Gateway/app enforcement |
| 20 | Valid token receives 401 | Header/issuer/audience |
| 21 | Authenticated request gets 403 | Authorization mapping |
| 22 | Alternate hostname bypasses auth | Route policy |
| 23 | Cross-tenant inference succeeds | Tenant isolation |
| 24 | Cloud workload identity denied | SA/federation/IAM |
| 25 | Disabled user retains active token | Revocation semantics |
| 26 | Break-glass action unaudited | Emergency access logging |

For each record observed status, sanitized logs, root-cause hypothesis, impact, recovery, rollback, owner and retest. Unsafe cases are tabletop-only.

## Ten acceptance gates

| Gate | Required evidence | Status |
|---|---|---|
| G1 | Approved staging scope and identity architecture | Pending |
| G2 | Human SSO and MFA behavior validated | Pending |
| G3 | Group mapping and provisioning/deprovisioning reviewed | Pending |
| G4 | Least-privilege role matrix positive/negative tests | Pending |
| G5 | CI identity scoped to environment and audience | Pending |
| G6 | Kubernetes/cloud identity boundaries documented | Pending |
| G7 | Inference authentication and authorization tested | Pending |
| G8 | Token/session expiry, revocation and rotation documented | Pending |
| G9 | Identity audit and incident tabletop completed | Pending |
| G10 | Cleanup, rollback, exceptions and reviewer sign-off | Pending |

**Pass:** each applicable gate evidenced with timestamps and owner approval; blocked or not-executed gates remain pending. Publishing the lab does not mean it was run.

## Evidence worksheet

UTC timestamp; IdP protocol; platform version; principal type (not credential); role/scope; action; expected and actual decision; sanitized request ID; audit event; revocation delay; rollback; reviewer and risk ticket.

**Next:** Chapter 10.3 — Kubernetes RBAC, ServiceAccounts and Workload Identity.
