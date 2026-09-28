# Part 2.4 — ServiceAccounts, RBAC & Permissions

## Purpose

Production authorization troubleshooting must answer four questions:

```text
WHO attempted WHAT operation
against WHICH resource
at WHICH authorization boundary?
```

A generic statement such as "the application has a permissions issue" is insufficient.

The objective is to prove:

```text
Request
   ↓
Requesting component
   ↓
Actual identity
   ↓
Authentication
   ↓
Authorization boundary
   ↓
Exact operation
   ↓
Authorization result
   ↓
Target resource
```

> **Production rule:** `UNKNOWN IDENTITY = NO AUTHORIZATION CHANGE`

Do not modify RBAC until the requesting identity and failed operation are understood.

---

## 1. Authentication and Authorization

Authentication answers:

```text
Who are you?
```

Authorization answers:

```text
What are you allowed to do?
```

Therefore:

```text
AUTHENTICATED ≠ AUTHORIZED
```

An identity can authenticate successfully and still receive an authorization denial.

---

## 2. Multiple Authorization Boundaries

A TrueFoundry workload can cross several independent authorization systems:

```text
User / CI identity
        ↓
TrueFoundry authorization
        ↓
Platform / tfy-agent identity
        ↓
Kubernetes API
        ↓
Kubernetes RBAC
        ↓
Workload
        ↓
Kubernetes ServiceAccount
        ↓
Cloud workload identity
        ↓
Cloud IAM
        ↓
External resource
```

Never collapse these systems into one permission model.

```text
TrueFoundry Authorization
        ≠
Kubernetes RBAC
        ≠
Cloud IAM
```

Authorization success at one boundary does not prove authorization at another.

---

## 3. Identity Types

During an incident, classify the identity before investigating permissions.

Common categories include:

- Human identity
- CI/CD identity
- TrueFoundry identity
- Platform integration identity
- `tfy-agent` identity
- Kubernetes ServiceAccount
- Cloud identity
- External-service identity

Record the actual identity rather than relying on an expected identity.

---

## 4. Platform Identity vs Workload Identity

A TrueFoundry deployment operation may follow:

```text
TrueFoundry Control Plane
        ↓
tfy-agent / platform integration
        ↓
Kubernetes API
        ↓
Workload creation
```

The application may later use:

```text
Pod
 ↓
Workload ServiceAccount
 ↓
Kubernetes / cloud resources
```

These identities can be completely different.

```text
PLATFORM IDENTITY ≠ WORKLOAD IDENTITY
```

If workload creation fails before a Pod exists, investigating the intended Pod ServiceAccount may not explain the failure.

First identify the identity that actually attempted the Kubernetes API operation.

---

## 5. Human Identity vs Workload Identity

An engineer using `kubectl` follows approximately:

```text
Engineer
   ↓
kubectl
   ↓
Human authentication
   ↓
Kubernetes authorization
```

An application follows:

```text
Pod
 ↓
ServiceAccount
 ↓
Kubernetes / cloud authorization
```

Therefore:

```text
ENGINEER CAN ACCESS RESOURCE
        ≠
APPLICATION CAN ACCESS RESOURCE
```

and:

```text
APPLICATION CAN ACCESS RESOURCE
        ≠
ENGINEER CAN ACCESS RESOURCE
```

Manual access does not prove workload access.

---

## 6. Kubernetes ServiceAccounts

A Kubernetes ServiceAccount provides a workload identity within a namespace.

A Pod may specify:

```yaml
spec:
  serviceAccountName: model-serving-sa
```

Production investigation should establish:

```text
Expected ServiceAccount:
Observed ServiceAccount:
Namespace:
Cluster:
```

Never treat the ServiceAccount name alone as complete identity.

```text
Cluster
+
Namespace
+
ServiceAccount
```

is the useful operational identity.

```text
SERVICEACCOUNT NAME ≠ COMPLETE IDENTITY
```

---

## 7. Verify the Observed ServiceAccount

For an existing Pod:

```bash
kubectl get pod <pod> \
  -n <namespace> \
  -o jsonpath='{.spec.serviceAccountName}{"\n"}'
```

Then verify the object:

```bash
kubectl get serviceaccount <service-account> \
  -n <namespace>
```

Record:

```text
Expected ServiceAccount:
Observed ServiceAccount:
Match:
YES / NO / UNKNOWN
```

Until verified:

```text
EXPECTED SERVICEACCOUNT ≠ OBSERVED SERVICEACCOUNT
```

---

## 8. ServiceAccount Existence Does Not Prove Authorization

A successful:

```bash
kubectl get serviceaccount <service-account> \
  -n <namespace>
```

proves that the object exists.

It does not prove that the identity has the permissions required by the application.

```text
ServiceAccount Exists ≠ Authorization Correct
```

Authorization must be evaluated against the exact required operation.

---

## 9. The `default` ServiceAccount

If a Pod does not explicitly select another ServiceAccount, Kubernetes normally associates it with the namespace's `default` ServiceAccount.

Do not automatically classify:

```text
serviceAccountName: default
```

as a defect.

```text
default ServiceAccount ≠ Automatic Misconfiguration
```

Determine whether it is expected.

However, permissions granted to a default ServiceAccount may affect multiple workloads that use it.

```text
Permission Granted to default ServiceAccount
        =
Potential Multi-Workload Blast Radius
```

---

## 10. Modern ServiceAccount Tokens

Do not assume every ServiceAccount has or requires a long-lived Secret-backed token.

Modern Kubernetes supports bound, time-limited ServiceAccount credentials.

```text
SERVICEACCOUNT EXISTS
        ≠
LONG-LIVED TOKEN SECRET MUST EXIST
```

Do not troubleshoot ordinary RBAC incidents by extracting ServiceAccount tokens.

```text
RBAC TROUBLESHOOTING
        ≠
SERVICEACCOUNT TOKEN EXTRACTION
```

---

## 11. Kubernetes RBAC Objects

Kubernetes RBAC commonly uses:

```text
Role
ClusterRole
RoleBinding
ClusterRoleBinding
```

Conceptually:

```text
Subject
   ↓
Binding
   ↓
Role / ClusterRole
   ↓
Rules
   ↓
Authorization result
```

Do not infer permissions from object names.

```text
ROLE NAME ≠ AUTHORIZATION PROOF
```

---

## 12. Role

A Role defines authorization rules within a namespace.

A rule can include:

- API groups
- Resources
- Subresources
- Verbs
- Resource-name restrictions

A Role by itself grants nothing to a subject.

```text
Role Exists ≠ Subject Has Permission
```

A binding must connect an identity to those rules.

---

## 13. RoleBinding

A RoleBinding grants permissions to subjects within the RoleBinding's namespace.

Verify:

```text
Binding namespace
Subject kind
Subject name
Subject namespace
roleRef kind
roleRef name
```

Never stop because the RoleBinding exists.

```text
BINDING EXISTS ≠ CORRECT IDENTITY BOUND
```

and:

```text
Correct Identity Bound ≠ Required Permission Present
```

---

## 14. ClusterRole

A ClusterRole is cluster-scoped as an RBAC object, but its existence does not mean its rules grant unrestricted cluster access.

Its actual rules determine its permissions.

```text
ClusterRole ≠ Cluster Administrator
```

A ClusterRole can also be reused through namespace-scoped RoleBindings.

---

## 15. RoleBinding Referencing a ClusterRole

A RoleBinding can reference a ClusterRole.

Conceptually:

```text
ClusterRole
      +
RoleBinding in namespace-a
      ↓
Authorization within namespace-a
for applicable namespaced resources
```

Therefore:

```text
ClusterRole Referenced ≠ Cluster-Wide Grant
```

This distinction is important during blast-radius analysis.

---

## 16. ClusterRoleBinding

A ClusterRoleBinding binds a ClusterRole to subjects with cluster-wide binding scope.

```text
Subject
   ↓
ClusterRoleBinding
   ↓
ClusterRole
```

Do not infer the exact privilege solely from the existence of the ClusterRoleBinding. Inspect the referenced ClusterRole rules.

```text
ClusterRoleBinding
        =
Potential Cluster-Wide Authorization Impact
```

---

## 17. Exact Authorization Tuple

Production authorization analysis should capture:

```text
Identity
+
Verb
+
API group
+
Resource
+
Subresource, if applicable
+
Resource name, if constrained
+
Namespace / cluster scope
```

Example:

```text
Identity:
system:serviceaccount:model-serving:model-api

Verb:
create

API group:
batch

Resource:
jobs

Namespace:
model-serving
```

This is much stronger than saying that `model-api` "has Kubernetes access."

---

## 18. Permission Is Operation-Specific

These are separate authorization questions:

```text
get pods
list pods
get pods/log
create jobs
delete jobs
get secrets
```

Therefore:

```text
ONE ALLOWED ACTION ≠ GENERAL AUTHORIZATION
```

and:

```text
RESOURCE MATCH ≠ OPERATION MATCH
```

A subject allowed to `get pods` is not automatically allowed to create Pods, access `pods/log`, or read Secrets.

---

## 19. Start With the Failed Operation

Do not begin with:

```text
Check all RBAC.
```

Start with:

```text
What exact operation failed?
```

Record:

```text
Verb:
API group:
Resource:
Subresource:
Resource name:
Namespace / scope:
```

Production rule:

```text
TEST THE FAILED OPERATION
NOT A CONVENIENT OPERATION
```

---

## 20. `kubectl auth can-i`

For the current caller:

```bash
kubectl auth can-i get pods -n <namespace>
```

For an exact operation:

```bash
kubectl auth can-i create jobs -n <namespace>
```

Record:

```text
Identity:
Operation:
Namespace:
Result:
Timestamp:
```

A successful result for one operation must not be generalized.

```text
ONE ALLOWED ACTION ≠ GENERAL AUTHORIZATION
```

---

## 21. ServiceAccount Authorization Testing

Where impersonation is permitted, a targeted check may use:

```bash
kubectl auth can-i create jobs \
  --as=system:serviceaccount:<namespace>:<service-account> \
  -n <namespace>
```

But `--as` introduces another authorization boundary:

```text
Operator
   ↓
Impersonation authorization
   ↓
Target ServiceAccount
   ↓
Requested-resource authorization
```

Therefore:

```text
kubectl auth can-i --as ...
        ≠
Universally Available Read-Only Check
```

---

## 22. Impersonation Failure

If the operator cannot impersonate the target identity:

```text
Operator impersonation capability:
DENIED
```

the correct conclusion is:

```text
Target ServiceAccount authorization:
UNKNOWN
```

not:

```text
Target ServiceAccount:
DENIED
```

Production rule:

```text
IMPERSONATION DENIED
        ≠
TARGET IDENTITY DENIED
```

Never turn missing evidence into a root-cause claim.

---

## 23. Authorization Diagnostic Gates

Use four gates.

### Gate 1 — Identity

```text
WHO made the request?
```

### Gate 2 — Binding

```text
HOW is the identity connected
to authorization rules?
```

### Gate 3 — Permission

```text
DOES the effective rule authorize
the exact operation?
```

### Gate 4 — External Authorization

```text
DOES the operation cross another
authorization boundary?
```

Examples include cloud IAM, object storage, container registries, model repositories, external APIs, databases, and secret managers.

---

## 24. A 403 Is Evidence, Not Root Cause

An error such as:

```text
403 Forbidden
```

does not automatically mean Kubernetes RBAC.

Determine:

```text
Which endpoint returned it?
Which component made the request?
Which identity made the request?
Which authorization system evaluated it?
```

Potential sources include:

- TrueFoundry API
- Kubernetes API
- Cloud API
- Object storage
- Model repository
- External service

Therefore:

```text
403 ≠ Automatically Kubernetes RBAC
```

---

## 25. Preserve the Original Authorization Error

Whenever possible capture:

```text
Timestamp
Component
Identity
Operation
Resource
Namespace/scope
Original error
```

A Kubernetes authorization error may expose the identity, verb, resource, API group, namespace, and denial result.

Do not reduce this evidence immediately to "RBAC problem."

---

## 26. `tfy-agent` and Platform Authorization

The TrueFoundry integration path can conceptually involve:

```text
TrueFoundry Control Plane
        ↓
tfy-agent / platform components
        ↓
Kubernetes API
        ↓
RBAC evaluation
        ↓
Target resource
```

Do not hard-code one universal Role or ClusterRole layout for every TrueFoundry installation.

Instead:

```text
DISCOVER ACTUAL INSTALLED RBAC
```

Installation method and platform version can affect deployed authorization configuration.

Keep:

```text
Agent Running ≠ Agent Authorized for Requested Operation
```

and:

```text
Correct Namespace ≠ Agent Authorized for Namespace
```

---

## 27. Example — Platform Authorization Failure

Suppose:

```text
TrueFoundry request accepted       PROVEN
tfy-agent communication            PROVEN
Target namespace                   PROVEN
Kubernetes request attempted       PROVEN
Kubernetes authorization           DENIED
Application Pod                    NOT CREATED
```

Then:

```text
Lowest Proven Healthy Layer:
Kubernetes API request initiation
```

and:

```text
First Failed Transition:
Kubernetes request → RBAC authorization
```

The application's intended ServiceAccount may not yet be relevant. The requesting platform identity should be investigated first.

---

## 28. Example — Workload Authorization Failure

Suppose:

```text
Pod running                  PROVEN
ServiceAccount               PROVEN
Application request          PROVEN
Kubernetes authorization     DENIED
```

Now trace:

```text
Pod
 ↓
Observed ServiceAccount
 ↓
Binding
 ↓
Role / ClusterRole
 ↓
Exact operation
```

This is a different failure domain from platform deployment authorization.

---

## 29. Example — Wrong Binding Subject

Suppose:

```text
Actual workload ServiceAccount:
model-serving-sa

RoleBinding subject:
model-api-sa
```

while the referenced Role contains the required permission.

Then:

```text
Role exists              PROVEN
Permission exists        PROVEN
RoleBinding exists       PROVEN
Correct identity bound   FAILED
```

Possible first failed transition:

```text
Workload identity
      →
RBAC binding
```

Therefore:

```text
Role Exists
+
RoleBinding Exists
≠
Correct Authorization
```

---

## 30. Example — Wrong Namespace

Suppose:

```text
Workload identity:
namespace-a/model-sa
```

while the relevant RoleBinding exists in:

```text
namespace-b
```

Matching names do not prove authorization.

Always reason using:

```text
Cluster
+
Namespace
+
Identity
```

---

## 31. Secret Access

Secret authorization deserves additional security scrutiny.

Do not expose Secret values simply to prove authorization.

Prefer authorization evidence such as:

```text
Can identity X perform operation Y
against Secret resources?
```

rather than retrieving and decoding the Secret.

Production rules:

```text
SECRET EXISTS ≠ SECRET SHOULD BE DECODED
```

and:

```text
AUTHORIZATION VALIDATION ≠ DATA EXTRACTION
```

The production lab must not decode Kubernetes Secrets.

---

## 32. Effective Privilege

Direct RBAC rules do not always represent the complete security impact of an identity.

Potentially sensitive capabilities include:

- Secret access
- RBAC modification
- Identity impersonation
- Workload creation
- Use of powerful ServiceAccounts

Therefore:

```text
DIRECT PERMISSION
        ≠
COMPLETE EFFECTIVE PRIVILEGE
```

Part 2.4 recognizes these relationships; deeper security hardening belongs in the security curriculum.

---

## 33. Least Privilege

When authorization is missing, determine:

```text
Failed operation
      ↓
Required API group
      ↓
Required resource
      ↓
Required verb
      ↓
Required scope
```

Do not solve permission incidents by immediately granting broad privileges.

```text
PERMISSION FAILURE
        ≠
JUSTIFICATION FOR ADMIN ACCESS
```

Broad administrative access used merely to see whether an error disappears produces poor diagnostic evidence while increasing risk.

---

## 34. System-Managed RBAC

Treat Kubernetes `system:*` RBAC resources carefully.

They may represent Kubernetes-managed authorization configuration.

```text
system:* RBAC resource
        =
Potential Kubernetes-managed object
```

Production rule:

```text
DO NOT MODIFY SYSTEM-MANAGED RBAC
AS INCIDENT TROUBLESHOOTING
```

First determine ownership and reconciliation behavior.

---

## 35. Desired-State Ownership

RBAC resources may be managed through:

- TrueFoundry
- Helm
- Argo CD
- GitOps
- Terraform / OpenTofu
- Kubernetes Operator
- Platform automation
- Manual configuration

Trace:

```text
Observed Kubernetes RBAC
          ↑
Rendered configuration
          ↑
Authoritative desired state
```

Production rule:

```text
DO NOT PATCH RBAC DIRECTLY
WHEN ANOTHER SYSTEM OWNS DESIRED STATE
```

---

## 36. Authorization Drift

Compare:

```text
Expected identity
vs
Observed identity

Expected binding
vs
Observed binding

Expected roleRef
vs
Observed roleRef

Expected permission
vs
Observed permission

Expected scope
vs
Observed scope

Expected cloud identity
vs
Observed cloud identity
```

But:

```text
Observed Drift ≠ Permission to Patch
```

Detection and remediation are separate activities.

---

## 37. Shared ServiceAccount Blast Radius

A ServiceAccount may be used by multiple workloads.

Before changing its authorization, determine its consumers.

```text
ServiceAccount
      ↓
Workload A
Workload B
Workload C
```

Therefore:

```text
SERVICEACCOUNT CHANGE
        ≠
SINGLE-WORKLOAD CHANGE
```

unless proven otherwise.

---

## 38. Shared Role Blast Radius

Roles and especially reusable ClusterRoles may participate in multiple bindings.

Before changing authorization rules, determine which bindings depend on them.

```text
ClusterRole
   ↓
Binding A
Binding B
Binding C
```

Therefore:

```text
ROLE CHANGE
        ≠
SINGLE-BINDING CHANGE
```

Authorization remediation requires blast-radius analysis.

---

## 39. RBAC Change Safety

An RBAC change can affect:

- One operation
- One identity
- One workload
- Multiple workloads
- One namespace
- Multiple namespaces
- Cluster-wide behavior

Therefore:

```text
RBAC CHANGE
        =
SECURITY + BLAST-RADIUS EVENT
```

The production lab should detect and document authorization problems but must not modify RBAC.

---

## 40. Cloud Identity

Applications commonly access resources such as:

- Object storage
- Container registries
- Model storage
- KMS
- Databases
- Queues
- Secret managers
- Cloud APIs

The path may look like:

```text
Pod
 ↓
Kubernetes ServiceAccount
 ↓
Workload identity integration
 ↓
Cloud identity
 ↓
Cloud IAM
 ↓
Cloud resource
```

Keep:

```text
Kubernetes ServiceAccount ≠ Cloud IAM Principal
```

and:

```text
Kubernetes RBAC Success ≠ Cloud IAM Success
```

---

## 41. Example — Kubernetes Works, Cloud Fails

Suppose:

```text
Pod running                  PROVEN
ServiceAccount               PROVEN
Kubernetes authorization     PROVEN
Cloud request attempted      PROVEN
Cloud authorization          DENIED
```

Then Kubernetes RBAC is not the first failed transition.

Record:

```text
Lowest Proven Healthy Layer:
Workload execution / Kubernetes authorization
```

and:

```text
First Failed Transition:
Workload cloud identity → cloud authorization
```

Do not keep modifying Kubernetes RBAC.

---

## 42. Provider-Neutral Cloud Identity

Part 2.4 remains provider-neutral.

Record:

```text
Kubernetes ServiceAccount:
Cloud identity:
Identity mapping mechanism:
Target cloud resource:
Failed cloud operation:
Authorization result:
```

AWS, Azure, and GCP implementations can then be handled in provider-specific operational documentation.

---

## 43. Minimum Supported Blast Radius

Do not generalize authorization failures.

Evidence showing:

```text
model-api-sa cannot create Jobs
```

supports:

```text
This identity cannot perform this tested operation.
```

It does not automatically support:

```text
Namespace RBAC is broken.
```

or:

```text
Cluster RBAC is broken.
```

Determine the smallest scope supported by evidence:

```text
Operation
Identity
Workload
Namespace
Cluster
```

Record:

```text
Minimum Supported Blast Radius:
```

---

## 44. Lowest Proven Healthy Layer

Example:

```text
TrueFoundry request       PROVEN
Agent connectivity        PROVEN
Target namespace          PROVEN
Kubernetes request        PROVEN
Authorization             DENIED
Resource creation         NOT REACHED
```

Record:

```text
Lowest Proven Healthy Layer:
Kubernetes API request initiation
```

This prevents investigation from repeatedly returning to already-proven layers.

---

## 45. First Failed Transition

For the same example:

```text
Kubernetes request
        ↓
Authorization evaluation
        X
```

Record:

```text
First Failed Transition:
Kubernetes API request → authorization
```

This is more actionable than saying "TrueFoundry permissions problem."

---

## 46. Evidence Confidence

Use:

```text
PROVEN
SUPPORTED
UNKNOWN
ASSUMED
```

Example:

```text
Actual ServiceAccount       PROVEN
Failed operation            PROVEN
Authorization denied        PROVEN
Missing binding             SUPPORTED
Cloud IAM issue             UNKNOWN
TrueFoundry defect          ASSUMED
```

Keep:

```text
ASSUMED ≠ ROOT CAUSE
```

---

## 47. Current Actionable Owner

Ownership should follow the first supported failure domain and the authoritative desired-state owner.

Example:

```text
Symptom:
Deployment creation failed

Requesting identity:
Platform integration identity

Failed operation:
create deployment

Authorization boundary:
Kubernetes RBAC

Authorization result:
DENIED

Authoritative configuration:
GitOps repository

Current Actionable Owner:
Platform/IaC team

Requested Action:
Validate the minimum required namespace-scoped authorization
```

Therefore:

```text
SYMPTOM ≠ FAILURE DOMAIN ≠ OWNER
```

---

## 48. Authorization Evidence Handoff Contract

Use this during production escalation:

```text
Environment:
TrueFoundry Workspace:

Cluster:
Kubernetes context:
Namespace:

Original error:
Observation timestamp:

Failed operation:

Verb:
API group:
Resource:
Subresource:
Resource name:
Scope:

Requesting component:

Identity type:
Actual identity:

Platform identity:
If applicable

Workload ServiceAccount:
If applicable

Authorization boundary:
TrueFoundry / Kubernetes / Cloud / External

Authorization result:

Impersonation required:
YES / NO

Impersonation permitted:
YES / NO / NOT TESTED

Relevant Role:

Relevant ClusterRole:

Relevant RoleBinding:

Relevant ClusterRoleBinding:

Cloud identity:
If applicable

Cloud IAM result:
If applicable

Shared identity consumers:
If applicable

Shared role consumers:
If applicable

Lowest Proven Healthy Layer:

First Failed Transition:

Minimum Supported Blast Radius:

Evidence Confidence:

Authoritative Configuration Owner:

Current Actionable Owner:

Requested Action:

Secret values exposed:
NO

ServiceAccount token extracted:
NO

Authorization mutation performed:
NO
```

---

## 49. Production Troubleshooting Workflow

```text
CAPTURE ORIGINAL FAILURE
        ↓
IDENTIFY EXACT FAILED OPERATION
        ↓
IDENTIFY REQUESTING COMPONENT
        ↓
PROVE ACTUAL IDENTITY
        ↓
CLASSIFY IDENTITY
Human / Platform / Workload
        ↓
IDENTIFY AUTHORIZATION BOUNDARY
        ↓
VERIFY TARGET CLUSTER / NAMESPACE / SCOPE
        ↓
TEST EXACT OPERATION
        ↓
IF IMPERSONATING:
VERIFY IMPERSONATION CAPABILITY
        ↓
TRACE SUBJECT
        ↓
TRACE BINDING
        ↓
TRACE ROLE / CLUSTERROLE
        ↓
VERIFY EXACT RULE
        ↓
CHECK EFFECTIVE / INDIRECT PRIVILEGE
WHEN RELEVANT
        ↓
IF KUBERNETES AUTHORIZATION SUCCEEDS:
MOVE TO NEXT AUTHORIZATION BOUNDARY
        ↓
CLOUD IAM / EXTERNAL AUTHORIZATION
        ↓
IDENTIFY LOWEST PROVEN HEALTHY LAYER
        ↓
IDENTIFY FIRST FAILED TRANSITION
        ↓
DETERMINE MINIMUM SUPPORTED BLAST RADIUS
        ↓
IDENTIFY SHARED IDENTITY / ROLE CONSUMERS
        ↓
IDENTIFY AUTHORITATIVE CONFIGURATION OWNER
        ↓
IDENTIFY CURRENT ACTIONABLE OWNER
        ↓
APPROVED LEAST-PRIVILEGE REMEDIATION
        ↓
REGRESSION VALIDATION
```

---

## 50. Production Rules

Preserve these rules during production investigation:

```text
AUTHENTICATED ≠ AUTHORIZED

UNKNOWN IDENTITY = NO AUTHORIZATION CHANGE

PLATFORM IDENTITY ≠ WORKLOAD IDENTITY

SERVICEACCOUNT NAME ≠ COMPLETE IDENTITY

EXPECTED SERVICEACCOUNT ≠ OBSERVED SERVICEACCOUNT

ServiceAccount Exists ≠ Authorization Correct

default ServiceAccount ≠ Automatic Misconfiguration

SERVICEACCOUNT EXISTS
≠
LONG-LIVED TOKEN SECRET MUST EXIST

Role Exists ≠ Subject Has Permission

BINDING EXISTS ≠ CORRECT IDENTITY BOUND

Correct Identity Bound ≠ Required Permission Present

ClusterRole ≠ Cluster Administrator

ClusterRole Referenced ≠ Cluster-Wide Grant

RESOURCE MATCH ≠ OPERATION MATCH

ONE ALLOWED ACTION ≠ GENERAL AUTHORIZATION

TEST THE FAILED OPERATION
NOT A CONVENIENT OPERATION

ENGINEER CAN ACCESS RESOURCE
≠
APPLICATION CAN ACCESS RESOURCE

APPLICATION CAN ACCESS RESOURCE
≠
ENGINEER CAN ACCESS RESOURCE

IMPERSONATION DENIED
≠
TARGET IDENTITY DENIED

403 ≠ Automatically Kubernetes RBAC

Agent Running
≠
Agent Authorized for Requested Operation

TrueFoundry Authorization
≠
Kubernetes RBAC
≠
Cloud IAM

Kubernetes ServiceAccount
≠
Cloud IAM Principal

Kubernetes RBAC Success
≠
Cloud IAM Success

SECRET EXISTS
≠
SECRET SHOULD BE DECODED

AUTHORIZATION VALIDATION
≠
DATA EXTRACTION

RBAC TROUBLESHOOTING
≠
SERVICEACCOUNT TOKEN EXTRACTION

DIRECT PERMISSION
≠
COMPLETE EFFECTIVE PRIVILEGE

PERMISSION FAILURE
≠
JUSTIFICATION FOR ADMIN ACCESS

SERVICEACCOUNT CHANGE
≠
SINGLE-WORKLOAD CHANGE

ROLE CHANGE
≠
SINGLE-BINDING CHANGE

Observed Drift
≠
Permission to Patch

RBAC CHANGE
=
SECURITY + BLAST-RADIUS EVENT

DO NOT MODIFY SYSTEM-MANAGED RBAC
AS INCIDENT TROUBLESHOOTING

DO NOT PATCH RBAC DIRECTLY
WHEN ANOTHER SYSTEM OWNS DESIRED STATE

ASSUMED ≠ ROOT CAUSE

SYMPTOM ≠ FAILURE DOMAIN ≠ OWNER
```

---

## 51. Acceptance Standard

A production authorization investigation is complete only when it can answer:

```text
WHAT operation failed?

WHO actually attempted it?

WHAT identity type was involved?

WHICH authorization boundary evaluated it?

WHAT exact permission was required?

WHAT authorization result was observed?

WHAT is the lowest proven healthy layer?

WHERE is the first failed transition?

WHAT is the minimum supported blast radius?

IS the identity or role shared?

WHO owns authoritative desired state?

WHO is the current actionable owner?

WHAT minimum remediation is required?

WERE any Secret values exposed?
NO

WAS any ServiceAccount token extracted?
NO

WAS production RBAC mutated during diagnosis?
NO
```

---

## Summary

Production RBAC troubleshooting should be evidence-driven and operation-specific.

The correct approach is:

```text
PROVE THE FAILED OPERATION
        ↓
PROVE THE REQUESTING IDENTITY
        ↓
PROVE THE AUTHORIZATION BOUNDARY
        ↓
PROVE THE EXACT AUTHORIZATION RESULT
        ↓
FIND THE FIRST FAILED TRANSITION
        ↓
PROVE THE MINIMUM SUPPORTED BLAST RADIUS
        ↓
IDENTIFY AUTHORITATIVE DESIRED-STATE OWNERSHIP
        ↓
APPLY ONLY APPROVED LEAST-PRIVILEGE REMEDIATION
        ↓
VALIDATE THE ORIGINAL OPERATION
```

The goal is not to make an authorization error disappear by increasing privilege. The goal is to identify the exact failed authorization transition, preserve the smallest justified permission scope, and hand remediation to the authoritative owner with defensible evidence.
