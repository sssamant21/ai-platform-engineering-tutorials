# Part 2.1 Hands-On Lab — Connecting a Kubernetes Cluster to TrueFoundry

> **[SAFE-READ]**
>
> This lab is designed for production-safe observation. It does **not** install, modify, restart, scale, patch, or delete Kubernetes resources.
>
> Do not decode or display Kubernetes Secret values.

## Objective

Validate the Kubernetes-side prerequisites and observable evidence needed before or after connecting a Kubernetes cluster to TrueFoundry.

By the end of this lab, you should be able to:

- prove which Kubernetes cluster your current context targets;
- establish a basic cluster-health baseline;
- inspect node capacity and scheduling characteristics;
- identify existing platform components that may overlap with TrueFoundry integration;
- inspect storage, ingress, autoscaling, GPU, and GitOps indicators;
- perform safe Kubernetes RBAC checks;
- inspect recent events without changing cluster state;
- distinguish installation evidence from integration and production-acceptance evidence;
- record unknowns instead of turning assumptions into conclusions.

---

## Safety Rules

Allowed examples in this lab include:

```bash
kubectl config current-context
kubectl config view
kubectl cluster-info
kubectl get
kubectl describe
kubectl logs
kubectl auth can-i
```

Do **not** use this lab to run:

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

Do not run commands that decode or print Secret data.

Production rule:

```text
UNKNOWN TARGET = NO MUTATION
```

---

## Prerequisites

You need:

- `kubectl`;
- an existing kubeconfig/context;
- read access to the target cluster;
- authorization to run the read-only commands permitted by your organization;
- expected cluster identity information from an authoritative source.

Before beginning, record:

```text
Expected Environment:
Expected Cloud Provider:
Expected Account / Subscription / Project:
Expected Region:
Expected Kubernetes Cluster:
Expected TrueFoundry Control Plane:
Expected Compute Plane:
Expected Workspace:
Expected Namespace Mapping:
```

If these values are unknown, record them as `UNKNOWN`.

Do not invent them.

---

# Lab 1 — Verify the Active Kubernetes Context

Run:

```bash
kubectl config current-context
```

Record:

```text
Observed Context:
```

Inspect the active context without exposing credentials:

```bash
kubectl config view --minify
```

Focus on:

```text
cluster
context
namespace
user identity/reference
server/API endpoint
```

Do not treat a friendly context name alone as proof of cluster identity.

### Evidence

```text
Expected Cluster:
Observed Context:
Observed API Endpoint:
Match: YES / NO / UNKNOWN
Evidence Confidence: PROVEN / SUPPORTED / UNKNOWN / ASSUMED
```

### Gate

If the observed target cannot be matched to the expected target:

```text
STOP
```

Do not perform cluster mutations.

---

# Lab 2 — Verify Kubernetes API Reachability

Run:

```bash
kubectl cluster-info
```

Then:

```bash
kubectl get --raw=/readyz
```

If your permissions or Kubernetes version do not allow the raw readiness endpoint, record that result rather than changing permissions.

Record:

```text
API Reachable:
Readiness Result:
Authorization Error:
Other Error:
```

Remember:

```text
kubectl works
    ≠
cluster is fully healthy
```

---

# Lab 3 — Inspect Node Health

Run:

```bash
kubectl get nodes -o wide
```

Record:

```text
Total Nodes:
Ready Nodes:
NotReady Nodes:
Unknown Nodes:
Kubernetes Version(s):
```

Investigate any non-Ready node using:

```bash
kubectl describe node <NODE_NAME>
```

Focus on:

```text
Conditions
Capacity
Allocatable
Taints
Labels
Allocated resources
Recent events
```

Do not change node labels, taints, or scheduling state.

---

# Lab 4 — Inspect Scheduling and Capacity Evidence

Run:

```bash
kubectl get nodes -o custom-columns="NAME:.metadata.name,CPU:.status.capacity.cpu,MEMORY:.status.capacity.memory,PODS:.status.capacity.pods"
```

Inspect labels:

```bash
kubectl get nodes --show-labels
```

Inspect taints safely:

```bash
kubectl get nodes -o custom-columns="NAME:.metadata.name,TAINTS:.spec.taints"
```

Record:

```text
CPU Node Pools Observed:
GPU Node Pools Observed:
Important Labels:
Important Taints:
Potential Capacity Concern:
```

This lab does not perform full capacity planning.

Remember:

```text
Resources visible now
        ≠
safe production headroom
```

---

# Lab 5 — Check for GPU Resources

If GPU workloads are expected, inspect advertised allocatable resources:

```bash
kubectl get nodes -o custom-columns="NAME:.metadata.name,NVIDIA_GPU:.status.allocatable.nvidia\.com/gpu"
```

A blank value can mean that the node does not advertise `nvidia.com/gpu`.

Record:

```text
GPU Required: YES / NO / UNKNOWN
GPU Nodes Observed:
GPU Resource Advertised:
```

Do not troubleshoot CUDA, drivers, MIG, GPU Operator, or vLLM deeply here.

Those topics belong in later parts.

---

# Lab 6 — Inventory Namespaces

Run:

```bash
kubectl get namespaces
```

Look for namespaces associated with:

```text
TrueFoundry
GitOps
Ingress/Gateway
Monitoring
Autoscaling
GPU infrastructure
Certificate management
External Secrets
Service mesh
```

Do not assume a namespace's purpose solely from its name.

Record:

```text
Relevant Namespaces:
Unknown Namespaces Requiring Ownership Confirmation:
```

---

# Lab 7 — Discover Existing Platform Components

Start with:

```bash
kubectl get deployments -A
```

Then:

```bash
kubectl get daemonsets -A
```

and:

```bash
kubectl get statefulsets -A
```

Look for evidence of components such as:

```text
tfy-agent
Argo CD
Argo Workflows
Ingress controllers
Gateway implementations
cert-manager
KEDA
Prometheus
NVIDIA GPU Operator
NVIDIA device plugin
CSI drivers
External Secrets
Service mesh
DNS controllers
```

Names vary between installations.

Do not conclude that a component is absent merely because an expected name was not found.

Build an inventory:

| Component | Observed | Namespace | Management Source | Confidence |
|---|---|---|---|---|
| TrueFoundry integration | | | | |
| GitOps | | | | |
| Ingress/Gateway | | | | |
| Certificate management | | | | |
| Autoscaling | | | | |
| Monitoring | | | | |
| GPU infrastructure | | | | |
| Storage/CSI | | | | |
| External Secrets | | | | |
| Service mesh | | | | |

Use `UNKNOWN` when ownership or management source cannot be proven.

---

# Lab 8 — Inspect TrueFoundry Integration Components

If the TrueFoundry namespace or component names are known, inspect them with read-only commands.

Example:

```bash
kubectl get pods -n <TRUEFOUNDRY_NAMESPACE> -o wide
```

```bash
kubectl get deployments -n <TRUEFOUNDRY_NAMESPACE>
```

For a specific Pod:

```bash
kubectl describe pod <POD_NAME> -n <TRUEFOUNDRY_NAMESPACE>
```

Where organizational policy permits log access:

```bash
kubectl logs <POD_NAME> -n <TRUEFOUNDRY_NAMESPACE>
```

If the Pod has multiple containers:

```bash
kubectl get pod <POD_NAME> -n <TRUEFOUNDRY_NAMESPACE> -o jsonpath='{.spec.containers[*].name}'
```

Then:

```bash
kubectl logs <POD_NAME> -n <TRUEFOUNDRY_NAMESPACE> -c <CONTAINER_NAME>
```

### Safety

Do not paste unsanitized logs into tickets or public channels.

Review logs for tokens, credentials, URLs containing secrets, customer data, or other sensitive information before sharing.

Record:

```text
Component:
Namespace:
Pod State:
Ready:
Restarts:
Relevant Event:
Relevant Sanitized Log Evidence:
```

Remember:

```text
Pod Running ≠ Integration Healthy
```

---

# Lab 9 — Inspect StorageClasses

Run:

```bash
kubectl get storageclass
```

For a relevant class:

```bash
kubectl describe storageclass <STORAGECLASS_NAME>
```

Record:

```text
StorageClasses Observed:
Default StorageClass:
Relevant Provisioner:
Potential Concern:
```

Do not create a PVC as part of this `[SAFE-READ]` lab.

Remember:

```text
StorageClass Exists ≠ Persistent Storage Works
```

---

# Lab 10 — Inspect Ingress and Gateway Resources

Check standard Ingress resources:

```bash
kubectl get ingress -A
```

If Gateway API resources are installed and your permissions allow access:

```bash
kubectl get gateway -A
```

```bash
kubectl get httproute -A
```

If a resource type is unavailable:

```text
Record the result.
Do not install the CRD during this lab.
```

Record:

```text
Ingress Resources:
Gateway Resources:
Observed Controller/Implementation:
Ownership:
Confidence:
```

Detailed networking analysis belongs in Part 2.6 and Part 9.

---

# Lab 11 — Inspect Kubernetes RBAC Safely

Check your current authorization without changing permissions:

```bash
kubectl auth can-i get pods --all-namespaces
```

```bash
kubectl auth can-i list deployments --all-namespaces
```

```bash
kubectl auth can-i get events --all-namespaces
```

If a known ServiceAccount must be assessed and your own identity is authorized to impersonate it, a command such as the following can be used:

```bash
kubectl auth can-i get pods -n <NAMESPACE> --as=system:serviceaccount:<NAMESPACE>:<SERVICEACCOUNT>
```

If impersonation is forbidden, record the denial.

Do not request or grant broader permissions merely to complete the lab.

Record:

```text
Identity Tested:
Operation:
Allowed:
Denied:
Unknown:
```

Remember:

```text
TrueFoundry Authorization
        ≠
Kubernetes RBAC
        ≠
Cloud IAM
```

---

# Lab 12 — Inspect Recent Kubernetes Events

Run:

```bash
kubectl get events -A --sort-by=.lastTimestamp
```

Focus on relevant recent evidence such as:

```text
FailedScheduling
FailedMount
FailedAttachVolume
Failed
BackOff
Unhealthy
Pulling
Pulled
Created
Started
Killing
```

Record relevant evidence:

```text
Timestamp:
Namespace:
Object:
Reason:
Message:
```

Events are time-sensitive evidence.

Do not assume an old event describes the current state.

---

# Lab 13 — Inspect Workload Conditions

List Pods:

```bash
kubectl get pods -A -o wide
```

Look for:

```text
Pending
CrashLoopBackOff
ImagePullBackOff
Error
ContainerCreating
Terminating
```

For a relevant Pod:

```bash
kubectl describe pod <POD_NAME> -n <NAMESPACE>
```

Classify what the evidence proves.

Example:

```text
Observed:
Pod Pending

Event:
FailedScheduling

PROVEN:
Scheduler could not currently place the Pod.

UNKNOWN:
Underlying reason until scheduler event details are inspected.
```

Kubernetes status is evidence, not automatically a complete RCA.

---

# Lab 14 — Determine the Minimum Supported Blast Radius

For any observed problem, classify the smallest scope supported by evidence:

```text
One container
One Pod
One deployment
One platform component
One namespace
One node
One node pool
One Compute Plane
Multiple Compute Planes
Control Plane
UNKNOWN
```

Record:

```text
Observed Symptom:
Minimum Supported Blast Radius:
Evidence:
Confidence:
```

Avoid broad statements unsupported by evidence.

---

# Lab 15 — Find the Lowest Proven Healthy Layer

Use:

```text
Control Plane
      ↓
Cluster Integration
      ↓
Kubernetes API
      ↓
Scheduler
      ↓
Node
      ↓
Container Runtime
      ↓
Application
      ↓
Network / DNS / TLS
      ↓
Dependency
```

Example:

```text
Pod Created:                PROVEN HEALTHY
Pod Scheduled:              PROVEN HEALTHY
Container Started:          PROVEN HEALTHY
DNS Resolution:             FAILED
TLS:                        UNKNOWN
Control Plane Connectivity: UNKNOWN
```

Record:

```text
Lowest Proven Healthy Layer:
Evidence:
```

---

# Lab 16 — Identify the First Failed Transition

Using the evidence collected, identify the first transition that can be shown to fail.

Example:

```text
Container
    ↓
DNS Resolution
    X
Destination
```

Record:

```text
First Failed Transition:
Failure Domain:
Evidence:
Confidence:
```

Possible failure domains include:

```text
Deployment Identity
TrueFoundry Control Plane
Cluster Integration
Kubernetes API / Reconciliation
Kubernetes RBAC
Cloud IAM
Scheduling / Capacity
Node / Container Runtime
DNS
Network / Firewall / Proxy
TLS / Certificate Trust
Container Registry
Storage
Configuration
GPU Infrastructure
External Dependency
Unknown
```

---

# Lab 17 — Classify Evidence Confidence

For each important statement, use:

```text
PROVEN
SUPPORTED
UNKNOWN
ASSUMED
```

Example:

```text
PROVEN:
Pod is Running.

PROVEN:
Connection attempt returned a TLS trust error.

SUPPORTED:
The first failed transition is TLS validation.

UNKNOWN:
Why the trust configuration changed.

ASSUMED:
TrueFoundry caused the trust configuration change.
```

Do not use `ASSUMED` statements as established RCA facts.

---

# Lab 18 — Determine the Current Actionable Owner

Use:

```text
Evidence
   ↓
Failure Domain
   ↓
Next Required Action
   ↓
Current Actionable Owner
```

Record:

```text
Failure Domain:
Next Required Action:
Authoritative Configuration Source:
Current Actionable Owner:
```

Do not assign ownership solely from where an error appeared.

---

# Lab 19 — Production Acceptance Evidence

Evaluate three separate gates.

## Gate 1 — Installation

```text
Expected Resources Exist:
Pods Scheduled:
Containers Started:
Result: PASS / FAIL / UNKNOWN
```

## Gate 2 — Integration

```text
Cluster Integration Healthy:
Control Plane Communication Healthy:
Required Permissions Working:
Reconciliation Working:
Result: PASS / FAIL / UNKNOWN
```

## Gate 3 — Production Acceptance

```text
Representative Workload Healthy:
Networking Validated:
DNS/TLS Validated:
Required Storage Validated:
GPU Validated Where Required:
Observability Validated:
Existing Workloads Healthy:
Result: PASS / FAIL / UNKNOWN
```

Do not mark an item PASS without evidence.

Remember:

```text
Installation Success
        ≠
Integration Success
        ≠
Production Acceptance
```

---

# Lab 20 — Build the Evidence Handoff

Create a sanitized handoff:

```text
Timestamp:
Environment:
Compute Plane:
Cluster:
Affected Component:
Observed Symptom:

Minimum Supported Blast Radius:

Relevant Kubernetes State:
Relevant Events:
Relevant Sanitized Logs:

Lowest Proven Healthy Layer:
First Failed Transition:
Failure Domain:

Evidence Confidence:

Actions Already Attempted:

Next Required Action:
Current Actionable Owner:

Requested Action:
```

Do not include credentials or Secret values.

---

# Lab Acceptance Criteria

The lab is complete when you can demonstrate:

- [ ] active Kubernetes target was verified or explicitly marked `UNKNOWN`;
- [ ] Kubernetes API reachability was checked;
- [ ] node health was reviewed;
- [ ] scheduling/capacity indicators were inspected;
- [ ] GPU resources were checked where relevant;
- [ ] namespaces and existing platform components were inventoried;
- [ ] TrueFoundry integration components were inspected where present;
- [ ] StorageClasses were inspected;
- [ ] ingress/gateway evidence was inspected where available;
- [ ] relevant RBAC capability was checked without modifying permissions;
- [ ] recent Kubernetes events were reviewed;
- [ ] no Secret values were decoded or exposed;
- [ ] Minimum Supported Blast Radius was established for any observed problem;
- [ ] Lowest Proven Healthy Layer was identified where troubleshooting was required;
- [ ] First Failed Transition was identified where evidence permitted;
- [ ] evidence was classified as `PROVEN`, `SUPPORTED`, `UNKNOWN`, or `ASSUMED`;
- [ ] Current Actionable Owner was based on the next required action;
- [ ] installation, integration, and production acceptance were evaluated separately.

---

# Final Rule

```text
OBSERVE
   ↓
PROVE TARGET
   ↓
COLLECT EVIDENCE
   ↓
IDENTIFY FAILURE BOUNDARY
   ↓
IDENTIFY NEXT ACTION
   ↓
IDENTIFY OWNER
```

For this lab:

```text
NO MUTATION
NO SECRET DECODING
NO ASSUMPTION PRESENTED AS FACT
```
