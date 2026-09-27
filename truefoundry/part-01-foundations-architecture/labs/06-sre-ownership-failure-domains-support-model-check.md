# Lab 1.6 — SRE Ownership, Failure Domains & Support Model

**Classification:** `[SAFE-READ]`  
**Track:** Part 1 — Foundations & Architecture  
**Tutorial:** Part 1.6 — SRE Ownership, Failure Domains & Support Model

---

## 1. Lab Purpose

This lab practices evidence-driven incident triage across TrueFoundry and Kubernetes.

The goal is not to repair or mutate a workload. The goal is to determine:

```text
WHAT is affected?
WHERE is the first proven failure?
WHAT action is required next?
WHO should own that action?
```

The lab reinforces:

```text
SYMPTOM ≠ FAILURE DOMAIN ≠ OWNER
```

---

## 2. Safety Boundary

This is a read-only production-safe investigation lab.

### Allowed

```text
kubectl config current-context
kubectl get
kubectl describe
kubectl logs
```

### Not Allowed

Do not run mutation commands such as:

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

Do not restart Pods, modify workloads, change replicas, edit Secrets, or change cluster configuration.

If the deployment identity or target cluster is uncertain:

```text
UNKNOWN TARGET = NO MUTATION
```

---

## 3. Prerequisites

You need:

- `kubectl`
- read-only access to the target Kubernetes cluster
- an existing TrueFoundry-connected Compute Plane
- permission to inspect namespaces, workloads, Pods, events, and logs
- the expected environment and workload identity

This lab does not require administrative permissions.

---

## 4. Define the Expected Deployment Identity

Record the expected values before querying the cluster.

```text
EXPECTED_ENVIRONMENT=
EXPECTED_COMPUTE_PLANE=
EXPECTED_WORKSPACE=
EXPECTED_NAMESPACE=
EXPECTED_WORKLOAD=
EXPECTED_ARTIFACT=
EXPECTED_DEPLOYMENT_VERSION=
```

Do not enter Secret values.

The expected identity is:

```text
Environment
 + Compute Plane
 + Workspace
 + Namespace
 + Workload
 + Artifact
 + Configuration
 + Deployment Version
 = Deployment Identity
```

---

## 5. Verify Kubernetes Context

Run:

```bash
kubectl config current-context
```

Record:

```text
Observed Context:
Expected Compute Plane / Cluster:
Match: YES / NO / UNKNOWN
```

### Acceptance

Continue only if you can establish that you are observing the intended cluster.

If the result is unexpected or unknown, stop the lab and record:

```text
Target validation failed.
No mutation performed.
```

---

## 6. Verify Kubernetes API Visibility

Run:

```bash
kubectl get nodes
```

Record:

```text
Kubernetes API reachable: YES / NO
Node visibility available: YES / NO
```

If nodes are visible, this provides positive evidence that the Kubernetes API is reachable from your current investigation context.

Do not conclude that all Kubernetes components or workloads are healthy solely from this command.

---

## 7. Verify Namespace

Run:

```bash
kubectl get namespaces
```

Then:

```bash
kubectl get namespace <EXPECTED_NAMESPACE>
```

Record:

```text
Expected Namespace:
Observed Namespace:
Namespace Exists: YES / NO
```

Remember:

```text
TrueFoundry Workspace ≠ Kubernetes Namespace
```

Do not infer Workspace identity from namespace name alone.

---

## 8. Discover TrueFoundry Cluster Integration

Use read-only discovery appropriate for your cluster.

Examples:

```bash
kubectl get pods -A | grep -i truefoundry
```

```bash
kubectl get pods -A | grep -i tfy
```

On Windows CMD, use:

```bat
kubectl get pods -A | findstr /I "truefoundry tfy"
```

If relevant components are found, inspect them:

```bash
kubectl get pods -n <INTEGRATION_NAMESPACE>
```

```bash
kubectl describe pod <INTEGRATION_POD> -n <INTEGRATION_NAMESPACE>
```

Record:

```text
Integration component discovered:
Namespace:
Pod:
Observed state:
```

Do not assume a particular namespace or component name if your environment differs.

---

## 9. Separate Management, Deployment, and Runtime Impact

For the incident or workload being examined, classify:

```text
Management Impact:
YES / NO / UNKNOWN

Deployment Impact:
YES / NO / UNKNOWN

Runtime Impact:
YES / NO / UNKNOWN
```

Do not infer runtime failure only because management or deployment behavior is unhealthy.

---

## 10. Locate the Workload

Inspect the expected namespace:

```bash
kubectl get deployments -n <EXPECTED_NAMESPACE>
```

```bash
kubectl get statefulsets -n <EXPECTED_NAMESPACE>
```

```bash
kubectl get jobs -n <EXPECTED_NAMESPACE>
```

```bash
kubectl get pods -n <EXPECTED_NAMESPACE> -o wide
```

Use only the workload types relevant to your environment.

Record:

```text
Expected Workload:
Observed Kubernetes Resource:
Resource Type:
Namespace:
Pod(s):
Node(s):
```

---

## 11. Establish the Minimum Supported Blast Radius

Inspect sibling Pods and workloads.

```bash
kubectl get pods -n <EXPECTED_NAMESPACE> -o wide
```

```bash
kubectl get deployments -n <EXPECTED_NAMESPACE>
```

Classify the smallest supported scope:

```text
Container
Pod
Workload
Namespace
Workspace
Compute Plane
Multiple Compute Planes
Platform
Unknown
```

Record:

```text
Minimum Supported Blast Radius:

Evidence:
```

Do not widen the reported impact beyond available evidence.

---

## 12. Inspect Workload Desired State

For a Deployment:

```bash
kubectl get deployment <WORKLOAD> -n <EXPECTED_NAMESPACE> -o wide
```

```bash
kubectl describe deployment <WORKLOAD> -n <EXPECTED_NAMESPACE>
```

For other workload types, use the equivalent read-only command.

Record:

```text
Desired replicas:
Available replicas:
Ready replicas:
Observed image:
ServiceAccount:
Relevant resource requests:
Relevant selectors:
```

Do not copy Secret values into the lab record.

---

## 13. Inspect Pod Observed State

Run:

```bash
kubectl get pods -n <EXPECTED_NAMESPACE> -o wide
```

Then inspect the relevant Pod:

```bash
kubectl describe pod <POD_NAME> -n <EXPECTED_NAMESPACE>
```

Record:

```text
Pod:
Phase:
Ready:
Restarts:
Node:
Image:
Reason:
Relevant Conditions:
```

Distinguish desired state from observed state.

---

## 14. Verify Artifact Identity

Inspect image references:

```bash
kubectl get pod <POD_NAME> -n <EXPECTED_NAMESPACE> -o jsonpath="{.spec.containers[*].image}"
```

Then:

```bash
kubectl get pod <POD_NAME> -n <EXPECTED_NAMESPACE> -o jsonpath="{.status.containerStatuses[*].imageID}"
```

Record:

```text
Expected Artifact:
Configured Image:
Observed Image ID / Digest:
Match: YES / NO / UNKNOWN
```

A running Pod does not prove that the correct artifact is deployed.

---

## 15. Inspect Configuration References Safely

Use:

```bash
kubectl describe pod <POD_NAME> -n <EXPECTED_NAMESPACE>
```

Inspect only references such as:

```text
ConfigMap names
Secret names
ServiceAccount
volume references
environment-variable source references
```

Do not retrieve or print Secret values.

Record:

```text
Configuration references observed:
Expected references:
Mismatch detected: YES / NO / UNKNOWN
```

---

## 16. Inspect Events

Run:

```bash
kubectl get events -n <EXPECTED_NAMESPACE> --sort-by=.lastTimestamp
```

Also review the Events section from:

```bash
kubectl describe pod <POD_NAME> -n <EXPECTED_NAMESPACE>
```

Look for evidence such as:

```text
FailedScheduling
FailedMount
FailedAttachVolume
FailedCreate
Failed
BackOff
Unhealthy
Pulling
Pulled
Created
Started
```

Record the exact relevant reason and timestamp.

Do not treat the Kubernetes reason alone as a complete root cause.

---

## 17. Inspect Application Logs

For a current container:

```bash
kubectl logs <POD_NAME> -n <EXPECTED_NAMESPACE>
```

If the Pod has multiple containers:

```bash
kubectl logs <POD_NAME> -n <EXPECTED_NAMESPACE> -c <CONTAINER_NAME>
```

If a container restarted and previous logs are available:

```bash
kubectl logs <POD_NAME> -n <EXPECTED_NAMESPACE> -c <CONTAINER_NAME> --previous
```

Record only sanitized evidence.

Never record:

```text
passwords
tokens
API keys
Secret values
credentials
private keys
```

---

## 18. Build an Evidence Timeline

Create a small timeline.

```text
Timestamp        Evidence
---------------  ---------------------------------------
                 Symptom first observed
                 Deployment/revision evidence
                 Kubernetes event
                 Container state change
                 Application error
                 Recovery observation
```

Remember:

```text
Change before failure ≠ Proven root cause
```

---

## 19. Classify Evidence Confidence

For each important observation, classify:

```text
PROVEN
SUPPORTED
UNKNOWN
ASSUMED
```

Example:

```text
PROVEN:
Pod scheduled to node X.

PROVEN:
Container started.

SUPPORTED:
Application initialization is failing.

UNKNOWN:
Exact configuration field causing failure.

ASSUMED:
Recent deployment caused the failure.
```

Do not promote an assumption to a fact.

---

## 20. Find the Lowest Proven Healthy Layer

Evaluate only layers for which you have evidence.

Example worksheet:

```text
Layer                               Result
----------------------------------  ----------------
TrueFoundry management path         PASS/FAIL/UNKNOWN
Cluster integration                 PASS/FAIL/UNKNOWN
Kubernetes API                      PASS/FAIL/UNKNOWN
Resource creation                   PASS/FAIL/UNKNOWN
Scheduling                          PASS/FAIL/UNKNOWN
Node                                PASS/FAIL/UNKNOWN
Container startup                   PASS/FAIL/UNKNOWN
GPU allocation                      PASS/FAIL/UNKNOWN
Application initialization          PASS/FAIL/UNKNOWN
Endpoint                            PASS/FAIL/UNKNOWN
External dependency                 PASS/FAIL/UNKNOWN
User transaction                    PASS/FAIL/UNKNOWN
```

Record:

```text
Lowest Proven Healthy Layer:
Evidence:
```

`UNKNOWN` is acceptable.

---

## 21. Find the First Failed Transition

Using the evidence collected, identify the first supported failure between two layers.

Examples:

```text
Scheduler → Eligible Node
Container → Application Initialization
GPU Allocation → Device Visibility
Endpoint → External Dependency
```

Record:

```text
First Failed Transition:
Evidence:
Confidence:
```

If insufficient evidence exists:

```text
First Failed Transition: UNKNOWN
```

Do not guess.

---

## 22. Optional GPU Investigation

Run this section only for a GPU workload.

First inspect node GPU capacity:

```bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,GPU:.status.allocatable.nvidia\.com/gpu
```

Inspect Pod resource requests:

```bash
kubectl describe pod <POD_NAME> -n <EXPECTED_NAMESPACE>
```

Classify the GPU path:

```text
1. Scheduling
2. Device Access
3. CUDA / Application Runtime
```

Record:

```text
Pod scheduled: YES / NO / UNKNOWN
GPU allocated: YES / NO / UNKNOWN
GPU visible: YES / NO / UNKNOWN
CUDA initialized: YES / NO / UNKNOWN
Model server initialized: YES / NO / UNKNOWN
```

Do not execute mutation or restart commands to test GPU behavior.

---

## 23. Classify the Failure Domain

Choose only evidence-supported domains:

```text
Deployment Identity
TrueFoundry Management / Control Plane
TrueFoundry Cluster Integration
Kubernetes Control / Reconciliation
Scheduling / Capacity
Node / Container Runtime
GPU / Device / CUDA Runtime
Application / Model Runtime
Networking
Storage
Identity / RBAC / IAM
Configuration / Secrets
External Dependency
Unknown
```

Record:

```text
Current Failure Domain:
Evidence:
Confidence:
```

---

## 24. Identify the Next Required Action

Do not jump directly from failure domain to team ownership.

First define:

```text
Next Required Action:
Expected Evidence / Result:
```

Example:

```text
Next Required Action:
Validate whether the affected Pod received the requested GPU device.

Expected Evidence:
GPU allocation/device visibility evidence for the Pod.
```

---

## 25. Determine the Current Actionable Owner

Now apply your organization's support model.

Record:

```text
Current Actionable Owner:
Reason:
Supporting Function(s):
```

Remember:

```text
Failure Domain ≠ Organizational Owner
```

and:

```text
Investigation Responsibility ≠ Component Ownership
```

Do not invent an owner if your organization's ownership map is unknown.

Use:

```text
Current Actionable Owner: UNKNOWN
Required: Consult organization support/escalation map
```

---

## 26. Build a Hypothesis Record

Create at least one hypothesis:

```text
Hypothesis:

Evidence For:

Evidence Against:

Validation Required:

Current Actionable Owner:

Status:
OPEN / REJECTED / SUPPORTED / PROVEN
```

If evidence disproves the hypothesis, mark it `REJECTED` rather than deleting it.

---

## 27. Create an Incident Handoff

Complete:

```text
Incident ID:
Timestamp:

Environment:
Compute Plane:
Workspace:
Namespace:
Workload:
Artifact / Version:

Impact:
Minimum Supported Blast Radius:

Management Impact:
Deployment Impact:
Runtime Impact:

Lowest Proven Healthy Layer:
First Failed Transition:

Evidence:
Evidence Confidence:

What Has Been Ruled Out:
What Remains Unknown:

Current Failure Domain:
Current Actionable Owner:

Requested Action:
Expected Evidence / Result:
```

The requested action must be specific.

Avoid:

```text
Please investigate.
```

---

## 28. Determine Whether Vendor Escalation Is Supported

Do not automatically escalate because the symptom is visible in TrueFoundry.

Ask:

```text
Is the suspected failure inside a TrueFoundry-controlled component?

Is Kubernetes/runtime evidence already sufficient to explain the symptom?

Is the cluster-integration boundary implicated?

Is the Control Plane implicated?

Is the minimum blast radius established?

Is the evidence sanitized?
```

If vendor escalation is supported, prepare:

```text
Time window:
Compute Plane:
Workspace/workload:
Affected component:
Version information:
Sanitized error:
Kubernetes evidence:
Cluster-integration evidence:
Blast radius:
Management impact:
Deployment impact:
Runtime impact:
Troubleshooting completed:
Requested vendor action:
```

Do not include credentials or Secret values.

---

## 29. Mitigation vs Remediation Exercise

Using the incident under investigation, record separately:

```text
Potential Mitigation:
```

and:

```text
Potential Permanent Remediation:
```

Do not execute either action in this lab.

Remember:

```text
Service Restored ≠ Root Cause Remediated
```

---

## 30. Recovery Validation Plan

Without changing the environment, write the checks that would be required after remediation.

Use:

```text
[ ] Authoritative desired state verified
[ ] Kubernetes reconciliation verified
[ ] Expected Pods healthy
[ ] Application/model healthy
[ ] Endpoint healthy
[ ] Dependencies healthy
[ ] Error rate normalized
[ ] Latency normalized
[ ] User transaction validated
```

A green deployment status alone is not sufficient.

---

## 31. Production Incident Snapshot

Complete the final lab snapshot:

```text
Timestamp:

Environment:
Compute Plane:
Workspace:
Namespace:
Workload:

Impact:
Minimum Supported Blast Radius:

Management Impact:
Deployment Impact:
Runtime Impact:

Lowest Proven Healthy Layer:
First Failed Transition:

Current Failure Domain:
Evidence Confidence:

Current Actionable Owner:
Supporting Functions:

Authoritative Configuration Source:
Configuration Owner:

Next Required Action:
Expected Evidence:

Potential Mitigation:
Potential Permanent Remediation:

Recovery Validation Required:
```

---

## 32. Acceptance Checklist

The lab passes when you can demonstrate:

```text
[ ] Kubernetes context verified
[ ] Target namespace verified
[ ] TrueFoundry/cluster-integration evidence inspected where available
[ ] Workload correlated to Kubernetes resources
[ ] Deployment identity recorded
[ ] Artifact identity inspected
[ ] Configuration references inspected without exposing Secret values
[ ] Minimum supported blast radius established
[ ] Management/deployment/runtime impact classified
[ ] Relevant Kubernetes events inspected
[ ] Relevant logs inspected safely
[ ] Evidence timestamped
[ ] Evidence confidence classified
[ ] Lowest Proven Healthy Layer identified or marked UNKNOWN
[ ] First Failed Transition identified or marked UNKNOWN
[ ] Failure domain classified
[ ] Next required action defined
[ ] Current Actionable Owner determined from organizational ownership model or marked UNKNOWN
[ ] Hypothesis record created
[ ] Incident handoff prepared
[ ] Vendor escalation boundary evaluated
[ ] Mitigation and remediation distinguished
[ ] Recovery validation plan created
[ ] No cluster state changed
[ ] No Secret values exposed
```

---

## 33. Lab Failure Conditions

The lab should be considered failed if you:

- modify cluster state
- restart or delete a workload
- expose credentials or Secret values
- investigate the wrong cluster without recognizing the mismatch
- claim a broader blast radius than the evidence supports
- claim an assumed cause as proven
- assign organizational ownership without an ownership model
- treat a Kubernetes status reason as a complete root cause without supporting evidence

---

## 34. Final Lab Principle

The lab is complete when you can move from a user-visible symptom to an evidence-supported next action without changing production state.

```text
Evidence
   ↓
Failure Boundary
   ↓
Next Required Action
   ↓
Current Actionable Owner
   ↓
Authoritative Remediation
   ↓
End-to-End Validation
```

This is the operational foundation for the later TrueFoundry troubleshooting and SRE runbook modules.
