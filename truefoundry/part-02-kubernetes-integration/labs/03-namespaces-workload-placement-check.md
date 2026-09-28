# Part 2.3 — Namespaces & Workload Placement — Hands-On Validation Lab

> **[SAFE-READ] Production Validation Lab**
>
> This lab is designed for production-safe, read-only investigation. It does not authorize changes to namespaces, workloads, policies, controllers, or credentials.

## 1. Lab Objective

Validate the placement path of a TrueFoundry workload from platform context to Kubernetes runtime:

```text
TrueFoundry Workspace
        ↓
Associated Kubernetes cluster
        ↓
Target Kubernetes namespace
        ↓
Namespace authorization / policy
        ↓
Workload controller
        ↓
Pod creation / admission
        ↓
Pod
        ↓
Scheduler
        ↓
Node
        ↓
Container / readiness
```

The lab is complete when you can identify:

```text
Environment
Workspace
Cluster
Kubernetes context
Namespace
Namespace phase
Controller
Desired replicas
Created Pods
Scheduled Pods
Ready Pods
ServiceAccount
Node placement
Lowest Proven Healthy Layer
First Failed Transition
Minimum Supported Blast Radius
Authoritative Configuration Owner
Current Actionable Owner
Requested Action
```

---

## 2. Safety Rules

This lab permits read-only commands such as:

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
kubectl scale
kubectl rollout restart
helm install
helm upgrade
helm uninstall
```

Do not decode or print Secret values.

Production rules:

```text
UNKNOWN PLACEMENT = NO MUTATION

SECRET EXISTS ≠ SECRET SHOULD BE DECODED

OBSERVED DRIFT ≠ PERMISSION TO PATCH

NAMESPACE CHANGE = BLAST-RADIUS EVENT
```

---

## 3. Prepare the Evidence Record

Before running commands, create an evidence record:

```text
Environment:
TrueFoundry Workspace:

Expected cluster:
Expected namespace:

Operator:
Observation start time:
Incident/change reference:
```

If the expected cluster or namespace is unknown, record:

```text
UNKNOWN
```

Do not convert assumptions into facts.

Evidence confidence used throughout this lab:

```text
PROVEN
SUPPORTED
UNKNOWN
ASSUMED
```

---

## 4. Gate 1 — Verify Kubernetes Context

Run:

```bash
kubectl config current-context
```

Record:

```text
Current Kubernetes context:
Evidence confidence:
```

Then:

```bash
kubectl cluster-info
```

Record:

```text
Kubernetes API reachable:
YES / NO

Observed cluster/API identity:
Evidence confidence:
```

Production rule:

```text
VALID KUBERNETES OUTPUT ≠ CORRECT INVESTIGATION TARGET
```

If the context cannot be tied to the intended environment:

```text
STOP MUTATION

UNKNOWN TARGET = NO MUTATION
```

Continue only with safe discovery if authorized.

---

## 5. Verify Node Visibility

Run:

```bash
kubectl get nodes
```

Record:

```text
Nodes visible:
YES / NO

Ready nodes:
NotReady nodes:

Cluster visibility confidence:
```

This does not prove workload placement. It confirms that the current identity can observe relevant cluster state.

---

## 6. Discover Namespaces

Run:

```bash
kubectl get namespaces
```

Record:

```text
Expected namespace found:
YES / NO / UNKNOWN

Candidate namespace:
Evidence:
Evidence confidence:
```

Do not choose a namespace solely because its name resembles the TrueFoundry Workspace.

Production rule:

```text
WORKSPACE NAME ≠ PROOF OF ACTUAL NAMESPACE
```

---

## 7. Verify Namespace State

For the verified or candidate namespace:

```bash
kubectl get namespace <namespace>
```

Optionally inspect metadata:

```bash
kubectl describe namespace <namespace>
```

Record:

```text
Namespace:
Phase:
Labels relevant to investigation:
Annotations relevant to investigation:

Namespace operational:
PROVEN / SUPPORTED / UNKNOWN
```

Remember:

```text
Namespace Exists ≠ Namespace Operational
```

A `Terminating` namespace is materially different from an `Active` namespace.

---

## 8. Verify Workspace-to-Placement Evidence

Using approved TrueFoundry/platform evidence available to you, record:

```text
TrueFoundry Workspace:
Associated cluster:
Expected Kubernetes namespace:

Source of mapping evidence:
Timestamp:
Evidence confidence:
```

Do not infer the namespace from naming alone.

Acceptance rule:

```text
Workspace association:
PROVEN / SUPPORTED / UNKNOWN

Namespace mapping:
PROVEN / SUPPORTED / UNKNOWN
```

If the mapping remains uncertain:

```text
UNKNOWN PLACEMENT = NO MUTATION
```

---

## 9. Discover the Workload Across Namespaces

If the workload name is known:

Linux/macOS shell:

```bash
kubectl get pods -A | grep -i <workload>
```

Windows CMD:

```bat
kubectl get pods -A | findstr /I "<workload>"
```

Record:

```text
Workload search term:
Matching namespaces:
Matching Pods:
```

Do not stop after finding the first similarly named resource.

Duplicate names can exist across namespaces and clusters.

---

## 10. Discover Workload Controllers

Run the relevant read-only discovery commands:

```bash
kubectl get deployments -A
kubectl get statefulsets -A
kubectl get daemonsets -A
kubectl get jobs -A
kubectl get cronjobs -A
```

Filter locally when needed.

Record:

```text
Controller kind:
Controller name:
Namespace:
Evidence confidence:
```

Resource identity should be treated as:

```text
Cluster + Namespace + Kind + Name
```

not simply:

```text
Name
```

---

## 11. Verify Controller Details

For a Deployment:

```bash
kubectl get deployment <deployment> -n <namespace>
kubectl describe deployment <deployment> -n <namespace>
```

For a StatefulSet:

```bash
kubectl get statefulset <statefulset> -n <namespace>
kubectl describe statefulset <statefulset> -n <namespace>
```

For another controller, use the equivalent read-only command.

Record:

```text
Controller:
Desired replicas:
Observed/current replicas:
Ready replicas:
Available replicas:
Relevant conditions:
Relevant events:
```

Do not interpret one field in isolation.

---

## 12. Compare Desired vs Created State

Record:

```text
Desired replicas:
Created Pods:
Difference:
```

Classify:

```text
Desired = Created:
YES / NO / UNKNOWN
```

Production rule:

```text
Desired Replicas ≠ Created Pods
```

If desired replicas exceed created Pods, investigate controller and admission evidence before assuming a scheduler problem.

---

## 13. Discover Pods in the Verified Namespace

Run:

```bash
kubectl get pods -n <namespace> -o wide
```

Record each relevant Pod:

```text
Pod:
Phase:
Ready:
Restarts:
Node:
Age:
```

For replicated workloads, inspect all relevant Pods.

Do not classify the whole workload from one replica.

---

## 14. Verify Pod Ownership

For each affected Pod:

```bash
kubectl get pod <pod> \
  -n <namespace> \
  -o jsonpath='{.metadata.ownerReferences[*].kind}{" "}{.metadata.ownerReferences[*].name}{"\n"}'
```

Record:

```text
Pod:
Immediate owner kind:
Immediate owner name:
```

Where appropriate, continue the ownership chain.

Example:

```text
Deployment
    ↓
ReplicaSet
    ↓
Pod
```

Production rule:

```text
POD ≠ WORKLOAD DEFINITION
```

---

## 15. Identify the Desired-State Owner

Use read-only metadata and your deployment architecture to determine whether the workload is reconciled by something such as:

```text
TrueFoundry
Helm
Argo CD / GitOps
Kubernetes Operator
another controller
```

Record:

```text
Observed Kubernetes controller:
Higher-level reconciliation owner:
Authoritative configuration source:
Evidence:
Evidence confidence:
```

If unknown, record:

```text
UNKNOWN
```

Do not patch the observed resource.

```text
DO NOT PATCH THE SYMPTOM
WHEN ANOTHER SYSTEM OWNS DESIRED STATE
```

---

## 16. Inspect Pod Conditions

Run:

```bash
kubectl describe pod <pod> -n <namespace>
```

Record:

```text
Pod:
Phase:

PodScheduled:
Initialized:
ContainersReady:
Ready:

Node:
Relevant reason:
Relevant message:
```

Key boundaries:

```text
Pod Exists ≠ Pod Scheduled

Pod Scheduled ≠ Container Started

Container Started ≠ Ready
```

---

## 17. Distinguish Pod Not Created From Pod Pending

### Case A — Expected Pod does not exist

Record:

```text
Controller exists:
YES / NO

Expected replica missing:
YES / NO

Pod object found:
NO
```

Investigate:

```text
controller conditions
controller events
namespace quota
namespace limits
admission evidence
```

Do not classify this as scheduling failure.

### Case B — Pod exists but is Pending

Record:

```text
Pod object found:
YES

Phase:
Pending

PodScheduled:
TRUE / FALSE / UNKNOWN
```

Now scheduling evidence becomes relevant.

Production rule:

```text
POD NOT CREATED ≠ POD PENDING
```

---

## 18. Inspect Timestamped Namespace Events

Run:

```bash
kubectl get events \
  -n <namespace> \
  --sort-by=.metadata.creationTimestamp
```

Record relevant evidence:

```text
Timestamp:
Object:
Reason:
Message summary:
Related placement gate:
```

Potential categories:

```text
controller reconciliation
admission
quota
scheduler
volume
image
startup
probe
```

Do not automatically convert an event message into root cause.

```text
EVENT MESSAGE ≠ AUTOMATIC ROOT CAUSE
```

---

## 19. Check ResourceQuota

Run:

```bash
kubectl get resourcequota -n <namespace>
```

If quotas exist:

```bash
kubectl describe resourcequota -n <namespace>
```

Record:

```text
ResourceQuota present:
YES / NO

Relevant quota:
Used:
Hard:
Potential quota constraint:
PROVEN / SUPPORTED / UNKNOWN
```

Do not change quota during this lab.

Remember:

```text
Namespace Quota Available ≠ Cluster Capacity Available

Cluster Capacity Available ≠ Namespace Quota Available
```

---

## 20. Check LimitRange

Run:

```bash
kubectl get limitrange -n <namespace>
```

If present:

```bash
kubectl describe limitrange <name> -n <namespace>
```

Record:

```text
LimitRange present:
YES / NO

Relevant constraints/defaults:
Potential admission relevance:
PROVEN / SUPPORTED / UNKNOWN
```

Do not assume a newly changed LimitRange retroactively changed existing Pods.

---

## 21. Check NetworkPolicy Metadata

Run:

```bash
kubectl get networkpolicy -n <namespace>
```

If relevant:

```bash
kubectl describe networkpolicy <name> -n <namespace>
```

Record:

```text
NetworkPolicies present:
YES / NO

Potential relevance:
PROVEN / SUPPORTED / UNKNOWN
```

This lab does not perform deep network troubleshooting. That is covered later in the curriculum.

---

## 22. Check Secret and ConfigMap Metadata Safely

Run:

```bash
kubectl get configmaps -n <namespace>
kubectl get secrets -n <namespace>
```

Record only metadata needed for placement/context investigation.

Do not run commands intended to decode Secret data.

Production rule:

```text
SECRET EXISTS ≠ SECRET SHOULD BE DECODED
```

Record:

```text
Relevant ConfigMap metadata:
Relevant Secret metadata:

Secret values exposed:
NO
```

---

## 23. Identify the Pod ServiceAccount

Run:

```bash
kubectl get pod <pod> \
  -n <namespace> \
  -o jsonpath='{.spec.serviceAccountName}{"\n"}'
```

Record:

```text
Pod:
ServiceAccount:
```

Then verify metadata:

```bash
kubectl get serviceaccount <service-account> -n <namespace>
```

Record:

```text
ServiceAccount exists:
YES / NO
```

Remember:

```text
ServiceAccount Exists ≠ Authorization Correct
```

Detailed RBAC validation belongs in Part 2.4.

---

## 24. Optional Safe RBAC Check

When the required operation is known and you are authorized to perform impersonation checks, use read-only authorization evaluation.

Example:

```bash
kubectl auth can-i get pods \
  --as=system:serviceaccount:<namespace>:<service-account> \
  -n <namespace>
```

For an operation-specific investigation:

```bash
kubectl auth can-i <verb> <resource> \
  --as=system:serviceaccount:<namespace>:<service-account> \
  -n <namespace>
```

Record:

```text
Identity tested:
Verb:
Resource:
Namespace:
Result:
```

Do not generalize one successful permission check into proof that all required permissions are correct.

---

## 25. Check Labels

Run:

```bash
kubectl get pod <pod> \
  -n <namespace> \
  --show-labels
```

Record relevant labels:

```text
Application:
Component:
Release:
Platform metadata:
Other relevant labels:
```

Production rule:

```text
LABEL ≠ AUTOMATIC AUTHORITATIVE PROOF
```

---

## 26. Check Annotations

Run:

```bash
kubectl get pod <pod> \
  -n <namespace> \
  -o jsonpath='{.metadata.annotations}{"\n"}'
```

Review only non-sensitive metadata.

Record:

```text
Relevant controller metadata:
Deployment-tool metadata:
Policy metadata:
Platform metadata:
```

Production rule:

```text
ANNOTATION ≠ AUTOMATIC AUTHORITATIVE PROOF
```

---

## 27. Determine Scheduling State

From `kubectl describe pod` record:

```text
PodScheduled:
TRUE / FALSE / UNKNOWN

Assigned node:
```

If:

```text
PodScheduled=False
```

record relevant scheduler events.

Potential evidence can include:

```text
Insufficient CPU
Insufficient memory
Insufficient GPU
node selector mismatch
node affinity mismatch
untolerated taint
topology constraint
PVC-related scheduling dependency
```

Do not perform scheduling mutations in this lab.

---

## 28. Inspect Node Placement

For scheduled Pods:

```bash
kubectl get pod <pod> -n <namespace> -o wide
```

Record:

```text
Pod:
Node:
Pod IP:
```

When node investigation is relevant:

```bash
kubectl describe node <node>
```

Record:

```text
Node Ready:
Relevant taints:
Relevant labels:
Relevant pressure conditions:
```

Production rule:

```text
Namespace Placement ≠ Node Placement
```

---

## 29. Compare Created, Scheduled, and Ready Replicas

Create the following summary:

```text
Desired replicas:
Created Pods:
Scheduled Pods:
Ready Pods:
```

Example:

```text
Desired:      4
Created:      4
Scheduled:    3
Ready:        3
```

Classification:

```text
Materialization:
HEALTHY / DEGRADED / FAILED / UNKNOWN

Scheduling:
HEALTHY / DEGRADED / FAILED / UNKNOWN

Readiness:
HEALTHY / DEGRADED / FAILED / UNKNOWN
```

Do not collapse partial degradation into a binary healthy/down conclusion.

---

## 30. Identify Admission vs Scheduling Boundary

Use the evidence collected so far.

### Admission-side pattern

```text
Controller             PROVEN
Pod creation attempt   PROVEN
Expected Pod           NOT CREATED
Quota/policy rejection PROVEN
```

Record:

```text
First Failed Transition:
Pod creation request → admission
```

### Scheduling-side pattern

```text
Pod object        PROVEN
Admission         PROVEN
PodScheduled      FALSE
Scheduler event   PROVEN
```

Record:

```text
First Failed Transition:
Pod creation/admission → scheduler placement
```

This distinction is mandatory for lab acceptance.

---

## 31. Check for Placement Drift

Compare expected and observed values:

```text
Expected Workspace:
Observed Workspace context:

Expected cluster:
Observed cluster:

Expected namespace:
Observed namespace:

Expected controller:
Observed controller:

Expected ServiceAccount:
Observed ServiceAccount:

Expected image/version:
Observed image/version:

Expected node-placement constraints:
Observed constraints:
```

Classify:

```text
Drift:
PROVEN / SUPPORTED / UNKNOWN / NOT OBSERVED
```

Production rule:

```text
OBSERVED DRIFT ≠ PERMISSION TO PATCH
```

---

## 32. Check Cross-Namespace Dependencies

Using approved architecture/deployment evidence and safe Kubernetes discovery, record known dependencies:

```text
Workload namespace:

Cross-namespace service dependencies:
Shared ingress dependencies:
Shared platform dependencies:
External dependencies:

Evidence confidence:
```

Do not assume:

```text
Workload Namespace = Complete Dependency Boundary
```

Production rule:

```text
Workload Namespace ≠ Dependency Boundary
```

---

## 33. Check Shared Cluster Infrastructure Relevance

Determine whether the symptom could involve shared infrastructure such as:

```text
DNS
CNI
CSI
Ingress controller
admission webhook
scheduler
node pool
Kubernetes API
```

Record:

```text
Shared component:
Evidence:
Potential scope:
Evidence confidence:
```

Remember:

```text
Symptom in Namespace ≠ Failure Originates in Namespace
```

---

## 34. Determine the Lowest Proven Healthy Layer

Use the evidence chain:

```text
Environment
    ↓
Workspace
    ↓
Cluster
    ↓
Namespace
    ↓
Namespace policy/access
    ↓
Controller
    ↓
Pod creation request
    ↓
Admission
    ↓
Pod
    ↓
Scheduler
    ↓
Node
    ↓
Container
    ↓
Readiness
```

Record:

```text
Lowest Proven Healthy Layer:

Evidence:
Timestamp:
Confidence:
```

Example:

```text
Lowest Proven Healthy Layer:
Pod admission / creation

Evidence:
Pod object exists, but PodScheduled=False.

Confidence:
PROVEN
```

---

## 35. Determine the First Failed Transition

Record:

```text
First Failed Transition:

From:
To:

Evidence:
Timestamp:
Confidence:
```

Examples:

```text
Pod creation request → admission
```

or:

```text
Pod creation/admission → scheduler placement
```

or:

```text
Node placement → container startup
```

Avoid vague statements such as:

```text
Deployment failed.
```

---

## 36. Determine the Minimum Supported Blast Radius

Evaluate the smallest scope supported by evidence:

```text
One container
One Pod
One replica
One controller
One workload
One namespace
One node
One node pool
One cluster
Multiple namespaces
Multiple clusters
```

Record:

```text
Minimum Supported Blast Radius:

Evidence:
Confidence:
```

Production rule:

```text
One Pod Pending ≠ Namespace Scheduling Failure

One Pod Pending ≠ Cluster Scheduling Failure
```

---

## 37. Classify Evidence Confidence

For each important conclusion:

```text
Finding:
Evidence:
Confidence:
PROVEN / SUPPORTED / UNKNOWN / ASSUMED
```

At minimum classify:

```text
Correct environment:
Correct cluster:
Correct namespace:
Controller ownership:
Pod creation:
Admission:
Scheduling:
Node placement:
Runtime/readiness:
Blast radius:
Failure domain:
```

Production rule:

```text
ASSUMED ≠ ROOT CAUSE
```

---

## 38. Identify the Current Actionable Owner

Do not assign ownership from the visible symptom alone.

Record:

```text
Symptom:

Lowest Proven Healthy Layer:

First Failed Transition:

Failure domain:

Authoritative Configuration Owner:

Current Actionable Owner:

Evidence:

Requested Action:
```

Production rule:

```text
SYMPTOM ≠ FAILURE DOMAIN ≠ OWNER
```

---

## 39. Build the Evidence Handoff Contract

Complete:

```text
Environment:
TrueFoundry Workspace:

Cluster:
Kubernetes context:
Namespace:
Namespace phase:

Controller kind:
Controller name:

Desired replicas:
Created Pods:
Scheduled Pods:
Ready Pods:

Affected Pod:
Pod phase:
PodScheduled:
Node:

ServiceAccount:

ResourceQuota:
LimitRange:
NetworkPolicy relevance:

Relevant events:

Observation timestamp:

Lowest Proven Healthy Layer:

First Failed Transition:

Minimum Supported Blast Radius:

Evidence Confidence:

Authoritative Configuration Owner:

Current Actionable Owner:

Requested Action:

Sensitive information removed:
YES
```

The handoff must make clear:

```text
WHAT was proven?
WHAT remains unknown?
WHAT action is requested?
WHO can take that action?
```

---

## 40. Lab Acceptance Checklist

The lab passes when all applicable items are complete:

```text
[ ] Kubernetes context verified
[ ] Cluster/API identity verified
[ ] Workspace recorded
[ ] Actual namespace verified or explicitly UNKNOWN
[ ] Namespace phase checked
[ ] Workspace name was not treated as namespace proof
[ ] Workload controller discovered
[ ] Pod ownership chain inspected
[ ] Desired-state owner identified or explicitly UNKNOWN
[ ] Desired replicas recorded
[ ] Created Pods recorded
[ ] Scheduled Pods recorded
[ ] Ready Pods recorded
[ ] PodScheduled condition checked
[ ] Admission and scheduling treated as separate stages
[ ] ResourceQuota checked
[ ] LimitRange checked
[ ] NetworkPolicy metadata checked when relevant
[ ] Secret values were NOT decoded
[ ] ServiceAccount recorded
[ ] Node placement recorded when applicable
[ ] Timestamped events reviewed
[ ] Placement drift assessed
[ ] Cross-namespace dependencies considered
[ ] Shared cluster dependencies considered
[ ] Lowest Proven Healthy Layer recorded
[ ] First Failed Transition recorded
[ ] Minimum Supported Blast Radius recorded
[ ] Evidence confidence assigned
[ ] Authoritative Configuration Owner identified or UNKNOWN
[ ] Current Actionable Owner identified
[ ] Requested Action documented
[ ] No mutation commands executed
```

---

## 41. Final Safety Gate

Before closing the lab, verify:

```text
NO MUTATION

NO SECRET DECODING

NO CREDENTIAL EXPOSURE

NO ASSUMPTION PRESENTED AS FACT

NO NAMESPACE LIFECYCLE CHANGE

NO MANUAL PATCH TO CONTROLLER-OWNED STATE
```

Final operational rules:

```text
WORKSPACE NAME ≠ PROOF OF ACTUAL NAMESPACE

UNKNOWN PLACEMENT = NO MUTATION

Namespace Exists ≠ Namespace Operational

Namespace Exists ≠ Namespace Usable for Deployment

Correct Namespace ≠ Agent Authorized for Namespace

POD ≠ WORKLOAD DEFINITION

Deployment Exists ≠ Pod Creation Succeeded

POD NOT CREATED ≠ POD PENDING

Pod Exists ≠ Pod Scheduled

Pod Scheduled ≠ Container Started

Container Started ≠ Ready

Namespace Quota Available ≠ Cluster Capacity Available

Cluster Capacity Available ≠ Namespace Quota Available

Namespace Placement ≠ Node Placement

Namespace ≠ Complete Security Boundary

Workload Namespace ≠ Dependency Boundary

Symptom in Namespace ≠ Failure Originates in Namespace

SECRET EXISTS ≠ SECRET SHOULD BE DECODED

OBSERVED DRIFT ≠ PERMISSION TO PATCH

NAMESPACE CHANGE = BLAST-RADIUS EVENT

ASSUMED ≠ ROOT CAUSE

SYMPTOM ≠ FAILURE DOMAIN ≠ OWNER
```

## 42. Lab Completion Record

```text
Lab:
Part 2.3 — Namespaces & Workload Placement

Result:
PASS / FAIL / INCOMPLETE

Environment:

Workspace:

Cluster:

Namespace:

Lowest Proven Healthy Layer:

First Failed Transition:

Minimum Supported Blast Radius:

Current Actionable Owner:

Requested Action:

Evidence confidence:

Observation completed at:

Sensitive information removed:
YES

Mutation performed:
NO
```
