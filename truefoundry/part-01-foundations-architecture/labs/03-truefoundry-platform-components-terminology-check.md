# Lab 1.3 — TrueFoundry Platform Components & Workload Correlation

**Classification:** `[SAFE-READ]`  
**Associated Tutorial:** Part 1.3 — TrueFoundry Platform Components & Terminology  
**Audience:** SRE / Platform Engineering  
**Mode:** Observation only

## 1. Purpose

This lab turns the terminology from Part 1.3 into an operational investigation.

The goal is to move from:

```text
TrueFoundry workload
       ↓
Workspace / Compute Plane
       ↓
Kubernetes representation
       ↓
Pod / Container
       ↓
Runtime evidence
       ↓
Failure domain
```

without changing cluster state.

## 2. Learning Objectives

By completing this lab, you should be able to:

- verify the active Kubernetes context
- establish Compute Plane health
- discover TrueFoundry platform components
- identify `tfy-agent` where present
- identify different Kubernetes workload types
- correlate an approved workload with Kubernetes resources
- compare desired state with observed state
- inspect events and runtime evidence
- identify network and GPU dependencies
- classify the likely failure domain using evidence
- complete all checks without modifying cluster state

## 3. Safety Gate

This lab is `[SAFE-READ]`.

Allowed:

```text
kubectl config current-context
kubectl get
kubectl describe
kubectl logs
```

Not allowed:

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

Secret values must **not** be retrieved or decoded.

## 4. Confirm Kubernetes Context

```bash
kubectl config current-context
```

Record:

```text
Cluster context:
Environment:
Expected cluster: YES / NO
```

Stop if you are connected to an unintended cluster.

## 5. Establish Compute-Plane Health

```bash
kubectl get nodes -o wide
```

Record node readiness.

Then:

```bash
kubectl get pods -A
```

This establishes observed Kubernetes state before investigating an individual workload.

## 6. Discover TrueFoundry Components

```bash
kubectl get pods -A | grep -Ei 'truefoundry|tfy'
```

Then:

```bash
kubectl get deployments -A | grep -Ei 'truefoundry|tfy'
```

Record:

```text
Namespace:
Component:
Resource type:
Ready:
Status:
Restarts:
```

## 7. Identify `tfy-agent`

```bash
kubectl get pods -A | grep -i 'tfy-agent'
```

If present:

```bash
kubectl describe pod <tfy-agent-pod> -n <namespace>
```

Where policy permits:

```bash
kubectl logs <tfy-agent-pod> -n <namespace> --tail=100
```

This establishes evidence for the cluster-integration layer.

If no matching component is visible, record that observation rather than modifying the environment.

## 8. Discover Workload Types

List Deployments:

```bash
kubectl get deployments -A
```

List StatefulSets:

```bash
kubectl get statefulsets -A
```

List Jobs:

```bash
kubectl get jobs -A
```

List CronJobs:

```bash
kubectl get cronjobs -A
```

Different platform workload types may result in different Kubernetes workload patterns.

## 9. Select One Approved Workload

Choose an approved workload for observation.

Record:

```text
TrueFoundry workload:
Workspace:
Workload type:
Compute environment:
Kubernetes namespace:
Kubernetes resource:
```

If information cannot be determined from Kubernetes evidence, record:

```text
Not observable from current Kubernetes evidence
```

Do not guess.

## 10. Correlate the Workload to Pods

For the selected namespace:

```bash
kubectl get pods -n <namespace> -o wide
```

Where useful:

```bash
kubectl get pods -n <namespace> --show-labels
```

For a Deployment:

```bash
kubectl describe deployment <deployment> -n <namespace>
```

For a Job:

```bash
kubectl describe job <job> -n <namespace>
```

Record:

```text
Platform workload
      ↓
Kubernetes namespace
      ↓
Deployment / Job
      ↓
Pod
      ↓
Container
      ↓
Node
```

## 11. Identify Workload Lifecycle

Determine whether the workload behaves like a continuously running Service or task-oriented Job.

Service-like workload:

```text
Running
Ready
Serving
```

Job:

```text
Pending
Running
Completed / Failed
```

Do not interpret a successfully completed Job as an unhealthy stopped Service.

## 12. Compare Desired and Observed State

For a Deployment:

```bash
kubectl get deployment <deployment> -n <namespace>
```

Record:

```text
DESIRED:
Replicas:

OBSERVED:
Ready:
Available:
Up-to-date:
```

Classify:

```text
Desired = Observed
        → candidate healthy state

Desired ≠ Observed
        → investigate
```

Matching replica counts alone do not prove application-level health; they establish only one layer of evidence.

## 13. Inspect Pod State

```bash
kubectl get pods -n <namespace> -o wide
```

For an unhealthy or otherwise relevant pod:

```bash
kubectl describe pod <pod> -n <namespace>
```

Look for:

```text
Pending
CrashLoopBackOff
ImagePullBackOff
OOMKilled
Evicted
FailedScheduling
Probe failures
Volume failures
```

## 14. Inspect Events

Namespace-specific:

```bash
kubectl get events -n <namespace> --sort-by='.lastTimestamp'
```

Or cluster-wide where permitted:

```bash
kubectl get events -A --sort-by='.lastTimestamp'
```

Use events to identify the layer producing failure evidence.

## 15. Inspect Runtime Evidence

Where policy permits:

```bash
kubectl logs <pod> -n <namespace> --tail=100
```

If the pod contains multiple containers:

```bash
kubectl get pod <pod> -n <namespace> -o jsonpath="{.spec.containers[*].name}"
```

Then:

```bash
kubectl logs <pod> -n <namespace> -c <container> --tail=100
```

Do not expose credentials or sensitive application data in lab evidence.

## 16. Identify Network Objects

Discover Kubernetes Services:

```bash
kubectl get svc -n <namespace>
```

Remember:

```text
TrueFoundry Service
        ≠
Kubernetes Service
```

Where relevant:

```bash
kubectl get ingress -n <namespace>
```

The objective is to identify the runtime network path without changing it.

## 17. Safe Secret Validation

Identify Secret names only:

```bash
kubectl get secrets -n <namespace>
```

Allowed conclusion:

```text
Expected Secret object exists.
```

Do not retrieve Secret YAML and do not decode Secret values.

The lab validates object existence and references, not secret contents.

## 18. GPU Workload Correlation

Where applicable:

```bash
kubectl get nodes -o custom-columns="NODE:.metadata.name,GPU:.status.allocatable.nvidia\.com/gpu"
```

Then:

```bash
kubectl get pods -A | grep -Ei 'nvidia|gpu'
```

For a selected inference pod:

```bash
kubectl describe pod <pod> -n <namespace>
```

Follow:

```text
TrueFoundry workload
        ↓
GPU requirement
        ↓
Kubernetes Pod
        ↓
Scheduler
        ↓
GPU node
        ↓
NVIDIA/CUDA
        ↓
Model server
```

If the cluster is CPU-only, mark this section as not applicable.

## 19. Failure-Domain Classification

Using collected evidence, classify the investigation into one of these layers:

```text
Control Plane
Cluster Integration
Kubernetes
Application
Model Server
GPU Runtime
Network
Storage
Infrastructure
```

Do not assign a layer without evidence.

## 20. Incident Communication Exercise

Convert:

```text
TrueFoundry service is down.
```

into an evidence-based statement.

Example:

```text
The workload exists in the target namespace, but its
pod is in CrashLoopBackOff. Kubernetes successfully
scheduled the pod, and container logs show an
application startup configuration failure.
```

Convert:

```text
GPU deployment is broken.
```

into:

```text
The inference pod remains Pending. Kubernetes events
report insufficient allocatable nvidia.com/gpu on
eligible nodes.
```

## 21. Evidence Template

```text
Kubernetes context:

Compute-plane health:

Workspace:
TrueFoundry workload:
Workload type:

Kubernetes namespace:
Kubernetes workload:
Pod:
Container:
Node:

Desired state:
Observed state:

Recent events:

Runtime evidence:

Network objects:

GPU requirement:
GPU capacity:

Suspected failure domain:

Evidence supporting classification:
```

Do not include credentials, tokens, or Secret values.

## 22. Acceptance Checklist

```text
[ ] Correct Kubernetes context verified
[ ] Compute-plane node health inspected
[ ] TrueFoundry components discovered
[ ] tfy-agent identified where present
[ ] Workload type identified
[ ] Kubernetes representation identified
[ ] Workload correlated to Pod
[ ] Pod correlated to Node
[ ] Desired state compared with observed state
[ ] Events inspected
[ ] Runtime evidence inspected where permitted
[ ] Network objects identified
[ ] Secret values not exposed
[ ] GPU mapping checked where applicable
[ ] Failure domain classified using evidence
[ ] No cluster state modified
```

## 23. Completion Criteria

The lab passes when the engineer can trace:

```text
Platform Object
      ↓
Workspace / Compute Plane
      ↓
Kubernetes Resource
      ↓
Pod
      ↓
Container
      ↓
Node
      ↓
Runtime Evidence
      ↓
Failure Domain
```

without modifying cluster state or exposing sensitive information.

**Lab result:** PASS only when all applicable acceptance checks are satisfied.
