# Part 2.4 Lab — ServiceAccounts, RBAC & Permissions Validation

> **[SAFE-READ] Production Validation Lab**

## Purpose

This lab validates Kubernetes workload identities and RBAC authorization using production-safe, read-oriented investigation techniques.

The goal is not to change permissions. The goal is to prove:

```text
WHO attempted WHAT operation
against WHICH resource
at WHICH authorization boundary?
```

This lab follows the production rule:

```text
UNKNOWN IDENTITY = NO AUTHORIZATION CHANGE
```

---

## Safety Policy

This is a production validation lab.

Use only approved read-oriented commands such as:

```text
kubectl config current-context
kubectl cluster-info
kubectl get
kubectl describe
kubectl logs
kubectl auth can-i
```

Do not use this lab to run:

```text
kubectl create
kubectl apply
kubectl delete
kubectl edit
kubectl patch
kubectl replace
kubectl scale
kubectl rollout restart

helm install
helm upgrade
helm uninstall
```

Do not modify:

```text
ServiceAccounts
Roles
ClusterRoles
RoleBindings
ClusterRoleBindings
Secrets
Deployments
StatefulSets
DaemonSets
Jobs
CronJobs
Pods
```

Do not:

- Decode Kubernetes Secret values.
- Print Secret values.
- Extract ServiceAccount tokens.
- Create ServiceAccount tokens.
- Copy credentials into the lab evidence.
- Grant temporary `cluster-admin`.
- Patch RBAC to see whether an error disappears.
- Modify `system:*` RBAC resources.
- Assume `kubectl auth can-i --as` is permitted.
- Treat failed impersonation as proof that the target identity is unauthorized.

Production rules:

```text
AUTHORIZATION VALIDATION ≠ DATA EXTRACTION

RBAC TROUBLESHOOTING ≠ SERVICEACCOUNT TOKEN EXTRACTION

PERMISSION FAILURE ≠ JUSTIFICATION FOR ADMIN ACCESS

Observed Drift ≠ Permission to Patch
```

---

# Lab Objectives

By the end of this lab, you should be able to:

1. Prove the Kubernetes context and target cluster.
2. Prove the target namespace.
3. Identify the requesting component.
4. Distinguish human, platform, and workload identities.
5. Identify the actual ServiceAccount used by an existing Pod.
6. Trace ServiceAccount → binding → Role/ClusterRole.
7. Understand RoleBinding versus ClusterRoleBinding scope.
8. Test an exact authorization operation safely.
9. Recognize the impersonation boundary introduced by `--as`.
10. Separate Kubernetes RBAC from TrueFoundry authorization and cloud IAM.
11. Determine the minimum supported blast radius.
12. Identify shared ServiceAccount or role consumers.
13. Determine authoritative desired-state ownership.
14. Record the Lowest Proven Healthy Layer.
15. Record the First Failed Transition.
16. Produce a production-ready evidence handoff without changing RBAC.

---

# Variables

Choose values appropriate for the environment.

Examples below use placeholders.

```text
<NAMESPACE>
<POD>
<SERVICE_ACCOUNT>
<ROLE>
<ROLE_BINDING>
<CLUSTER_ROLE>
<CLUSTER_ROLE_BINDING>
<VERB>
<RESOURCE>
<API_GROUP>
```

Do not blindly paste values from another environment.

---

# Phase 1 — Prove the Kubernetes Target

## Step 1 — Verify Current Context

Run:

```bash
kubectl config current-context
```

Record:

```text
Kubernetes context:
Observation timestamp:
```

Do not continue if the context is not the intended environment.

Production rule:

```text
UNKNOWN TARGET = NO MUTATION
```

Although this lab performs no mutation, target verification is still mandatory.

---

## Step 2 — Verify Cluster Connectivity

Run:

```bash
kubectl cluster-info
```

Record:

```text
Cluster reachable:
YES / NO

Observed API endpoint:
Observation timestamp:
```

Do not expose credentials from kubeconfig or authentication plugins.

---

## Step 3 — Verify the Namespace

Run:

```bash
kubectl get namespace <NAMESPACE>
```

Optionally inspect metadata and status:

```bash
kubectl get namespace <NAMESPACE> -o wide
```

Record:

```text
Namespace:
Status:
Observation timestamp:
```

Do not assume that a namespace name from a TrueFoundry Workspace is the actual Kubernetes namespace without verification.

```text
WORKSPACE NAME ≠ PROOF OF ACTUAL NAMESPACE
```

---

# Phase 2 — Capture the Original Failure

## Step 4 — Record the Original Error

Before inspecting RBAC, preserve the original failure evidence.

Record:

```text
Original error:

Timestamp:

Requesting component:

Endpoint / API:
If known

Operation:
If known

Identity:
If present in the error

Namespace / scope:
If present

HTTP / Kubernetes status:
If present
```

Do not immediately rewrite:

```text
403 Forbidden
```

as:

```text
Kubernetes RBAC issue
```

Production rule:

```text
403 ≠ Automatically Kubernetes RBAC
```

---

## Step 5 — Classify the Authorization Boundary

Determine which authorization system appears to have evaluated the failed request.

Possible boundaries:

```text
TrueFoundry authorization
Kubernetes RBAC
Cloud IAM
External-service authorization
UNKNOWN
```

Record:

```text
Authorization boundary:
Evidence:
Confidence:
PROVEN / SUPPORTED / UNKNOWN / ASSUMED
```

Keep:

```text
TrueFoundry Authorization
        ≠
Kubernetes RBAC
        ≠
Cloud IAM
```

---

# Phase 3 — Identify the Requesting Identity

## Step 6 — Classify the Requesting Component

Determine whether the failing operation originated from:

```text
Human / kubectl
CI/CD
TrueFoundry platform operation
tfy-agent / platform component
Application workload
Cloud workload identity
External integration
UNKNOWN
```

Record:

```text
Requesting component:
Identity type:
Expected identity:
Observed identity:
Evidence:
```

Production rule:

```text
PLATFORM IDENTITY ≠ WORKLOAD IDENTITY
```

---

# Phase 4 — Verify Workload ServiceAccount

Only perform these steps when an application Pod exists and its identity is relevant to the failed operation.

## Step 7 — Verify the Pod Exists

Run:

```bash
kubectl get pod <POD> -n <NAMESPACE> -o wide
```

Record:

```text
Pod:
Namespace:
Phase:
Node:
Observation timestamp:
```

If the Pod does not exist, do not invent its observed ServiceAccount.

Record:

```text
Observed workload ServiceAccount:
NOT OBSERVABLE — POD DOES NOT EXIST
```

A platform deployment authorization failure may occur before the application ServiceAccount becomes relevant.

---

## Step 8 — Read the Pod ServiceAccount

Run:

```bash
kubectl get pod <POD> \
  -n <NAMESPACE> \
  -o jsonpath='{.spec.serviceAccountName}{"\n"}'
```

Record:

```text
Pod:
Namespace:
Expected ServiceAccount:
Observed ServiceAccount:
Match:
YES / NO / UNKNOWN
```

Production rules:

```text
EXPECTED SERVICEACCOUNT ≠ OBSERVED SERVICEACCOUNT

SERVICEACCOUNT NAME ≠ COMPLETE IDENTITY
```

The useful identity is:

```text
Cluster
+
Namespace
+
ServiceAccount
```

---

## Step 9 — Verify ServiceAccount Metadata

Run:

```bash
kubectl get serviceaccount <SERVICE_ACCOUNT> \
  -n <NAMESPACE>
```

For additional metadata:

```bash
kubectl describe serviceaccount <SERVICE_ACCOUNT> \
  -n <NAMESPACE>
```

Do not use this step to extract credentials or token data.

Record:

```text
ServiceAccount:
Namespace:
Exists:
YES / NO
```

Remember:

```text
ServiceAccount Exists ≠ Authorization Correct
```

---

## Step 10 — Check Whether the Workload Uses `default`

If the observed identity is:

```text
default
```

record it without immediately classifying it as a defect.

```text
default ServiceAccount ≠ Automatic Misconfiguration
```

Record:

```text
Observed ServiceAccount:
default

Expected:
YES / NO / UNKNOWN

Evidence:
```

Also consider the potential blast radius:

```text
Permission Granted to default ServiceAccount
        =
Potential Multi-Workload Blast Radius
```

---

# Phase 5 — Identify the Exact Failed Operation

## Step 11 — Build the Authorization Tuple

Record:

```text
Identity:

Verb:

API group:

Resource:

Subresource:
If applicable

Resource name:
If constrained

Namespace / cluster scope:

Expected result:

Observed result:
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

Production rules:

```text
RESOURCE MATCH ≠ OPERATION MATCH

ONE ALLOWED ACTION ≠ GENERAL AUTHORIZATION
```

---

# Phase 6 — Inspect Namespace RBAC

## Step 12 — List Roles

Run:

```bash
kubectl get roles -n <NAMESPACE>
```

Record only roles relevant to the investigation.

Do not assume a Role is relevant because its name resembles the application name.

```text
ROLE NAME ≠ AUTHORIZATION PROOF
```

---

## Step 13 — Inspect a Relevant Role

Run:

```bash
kubectl describe role <ROLE> -n <NAMESPACE>
```

Record:

```text
Role:
Namespace:

API groups:

Resources:

Resource names:
If present

Verbs:
```

Do not modify the Role.

---

## Step 14 — List RoleBindings

Run:

```bash
kubectl get rolebindings -n <NAMESPACE>
```

Identify only bindings relevant to the target identity.

---

## Step 15 — Inspect a Relevant RoleBinding

Run:

```bash
kubectl describe rolebinding <ROLE_BINDING> \
  -n <NAMESPACE>
```

Record:

```text
RoleBinding:
Namespace:

Subject kind:
Subject name:
Subject namespace:

roleRef kind:
roleRef name:
```

Verify all identity fields.

Production rule:

```text
BINDING EXISTS ≠ CORRECT IDENTITY BOUND
```

---

# Phase 7 — RoleBinding to ClusterRole

## Step 16 — Determine the roleRef Type

If the RoleBinding references:

```text
kind: ClusterRole
```

record:

```text
RoleBinding namespace:
ClusterRole:
```

Remember:

```text
ClusterRole Referenced ≠ Cluster-Wide Grant
```

A RoleBinding that references a ClusterRole grants the applicable namespaced permissions through that RoleBinding's namespace.

Do not confuse this with a ClusterRoleBinding.

---

## Step 17 — Inspect the Referenced ClusterRole

Run:

```bash
kubectl describe clusterrole <CLUSTER_ROLE>
```

Record only relevant rules:

```text
ClusterRole:

Relevant API group:

Relevant resource:

Relevant subresource:

Relevant resourceNames:

Relevant verb:
```

Do not interpret:

```text
ClusterRole
```

as:

```text
cluster-admin
```

Production rule:

```text
ClusterRole ≠ Cluster Administrator
```

---

# Phase 8 — ClusterRoleBinding Investigation

Only perform this phase when evidence indicates a ClusterRoleBinding may be relevant.

## Step 18 — Identify Relevant ClusterRoleBindings

If you already know the binding name:

```bash
kubectl get clusterrolebinding <CLUSTER_ROLE_BINDING>
```

Then inspect it:

```bash
kubectl describe clusterrolebinding <CLUSTER_ROLE_BINDING>
```

Record:

```text
ClusterRoleBinding:

Subject kind:
Subject name:
Subject namespace:

roleRef:
```

Do not dump every cluster-wide binding unless the investigation requires it.

Production principle:

```text
TARGETED
EVIDENCE-DRIVEN
MINIMUM NECESSARY SCOPE
```

---

## Step 19 — Assess Cluster-Wide Impact

A ClusterRoleBinding has cluster-wide binding scope.

Record:

```text
ClusterRoleBinding involved:
YES / NO

Potential authorization scope:

Relevant rules:
```

Do not infer exact privilege from the binding type alone.

Inspect the referenced ClusterRole rules.

---

# Phase 9 — Exact Authorization Check

## Step 20 — Test the Current Caller

For a safe current-user authorization question:

```bash
kubectl auth can-i <VERB> <RESOURCE> \
  -n <NAMESPACE>
```

Example:

```bash
kubectl auth can-i get pods -n <NAMESPACE>
```

Record:

```text
Tested identity:
CURRENT CALLER

Verb:
Resource:
Namespace:
Result:
Timestamp:
```

This proves only the tested operation.

```text
ONE ALLOWED ACTION ≠ GENERAL AUTHORIZATION
```

---

## Step 21 — Test the Exact Failed Operation

When the current caller is the identity relevant to the incident, test the exact failed operation.

Example:

```bash
kubectl auth can-i create jobs \
  -n <NAMESPACE>
```

Do not substitute:

```text
get pods
```

when the failed operation was:

```text
create jobs
```

Production rule:

```text
TEST THE FAILED OPERATION
NOT A CONVENIENT OPERATION
```

---

# Phase 10 — ServiceAccount Impersonation Boundary

## Step 22 — Understand `--as` Before Using It

An impersonated check such as:

```bash
kubectl auth can-i create jobs \
  --as=system:serviceaccount:<NAMESPACE>:<SERVICE_ACCOUNT> \
  -n <NAMESPACE>
```

introduces two authorization questions:

```text
Can the operator impersonate the target identity?

AND

Can the target identity perform the requested operation?
```

Do not assume the operator has impersonation permission.

---

## Step 23 — Optional Impersonated Check

Run this only when your organization's access model permits impersonation and the command is approved:

```bash
kubectl auth can-i <VERB> <RESOURCE> \
  --as=system:serviceaccount:<NAMESPACE>:<SERVICE_ACCOUNT> \
  -n <NAMESPACE>
```

Record:

```text
Impersonation attempted:
YES

Operator allowed to impersonate:
YES / NO / UNKNOWN

Target identity:

Tested operation:

Result:
YES / NO / UNKNOWN
```

If the operator cannot impersonate the target identity, record:

```text
Target ServiceAccount authorization:
UNKNOWN
```

Production rule:

```text
IMPERSONATION DENIED
        ≠
TARGET IDENTITY DENIED
```

Do not request broader impersonation rights solely to complete this lab.

---

# Phase 11 — Secret Safety

## Step 24 — Validate Without Reading Secret Values

If the incident involves Secret authorization, authorization can be reasoned about without retrieving Secret contents.

Do not run commands intended to decode or print Secret values.

Do not copy Secret values into evidence.

Record:

```text
Secret resource involved:
YES / NO

Secret value accessed:
NO

Secret value exposed:
NO
```

Production rules:

```text
SECRET EXISTS ≠ SECRET SHOULD BE DECODED

AUTHORIZATION VALIDATION ≠ DATA EXTRACTION
```

---

# Phase 12 — ServiceAccount Token Safety

## Step 25 — Do Not Extract Tokens

Do not search for, decode, create, or print ServiceAccount tokens as part of this RBAC investigation.

Record:

```text
ServiceAccount token extracted:
NO

ServiceAccount token created:
NO
```

Remember:

```text
SERVICEACCOUNT EXISTS
        ≠
LONG-LIVED TOKEN SECRET MUST EXIST
```

and:

```text
RBAC TROUBLESHOOTING
        ≠
SERVICEACCOUNT TOKEN EXTRACTION
```

---

# Phase 13 — Platform Identity vs Workload Identity

## Step 26 — Determine Which Identity Actually Failed

If the application Pod was never created, determine whether the failed operation belonged to a platform component.

Possible chain:

```text
TrueFoundry Control Plane
        ↓
tfy-agent / platform integration
        ↓
Kubernetes API
        ↓
RBAC
        ↓
Workload creation
```

Record:

```text
Application Pod exists:
YES / NO

Failed request made by:
Platform identity / Workload identity / Human / UNKNOWN

Evidence:
```

Do not inspect only the intended workload ServiceAccount when the actual failing request came from the platform integration path.

```text
PLATFORM IDENTITY ≠ WORKLOAD IDENTITY
```

---

# Phase 14 — `tfy-agent` / Platform RBAC

## Step 27 — Discover Actual Installed Identity

Do not assume a fixed `tfy-agent` ServiceAccount or RBAC object name across every installation.

Use deployment evidence to identify the actual platform component.

If the relevant namespace and workload are already known:

```bash
kubectl get pods -n <platform-namespace> -o wide
```

Then, for the relevant Pod:

```bash
kubectl get pod <platform-pod> \
  -n <platform-namespace> \
  -o jsonpath='{.spec.serviceAccountName}{"\n"}'
```

Record:

```text
Platform component:
Platform namespace:
Platform Pod:
Observed platform ServiceAccount:
```

Production rule:

```text
DISCOVER ACTUAL INSTALLED RBAC
```

Do not hard-code one universal TrueFoundry RBAC layout.

---

## Step 28 — Separate Availability From Authorization

A platform agent can be running while lacking permission for a requested operation.

Record separately:

```text
Platform Pod running:
YES / NO / UNKNOWN

Control-plane connectivity:
PROVEN / SUPPORTED / UNKNOWN

Requested Kubernetes operation authorized:
YES / NO / UNKNOWN
```

Keep:

```text
Agent Running ≠ Agent Authorized for Requested Operation
```

---

# Phase 15 — Cloud IAM Boundary

## Step 29 — Determine Whether Kubernetes Authorization Is the Final Boundary

If the Kubernetes operation succeeds but the application cannot access a cloud resource, stop treating Kubernetes RBAC as the only authorization system.

Record:

```text
Kubernetes authorization:
ALLOWED / DENIED / UNKNOWN

Cloud resource involved:
YES / NO

Cloud identity:
If known

Cloud operation:

Cloud authorization result:
ALLOWED / DENIED / UNKNOWN
```

Keep:

```text
Kubernetes ServiceAccount ≠ Cloud IAM Principal
```

and:

```text
Kubernetes RBAC Success ≠ Cloud IAM Success
```

Do not modify Kubernetes RBAC to address a proven cloud-IAM failure.

---

# Phase 16 — Shared ServiceAccount Blast Radius

## Step 30 — Identify Workloads Using the ServiceAccount

When evaluating a possible ServiceAccount authorization change, determine whether the identity is shared.

A targeted read-only approach can inspect workload specifications in the namespace.

Examples:

```bash
kubectl get deployments -n <NAMESPACE> \
  -o custom-columns='NAME:.metadata.name,SERVICEACCOUNT:.spec.template.spec.serviceAccountName'
```

```bash
kubectl get statefulsets -n <NAMESPACE> \
  -o custom-columns='NAME:.metadata.name,SERVICEACCOUNT:.spec.template.spec.serviceAccountName'
```

```bash
kubectl get daemonsets -n <NAMESPACE> \
  -o custom-columns='NAME:.metadata.name,SERVICEACCOUNT:.spec.template.spec.serviceAccountName'
```

For scheduled workloads:

```bash
kubectl get cronjobs -n <NAMESPACE> \
  -o custom-columns='NAME:.metadata.name,SERVICEACCOUNT:.spec.jobTemplate.spec.template.spec.serviceAccountName'
```

For Jobs:

```bash
kubectl get jobs -n <NAMESPACE> \
  -o custom-columns='NAME:.metadata.name,SERVICEACCOUNT:.spec.template.spec.serviceAccountName'
```

Record:

```text
Target ServiceAccount:

Observed workload consumers:

Shared:
YES / NO / UNKNOWN
```

Remember:

```text
SERVICEACCOUNT CHANGE
        ≠
SINGLE-WORKLOAD CHANGE
```

unless proven.

---

# Phase 17 — Shared Role Blast Radius

## Step 31 — Identify Binding Consumers Before Recommending a Role Change

If a Role or ClusterRole appears to require modification, determine whether it is referenced by more than one binding.

For namespace RoleBindings:

```bash
kubectl get rolebindings -n <NAMESPACE> \
  -o custom-columns='NAME:.metadata.name,ROLEKIND:.roleRef.kind,ROLENAME:.roleRef.name'
```

When cluster-scope evidence is required:

```bash
kubectl get clusterrolebindings \
  -o custom-columns='NAME:.metadata.name,ROLEKIND:.roleRef.kind,ROLENAME:.roleRef.name'
```

Record:

```text
Role / ClusterRole:

Known bindings:

Shared:
YES / NO / UNKNOWN
```

Remember:

```text
ROLE CHANGE
        ≠
SINGLE-BINDING CHANGE
```

---

# Phase 18 — System-Managed RBAC Safety

## Step 32 — Identify Potential System-Managed Objects

If a relevant RBAC object uses a `system:`-style name, treat it as potentially Kubernetes-managed.

Record:

```text
Potential system-managed RBAC:
YES / NO

Object:
```

Do not modify it.

Production rule:

```text
DO NOT MODIFY SYSTEM-MANAGED RBAC
AS INCIDENT TROUBLESHOOTING
```

---

# Phase 19 — Desired-State Ownership

## Step 33 — Determine the Authoritative Configuration Owner

Use metadata and organizational evidence to determine whether RBAC is managed by:

```text
TrueFoundry
Helm
Argo CD
GitOps
Terraform / OpenTofu
Kubernetes Operator
Platform automation
Manual configuration
UNKNOWN
```

Useful metadata inspection:

```bash
kubectl get rolebinding <ROLE_BINDING> \
  -n <NAMESPACE> \
  -o yaml
```

or:

```bash
kubectl get role <ROLE> \
  -n <NAMESPACE> \
  -o yaml
```

Use output only for metadata/configuration inspection.

Do not expose sensitive information in evidence.

Record:

```text
Observed RBAC object:

Management indicators:

Authoritative desired-state owner:

Confidence:
PROVEN / SUPPORTED / UNKNOWN / ASSUMED
```

Production rule:

```text
DO NOT PATCH RBAC DIRECTLY
WHEN ANOTHER SYSTEM OWNS DESIRED STATE
```

---

# Phase 20 — Authorization Drift

## Step 34 — Compare Expected and Observed State

Record:

```text
Expected identity:
Observed identity:

Expected binding:
Observed binding:

Expected roleRef:
Observed roleRef:

Expected operation:
Observed authorization:

Expected namespace/scope:
Observed namespace/scope:

Expected cloud identity:
Observed cloud identity:
If applicable
```

Classify differences:

```text
PROVEN
SUPPORTED
UNKNOWN
ASSUMED
```

Remember:

```text
Observed Drift ≠ Permission to Patch
```

---

# Phase 21 — Effective Privilege Awareness

## Step 35 — Identify Security-Sensitive Capabilities

Without changing anything, determine whether the relevant authorization includes potentially sensitive capabilities such as:

```text
Secret access
RBAC modification
Identity impersonation
Workload creation
Use of privileged workload identities
```

Record only what is relevant to the incident.

```text
Sensitive capability identified:
YES / NO / UNKNOWN

Capability:

Evidence:
```

Remember:

```text
DIRECT PERMISSION
        ≠
COMPLETE EFFECTIVE PRIVILEGE
```

Do not perform privilege-escalation tests in production.

---

# Phase 22 — Lowest Proven Healthy Layer

## Step 36 — Identify the Last Proven Healthy Layer

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
```

Do not revisit already-proven layers without new evidence.

---

# Phase 23 — First Failed Transition

## Step 37 — Identify the First Failure Boundary

Example:

```text
Kubernetes request
        ↓
Authorization evaluation
        X
```

Record:

```text
First Failed Transition:
```

Possible examples:

```text
TrueFoundry request → platform authorization

Platform identity → Kubernetes RBAC

Workload ServiceAccount → Kubernetes RBAC

Kubernetes workload identity → cloud identity

Cloud identity → cloud IAM authorization
```

---

# Phase 24 — Minimum Supported Blast Radius

## Step 38 — Determine the Smallest Proven Scope

Choose the smallest scope supported by evidence:

```text
Single operation
Single identity
Single workload
Shared ServiceAccount
Single namespace
Multiple namespaces
Cluster-wide
UNKNOWN
```

Record:

```text
Minimum Supported Blast Radius:

Evidence:
```

Do not generalize:

```text
model-api-sa cannot create Jobs
```

into:

```text
Cluster RBAC is broken.
```

---

# Phase 25 — Current Actionable Owner

## Step 39 — Map Failure Domain to Ownership

Record:

```text
Symptom:

Requesting identity:

Failed operation:

Authorization boundary:

Authorization result:

Authoritative configuration owner:

Current Actionable Owner:

Requested Action:
```

Keep:

```text
SYMPTOM ≠ FAILURE DOMAIN ≠ OWNER
```

---

# Phase 26 — Evidence Confidence

## Step 40 — Classify Every Important Finding

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
Cloud IAM problem           UNKNOWN
TrueFoundry defect          ASSUMED
```

Production rule:

```text
ASSUMED ≠ ROOT CAUSE
```

---

# Phase 27 — Authorization Evidence Handoff

Complete:

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

ServiceAccount token created:
NO

Authorization mutation performed:
NO
```

---

# Phase 28 — Acceptance Checklist

The lab passes when all applicable statements are true:

```text
[ ] Kubernetes context verified

[ ] Cluster connectivity verified

[ ] Namespace verified

[ ] Original authorization error preserved

[ ] Requesting component identified or explicitly UNKNOWN

[ ] Identity type classified

[ ] Actual identity proven or explicitly UNKNOWN

[ ] Platform identity distinguished from workload identity

[ ] Observed Pod ServiceAccount verified when applicable

[ ] Exact failed operation documented

[ ] Verb identified

[ ] API group identified when applicable

[ ] Resource identified

[ ] Subresource identified when applicable

[ ] Namespace / cluster scope identified

[ ] Relevant binding inspected

[ ] Relevant Role / ClusterRole inspected

[ ] RoleBinding vs ClusterRoleBinding scope understood

[ ] Exact authorization operation tested when permitted

[ ] One successful operation was not generalized to all permissions

[ ] Impersonation boundary handled correctly

[ ] Failed impersonation was not classified as target identity denial

[ ] Kubernetes RBAC separated from TrueFoundry authorization

[ ] Kubernetes RBAC separated from cloud IAM

[ ] No Secret value decoded

[ ] No Secret value printed

[ ] No ServiceAccount token extracted

[ ] No ServiceAccount token created

[ ] No temporary cluster-admin granted

[ ] No Role modified

[ ] No ClusterRole modified

[ ] No RoleBinding modified

[ ] No ClusterRoleBinding modified

[ ] No ServiceAccount modified

[ ] No system-managed RBAC modified

[ ] Shared ServiceAccount consumers assessed when relevant

[ ] Shared Role / ClusterRole consumers assessed when relevant

[ ] Desired-state owner identified or explicitly UNKNOWN

[ ] No observed drift was patched directly

[ ] Lowest Proven Healthy Layer documented

[ ] First Failed Transition documented

[ ] Minimum Supported Blast Radius documented

[ ] Evidence Confidence documented

[ ] Current Actionable Owner documented

[ ] Requested Action documented

[ ] Authorization mutation performed: NO
```

---

# Production Investigation Flow

```text
CAPTURE ORIGINAL FAILURE
        ↓
VERIFY CLUSTER / CONTEXT / NAMESPACE
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
TEST EXACT OPERATION
WHEN PERMITTED
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
ASSESS SHARED IDENTITY / ROLE CONSUMERS
        ↓
CHECK NEXT AUTHORIZATION BOUNDARY
WHEN APPLICABLE
        ↓
IDENTIFY LOWEST PROVEN HEALTHY LAYER
        ↓
IDENTIFY FIRST FAILED TRANSITION
        ↓
DETERMINE MINIMUM SUPPORTED BLAST RADIUS
        ↓
IDENTIFY AUTHORITATIVE CONFIGURATION OWNER
        ↓
IDENTIFY CURRENT ACTIONABLE OWNER
        ↓
HAND OFF EVIDENCE
```

---

# Final Production Rules

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

## Lab Completion Standard

This lab is complete when the operator can produce defensible authorization evidence without changing production authorization state.

The final evidence must answer:

```text
WHAT operation failed?

WHO actually attempted it?

WHICH authorization boundary evaluated it?

WHAT exact permission was required?

WHAT result was observed?

WHERE is the first failed transition?

WHAT is the minimum supported blast radius?

WHO owns authoritative desired state?

WHO is the current actionable owner?

WERE Secret values exposed?
NO

WAS a ServiceAccount token extracted or created?
NO

WAS production authorization mutated?
NO
```
