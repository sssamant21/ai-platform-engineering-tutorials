# Lab 1.5 — Workspaces, Environments & Deployment Concepts

**Classification:** `[SAFE-READ]`  
**Track:** TrueFoundry — Part 1: Foundations & Architecture  
**Tutorial:** Part 1.5 — Workspaces, Environments & Deployment Concepts

## 1. Purpose

This lab practices production-safe correlation between a TrueFoundry workload and its Kubernetes runtime.

The objective is to establish:

- the current Kubernetes context
- the target cluster / Compute Plane
- the Workspace and workload identity where available
- the Kubernetes namespace and workload resources
- artifact identity
- desired versus observed state
- environment-specific configuration differences
- recent deployment evidence
- blast radius
- the lowest proven healthy layer
- the first failed transition
- the authoritative configuration source and likely owner

This lab is observational only. Do not change workload or cluster state.

## 2. Safety Boundary

Allowed commands:

```bash
kubectl config current-context
kubectl get
kubectl describe
kubectl logs
```

Do not use:

```bash
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

Do not retrieve, decode, print, copy, or paste Secret values.

Production rule:

```text
UNKNOWN TARGET
      =
NO MUTATION
```

## 3. Variables

Record the values relevant to your environment before continuing:

```text
Expected operational environment:
Expected Compute Plane / cluster:
Expected Workspace:
Expected namespace:
Expected workload:
Expected artifact/version:
```

For shell commands below, substitute:

```text
<NAMESPACE>
<WORKLOAD>
<POD>
```

with validated values from your environment.

## 4. Validate Kubernetes Context

Run:

```bash
kubectl config current-context
```

Record:

```text
Expected context:
Observed context:
Match: YES / NO
```

If the context is unexpected, stop the lab for that environment.

Do not attempt corrective mutations.

## 5. Validate Cluster and Node Visibility

Run:

```bash
kubectl get nodes -o wide
```

Record:

```text
Kubernetes API reachable:
Node count:
Ready nodes:
NotReady nodes:
Observed node pools / relevant labels:
```

This establishes basic Kubernetes visibility. It does not prove application health.

## 6. Discover Namespaces

Run:

```bash
kubectl get namespaces
```

Identify the namespace associated with the target workload using platform/deployment evidence.

Do not assume:

```text
Workspace = Namespace
```

Record:

```text
Workspace:
Observed namespace:
Evidence used to establish mapping:
```

## 7. Discover TrueFoundry Integration Components

Use read-only discovery:

```bash
kubectl get pods -A | grep -i truefoundry
kubectl get pods -A | grep -i tfy
kubectl get deployments -A | grep -i tfy
```

If the environment uses different component names, use the names documented by your platform team.

Record:

```text
TrueFoundry-related namespace(s):
tfy-agent/component found:
Observed status:
```

Absence from these searches alone is not proof of a platform failure.

## 8. Identify the Workload

Search the expected namespace:

```bash
kubectl get deployments -n <NAMESPACE>
kubectl get statefulsets -n <NAMESPACE>
kubectl get jobs -n <NAMESPACE>
kubectl get pods -n <NAMESPACE> -o wide
```

Record:

```text
TrueFoundry workload:
Kubernetes workload type:
Kubernetes workload name:
Namespace:
Pod(s):
Node(s):
```

The goal is to correlate platform identity with Kubernetes identity.

## 9. Validate Intended vs Observed Target

Create the following comparison:

```text
                         INTENDED              OBSERVED

Environment:
Compute Plane / Cluster:
Workspace:
Namespace:
Workload:
```

Any mismatch is operational evidence.

A healthy Pod in the wrong target is still an incorrect deployment.

## 10. Inspect Workload Desired State

For a Deployment:

```bash
kubectl get deployment <WORKLOAD> -n <NAMESPACE> -o wide
kubectl describe deployment <WORKLOAD> -n <NAMESPACE>
```

If the workload uses another Kubernetes controller, inspect that controller instead.

Record:

```text
Desired replicas:
Current replicas:
Ready replicas:
Available replicas:
Container image:
ServiceAccount:
Resource requests:
Resource limits:
Node placement constraints:
```

Do not modify anything.

## 11. Inspect Pod Observed State

Run:

```bash
kubectl get pods -n <NAMESPACE> -o wide
kubectl describe pod <POD> -n <NAMESPACE>
```

Record:

```text
Pod:
Phase:
Ready:
Restart count:
Node:
Container state:
Previous container state:
Reason:
```

Compare desired workload state with observed Pod state.

## 12. Verify Artifact Identity

Inspect the workload and Pod:

```bash
kubectl get deployment <WORKLOAD> -n <NAMESPACE> -o wide
kubectl describe pod <POD> -n <NAMESPACE>
```

Record, where available:

```text
Image repository:
Image tag:
Image ID / digest:
Expected artifact:
Observed artifact:
Match:
```

Remember:

```text
Same tag != Proven same artifact
```

A digest or immutable image identifier is stronger evidence than a mutable tag.

## 13. Review Configuration Without Exposing Secrets

Inspect workload metadata and configuration references:

```bash
kubectl describe deployment <WORKLOAD> -n <NAMESPACE>
kubectl get configmaps -n <NAMESPACE>
kubectl get secrets -n <NAMESPACE>
kubectl get serviceaccounts -n <NAMESPACE>
```

Only record metadata and reference names.

Do not run commands intended to decode Secret data.

Record:

```text
ConfigMap references:
Secret reference names:
ServiceAccount:
Expected references:
Observed references:
```

Do not record credential values.

## 14. Determine Configuration Provenance

Using your organization's deployment documentation, GitOps configuration, TrueFoundry configuration, or deployment metadata, determine the expected authoritative source.

Record:

```text
Configuration item:
Observed Kubernetes value/reference:
Expected value/reference:
Authoritative source:
Configuration owner:
```

Examples of authoritative sources may include:

```text
TrueFoundry configuration
Git repository
GitOps repository
CI/CD configuration
Infrastructure-as-Code
Secret-management integration
```

Do not assume that a live Kubernetes object is the authoritative source.

## 15. Check for Drift

Compare:

```text
AUTHORITATIVE
      ↓
DESIRED
      ↓
OBSERVED
```

Record any mismatch:

```text
Artifact:
Replica count:
CPU:
Memory:
GPU:
ServiceAccount:
ConfigMap reference:
Secret reference:
Node placement:
```

Classify each result as:

```text
MATCH
EXPECTED DIFFERENCE
UNEXPECTED DIFFERENCE
UNKNOWN
```

Do not correct drift during this lab.

## 16. Review Recent Events

Run:

```bash
kubectl get events -n <NAMESPACE> --sort-by=.lastTimestamp
```

Look for evidence such as:

```text
FailedScheduling
FailedMount
FailedAttachVolume
BackOff
Unhealthy
Killing
Pulled
Pulling
Failed
```

Record:

```text
Timestamp:
Object:
Reason:
Message summary:
```

An event is evidence, not automatically root cause.

## 17. Review Application Logs

For the target Pod:

```bash
kubectl logs <POD> -n <NAMESPACE> --tail=100
```

For a restarted container, where appropriate:

```bash
kubectl logs <POD> -n <NAMESPACE> --previous --tail=100
```

Record only operationally relevant evidence.

Do not paste credentials, tokens, patient data, personal data, or other sensitive values into lab notes.

## 18. Determine Recent Change Correlation

Build a simple timeline:

```text
T0:
Service/workload state:

T1:
Deployment/configuration change:

T2:
New workload/Pod state:

T3:
First observed warning/error:

T4:
User/application impact:
```

Use:

```text
Change followed by failure = investigation lead
```

Do not use:

```text
Change followed by failure = automatically proven root cause
```

## 19. Determine Blast Radius

Classify the observed scope:

```text
[ ] One container
[ ] One Pod
[ ] One workload
[ ] One namespace
[ ] One Workspace
[ ] One Compute Plane / cluster
[ ] Multiple Compute Planes
[ ] Platform-wide
[ ] External dependency
[ ] Not yet established
```

Record the evidence supporting the classification.

Do not declare platform-wide impact from a single failed workload.

## 20. Compare Environments

If you have authorized read access to both staging and production, compare them without changing either environment.

Use four categories:

```text
1. Artifact
2. Configuration
3. Infrastructure
4. Dependencies
```

Worksheet:

```text
                               STAGING          PRODUCTION

Image tag:
Image digest:

CPU request:
CPU limit:
Memory request:
Memory limit:
GPU request:
Replicas:
ServiceAccount:
Secret reference:
Node placement:

Cluster:
Node pool:
GPU type:
Available capacity:
Storage class:

Database endpoint/reference:
Redis endpoint/reference:
Object storage reference:
External dependency:
```

Do not expose Secret values.

A difference is not automatically a defect. Determine whether it is intentional.

## 21. Capacity Check

Review relevant node resources:

```bash
kubectl get nodes
kubectl describe node <NODE>
```

For GPU-capable clusters, inspect allocatable GPU resources where applicable:

```bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,GPU:.status.allocatable.nvidia\.com/gpu
```

Record:

```text
Requested resource:
Relevant node capacity:
Observed allocatable capacity:
Scheduling evidence:
```

Remember:

```text
Same workload configuration
        !=
Same schedulability
```

## 22. Healthy vs Correct Deployment

Answer both questions separately.

### Runtime health

```text
Pods running:
Readiness passing:
Expected replicas available:
Service/endpoints present:
```

### Deployment correctness

```text
Correct environment:
Correct Compute Plane:
Correct Workspace:
Correct namespace:
Correct artifact:
Correct configuration:
Correct identity:
Correct dependencies:
```

Then classify:

```text
[ ] Healthy and correct
[ ] Healthy but incorrect
[ ] Correct target but unhealthy
[ ] Insufficient evidence
```

## 23. Lowest Proven Healthy Layer

Use the evidence collected to identify the lowest layer that is proven healthy.

Example layers:

```text
TrueFoundry management layer
        ↓
Kubernetes API
        ↓
Scheduler / controllers
        ↓
Node
        ↓
Container runtime
        ↓
Pod
        ↓
Application
        ↓
Network endpoint
        ↓
Dependency
        ↓
User transaction
```

Record:

```text
Lowest proven healthy layer:
Evidence:
```

## 24. First Failed Transition

Identify the first transition for which evidence shows failure.

Examples:

```text
Kubernetes API → workload reconciliation

Scheduler → node placement

Node → container startup

Container → application readiness

Application → dependency

Service → Pod

Endpoint → user transaction
```

Record:

```text
First failed transition:
Evidence:
```

Do not skip directly from symptom to assumed root cause.

## 25. Failure-Domain Classification

Classify the current evidence:

```text
[ ] Deployment identity
[ ] TrueFoundry management/control
[ ] Kubernetes API/control
[ ] Scheduling/capacity
[ ] Node/container runtime
[ ] GPU/device runtime
[ ] Application/model runtime
[ ] Networking
[ ] Storage
[ ] Identity/RBAC
[ ] Configuration
[ ] Dependency
[ ] External system
[ ] Unknown
```

Record:

```text
Failure domain:
Evidence:
Likely owner:
Additional evidence required:
```

## 26. Production Incident Snapshot

Complete:

```text
Timestamp:

Operational Environment:

Compute Plane:
Cluster:
Kubernetes Context:

Workspace:
Namespace:
Workload:

Artifact:
Image Tag:
Image Digest:
Deployment Version:

Authoritative Configuration Source:
Configuration Owner:

Desired Replicas:
Ready Replicas:

Pod Status:
Restart Count:
Node:

Recent Deployment:
Recent Configuration Change:

Events:
Application Evidence:
Infrastructure Evidence:
Dependency Evidence:

Lowest Proven Healthy Layer:
First Failed Transition:

Blast Radius:
Failure Domain:
Current Owner:

Recommended authoritative remediation path:
Validation still required:
```

Do not include Secret values.

## 27. Incident Communication Exercise

Write a short evidence-based status update.

Use this structure:

```text
The affected workload is <WORKLOAD> in <ENVIRONMENT>.

The intended target is <EXPECTED TARGET>, and the observed target is
<OBSERVED TARGET>.

The deployment is running artifact <ARTIFACT IDENTITY>.

The current blast radius is <SCOPE>.

The lowest proven healthy layer is <LAYER>, and the first failed
transition is <TRANSITION>.

Current evidence points to the <FAILURE DOMAIN> domain. The
authoritative configuration source is <SOURCE>, owned by <OWNER>.

End-to-end recovery has / has not yet been validated.
```

Use evidence, not unsupported conclusions.

## 28. Acceptance Checklist

The lab passes when you can demonstrate:

```text
[ ] Kubernetes context verified
[ ] Target Compute Plane / cluster identified
[ ] Workspace identified where available
[ ] Namespace established from evidence
[ ] Workload correlated with Kubernetes resources
[ ] Intended and observed targets compared
[ ] Artifact identity inspected
[ ] Desired and observed state compared
[ ] Configuration references inspected safely
[ ] No Secret values exposed
[ ] Authoritative configuration source identified where possible
[ ] Configuration owner identified where possible
[ ] Drift evaluated
[ ] Recent events reviewed
[ ] Relevant logs reviewed safely
[ ] Recent changes correlated without assuming causation
[ ] Blast radius classified
[ ] Environment differences evaluated where applicable
[ ] Capacity considered
[ ] Runtime health and deployment correctness evaluated separately
[ ] Lowest proven healthy layer identified
[ ] First failed transition identified
[ ] Failure domain classified
[ ] Incident snapshot completed
[ ] No cluster or workload state changed
```

## 29. Lab Completion Rule

This lab is complete only when the investigator can answer:

```text
WHAT is deployed?

WHERE is it intended to run?

WHERE is it actually running?

WHICH artifact is running?

WHICH configuration controls it?

WHO owns that configuration?

WHAT is the current blast radius?

WHAT is the lowest proven healthy layer?

WHERE is the first failed transition?

WHAT still needs validation?
```

The production principle is:

```text
IDENTIFY
   ↓
VERIFY TARGET
   ↓
VERIFY DEPLOYMENT
   ↓
COMPARE
   ↓
PROVE
   ↓
CLASSIFY
   ↓
REMEDIATE THROUGH AUTHORITATIVE SOURCE
   ↓
VALIDATE END TO END
```

No remediation is performed as part of this `[SAFE-READ]` lab.
