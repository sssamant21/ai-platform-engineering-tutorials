# Part 3.2 — Requests, Limits & Kubernetes QoS

## Purpose

This tutorial explains how Kubernetes CPU and memory requests, limits, and Quality of Service (QoS) classes affect scheduling, resource accounting, runtime enforcement, node-pressure behavior, and production incident investigation.

The production resource lifecycle is:

```text
DESIRED RESOURCE CONFIGURATION
        ↓
ADMISSION / DEFAULTING
        ↓
RENDERED POD RESOURCES
        ↓
QoS CLASSIFICATION
        ↓
SCHEDULER ACCOUNTING
        ↓
NODE PLACEMENT
        ↓
RUNTIME RESOURCE ENFORCEMENT
        ↓
NODE-PRESSURE / EVICTION BEHAVIOR
```

The primary investigation question is:

```text
WHERE IS THE FIRST FAILED TRANSITION?
```

## Learning Objectives

By the end of this tutorial, you should be able to:

- explain CPU and memory requests and limits;
- distinguish requests from runtime usage;
- distinguish scheduling from runtime enforcement;
- identify `Guaranteed`, `Burstable`, and `BestEffort` QoS;
- inspect the QoS class assigned to a Pod;
- distinguish QoS from workload priority;
- distinguish CPU scheduling pressure from CPU throttling;
- classify memory failures without assuming a memory leak;
- distinguish container OOM from node-pressure eviction;
- understand `LimitRange` and `ResourceQuota`;
- recognize fragmentation and node-eligibility constraints;
- preserve incident evidence before remediation.

## 1. Establish Workload Identity First

Capture:

```text
Environment
Cluster
Namespace
Workload
Pod
Pod UID
Node
Container
Timestamp
```

A recreated Pod may not be the Pod that experienced the incident.

```text
UNKNOWN WORKLOAD IDENTITY = NO RESOURCE CHANGE
```

## 2. Four Resource States

```text
DESIRED
  ↓
authoritative deployment configuration

RENDERED
  ↓
resources actually present on the admitted Pod

ACCOUNTED
  ↓
resources Kubernetes uses for scheduling/accounting

CONSUMED
  ↓
actual runtime resource consumption
```

Therefore:

```text
DESIRED ≠ RENDERED ≠ ACCOUNTED ≠ CONSUMED
```

## 3. Resource Requests

Requests are scheduling/resource-accounting inputs.

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
```

A request is not current consumption.

```text
REQUEST ≠ USAGE
LOW UTILIZATION ≠ LOW RESOURCE RESERVATION
```

## 4. Resource Limits

Limits participate in runtime resource enforcement.

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "1"
    memory: "1Gi"
```

Keep these concepts separate:

```text
REQUEST → SCHEDULING / ACCOUNTING
LIMIT → RUNTIME CONSTRAINT
REQUEST ≠ LIMIT
```

CPU and memory enforcement are not identical:

```text
CPU ENFORCEMENT SEMANTICS ≠ MEMORY ENFORCEMENT SEMANTICS
```

## 5. CPU Scheduling vs Runtime CPU

Scheduling path:

```text
CPU request
  ↓
Node eligibility
  ↓
Node allocatable
  ↓
Existing commitments
  ↓
Resource fit
  ↓
Scheduler decision
```

Runtime path:

```text
Application CPU demand
  ↓
CPU availability
  ↓
Runtime CPU control
  ↓
Possible throttling
  ↓
Possible latency / throughput impact
```

Therefore:

```text
INSUFFICIENT CPU SCHEDULING CAPACITY ≠ CPU THROTTLING
CPU LIMIT PRESENT ≠ CPU THROTTLING PROVEN
HIGH CPU ≠ CPU THROTTLING PROVEN
```

A production conclusion should correlate CPU demand, limit configuration, throttling metrics, workload impact, and timestamps.

## 6. Memory Enforcement and Failure Classification

Memory behavior can depend on Kubernetes version, cgroup version, runtime implementation, and feature/configuration state.

For version-sensitive behavior verify:

```text
Kubernetes version
cgroup version
feature/configuration state
runtime evidence
```

Classify memory incidents:

```text
Memory symptom
 |
 +-- Container terminated?
 |     +-- OOMKilled?
 |     +-- exit code?
 |     +-- previous logs?
 |
 +-- Pod evicted?
 |     +-- eviction reason?
 |     +-- node MemoryPressure?
 |
 +-- Node-level OOM?
 |
 +-- Application-controlled exit?
 |
 +-- Still running?
```

Production rules:

```text
OOMKilled ≠ MEMORY LEAK PROVEN
CONTAINER OOM ≠ NODE-PRESSURE EVICTION
```

## 7. Preserve Previous-Container Evidence

Inspect:

```bash
kubectl get pod <pod> -n <namespace>
kubectl describe pod <pod> -n <namespace>
kubectl logs <pod> -n <namespace> -c <container> --previous
```

Capture restart count, last state, termination reason, exit code, timestamps, and previous logs.

```text
CURRENT CONTAINER HEALTHY ≠ PREVIOUS CONTAINER WAS HEALTHY
```

## 8. Kubernetes QoS Classes

Kubernetes assigns Pods one of:

```text
Guaranteed
Burstable
BestEffort
```

Inspect the actual value:

```bash
kubectl get pod <pod> -n <namespace> -o jsonpath="{.status.qosClass}"
```

QoS is classification evidence, not a health score:

```text
QoS CLASS ≠ ROOT CAUSE
Guaranteed ≠ IMMUNE FROM FAILURE
Burstable ≠ BAD CONFIGURATION
BestEffort ≠ ZERO RESOURCES
```

### Guaranteed

With traditional container-level resources, Guaranteed classification requires the applicable CPU and memory requests and limits to satisfy Kubernetes Guaranteed requirements for every container.

Typical example:

```yaml
resources:
  requests:
    cpu: "1"
    memory: "1Gi"
  limits:
    cpu: "1"
    memory: "1Gi"
```

Always inspect the assigned QoS class rather than inferring it from an incomplete manifest.

### Burstable

A Pod that does not meet Guaranteed requirements but has applicable CPU or memory resource configuration can be `Burstable`.

`Burstable` does not itself indicate a problem.

### BestEffort

A Pod with no applicable CPU or memory requests or limits can be `BestEffort`.

This does not mean it receives zero resources.

## 9. Pod-Level Resources

Modern Kubernetes versions can support Pod-level CPU and memory resources.

Use:

```text
CONTAINER RESOURCES
        +
APPLICABLE POD-LEVEL RESOURCES
        ↓
COMPLETE RESOURCE INTERPRETATION
```

Because this behavior is version/feature sensitive:

```text
VERIFY CLUSTER VERSION
VERIFY FEATURE STATE
VERIFY RENDERED POD CONFIGURATION
```

## 10. QoS Is Not Priority

```text
QoS CLASS ≠ PriorityClass
```

QoS and Pod priority are separate mechanisms.

## 11. QoS and Node-Pressure Eviction

Investigate:

```text
Node pressure
  ↓
Identify pressured resource
  ↓
Inspect requests and usage
  ↓
Inspect Pod priority
  ↓
Inspect QoS
  ↓
Inspect eviction evidence
  ↓
Build timeline
```

Do not reduce eviction analysis to QoS alone.

```text
QoS CLASS ≠ COMPLETE EVICTION ORDER
EVICTED ≠ QoS ALONE PROVES ROOT CAUSE
```

## 12. LimitRange

Inspect:

```bash
kubectl get limitrange -n <namespace>
kubectl describe limitrange -n <namespace>
```

Admission/defaulting can cause the submitted configuration and rendered Pod to differ:

```text
SUBMITTED CONFIGURATION ≠ NECESSARILY RENDERED CONFIGURATION
```

Policies also change over time:

```text
CURRENT LimitRange ≠ PROOF OF HISTORICAL POD DEFAULTING
```

Use the affected Pod and timeline as evidence.

## 13. ResourceQuota

Conceptually:

```text
LimitRange
  ↓
Per-object/container resource policy/defaulting

ResourceQuota
  ↓
Aggregate namespace resource governance
```

Inspect:

```bash
kubectl get resourcequota -n <namespace>
kubectl describe resourcequota -n <namespace>
```

```text
QUOTA FAILURE ≠ SCHEDULER CAPACITY FAILURE
```

Adding worker nodes does not inherently resolve a namespace quota rejection.

## 14. Complete Pod Resource Model

Do not inspect only the main application container.

Consider:

```text
Application containers
Sidecars
Init containers
Pod overhead
Applicable Pod-level resources
```

Therefore:

```text
APPLICATION CONTAINER RESOURCES ≠ COMPLETE POD RESOURCE MODEL
```

Validate version-sensitive effective-resource semantics against the actual Kubernetes version.

## 15. kubectl top Is Runtime Evidence

When metrics are available:

```bash
kubectl top pod <pod> -n <namespace> --containers
kubectl top node <node>
```

But:

```text
kubectl top → RUNTIME CONSUMPTION EVIDENCE
kubectl top ≠ SCHEDULER CAPACITY MODEL
LOW kubectl top USAGE ≠ POD CAN SCHEDULE
```

Use requests, allocatable resources, scheduling constraints, and scheduler events for scheduling analysis.

## 16. Resource Fragmentation

Suppose:

```text
Node A → 1 CPU request headroom
Node B → 1 CPU request headroom
Node C → 1 CPU request headroom
```

A Pod requiring `2 CPU` cannot combine capacity across nodes.

```text
AGGREGATE FREE CAPACITY ≠ POD FIT
```

## 17. Eligibility Before Capacity

A node can have resources but remain ineligible because of scheduling constraints.

Use:

```text
NODE ELIGIBILITY
  ↓
RESOURCE FIT
  ↓
SCHEDULER DECISION
```

Therefore:

```text
NODE HAS RESOURCES ≠ NODE IS ELIGIBLE FOR THIS POD
```

Selectors, affinity, taints, tolerations, and dedicated node pools are covered more deeply in Parts 3.3–3.5.

## 18. Resource Failure vs Infrastructure Shortage

Possible failure domains include:

```text
Application behavior
Workload resource configuration
Admission/defaulting
Namespace policy
Scheduler eligibility
Resource fragmentation
Node capacity
Runtime enforcement
Node pressure
```

Therefore:

```text
RESOURCE FAILURE ≠ INFRASTRUCTURE EXPANSION REQUIRED
```

Scale infrastructure only when evidence demonstrates that capacity is the actionable constraint.

## 19. Minimum Supported Blast Radius

Determine whether evidence supports:

```text
Container
Pod
Workload
Namespace
Node
Node pool
Multiple workloads
Cluster
```

State only the smallest scope directly supported by evidence.

```text
MINIMUM SUPPORTED BLAST RADIUS
```

One OOMKilled container does not prove cluster-wide memory exhaustion.

## 20. Build a Timeline

Correlate:

```text
T0 Deployment/configuration change
T1 Pod admitted
T2 Pod scheduled
T3 Load changed
T4 Resource behavior changed
T5 Throttling/OOM/pressure evidence appeared
T6 Application impact began
T7 Restart/eviction occurred
T8 Recovery occurred
```

```text
CORRELATION ≠ CAUSATION
```

A coherent timeline is still necessary for causal analysis.

## 21. Preserve Evidence Before Remediation

Do not immediately change requests, limits, replicas, node count, node-pool size, or QoS-producing resource configuration.

Use:

```text
EVIDENCE FIRST
REMEDIATION SECOND
```

because:

```text
REMEDIATION CAN DESTROY INCIDENT EVIDENCE
```

## 22. Respect Authoritative Desired-State Ownership

Resource configuration may be owned by:

```text
TrueFoundry
GitOps
Helm
Terraform
another deployment controller
```

The observed Pod is evidence; it may not be the authoritative configuration source.

```text
DO NOT PATCH THE SYMPTOM
WHEN ANOTHER SYSTEM OWNS DESIRED STATE
```

## 23. Production-Safe Evidence Commands

Useful observational commands include:

```bash
kubectl config current-context
kubectl get pod <pod> -n <namespace> -o wide
kubectl get pod <pod> -n <namespace> -o yaml
kubectl describe pod <pod> -n <namespace>
kubectl get pod <pod> -n <namespace> -o jsonpath="{.status.qosClass}"
kubectl logs <pod> -n <namespace> -c <container> --previous
kubectl get events -n <namespace> --sort-by=.lastTimestamp
kubectl get limitrange -n <namespace>
kubectl describe limitrange -n <namespace>
kubectl get resourcequota -n <namespace>
kubectl describe resourcequota -n <namespace>
kubectl describe node <node>
kubectl top pod <pod> -n <namespace> --containers
kubectl top node <node>
```

Not every cluster exposes every metric or retains every historical event. Missing evidence from one source does not prove an event did not occur.

## 24. Production Anti-Patterns

Avoid:

```text
Pod Pending → increase CPU/memory
High CPU → increase CPU limit
CPU limit exists → throttling caused incident
OOMKilled → memory leak
BestEffort → bad workload
Burstable → performance problem
Guaranteed → resource failure impossible
Low node CPU usage → enough scheduler capacity
Low kubectl top → scheduler should place Pod
Evicted → QoS caused it
Quota error → add worker nodes
Resource failure → scale infrastructure
```

Each conclusion skips required evidence.

## 25. Production Investigation Workflow

1. Establish workload identity.
2. Capture authoritative desired configuration.
3. Capture rendered Pod resources.
4. Determine QoS class.
5. Inspect `LimitRange` and `ResourceQuota`.
6. Establish scheduler evidence.
7. Establish node eligibility.
8. Establish resource fit.
9. Capture runtime consumption.
10. Check CPU throttling evidence.
11. Classify memory failures.
12. Inspect previous-container evidence.
13. Inspect node-pressure and eviction evidence.
14. Build the incident timeline.
15. Determine Minimum Supported Blast Radius.
16. Identify Lowest Proven Healthy Layer.
17. Identify First Failed Transition.
18. Assign Evidence Confidence.
19. Identify Current Actionable Owner.
20. Remediate through the authoritative owner.

## 26. Resource Evidence Handoff Contract

Use:

```text
Environment:
Cluster:
Namespace:
Workload:
Pod:
Pod UID:
Node:
Container:
Incident timestamp:

Authoritative desired-state owner:

Desired CPU request:
Desired CPU limit:
Desired memory request:
Desired memory limit:

Rendered CPU request:
Rendered CPU limit:
Rendered memory request:
Rendered memory limit:

Applicable Pod-level resources:
Pod overhead:
QoS class:

LimitRange:
ResourceQuota:

Scheduler state:
Scheduler events:
Node eligibility:
Node allocatable:
Relevant requested resources:

Runtime CPU:
Runtime memory:
CPU throttling evidence:

Restart count:
Last state:
Termination reason:
Exit code:
Previous-container evidence:

Node conditions:
MemoryPressure:
DiskPressure:
PIDPressure:
Eviction evidence:

Minimum Supported Blast Radius:
Lowest Proven Healthy Layer:
First Failed Transition:

Evidence confidence:
Current Actionable Owner:
Requested Action:
```

Separate observation from interpretation.

## 27. Evidence Confidence

Classify conclusions as:

```text
PROVEN
SUPPORTED
UNKNOWN
ASSUMED
```

`PROVEN` means directly established by authoritative or primary evidence.

`SUPPORTED` means strongly indicated but not completely proven.

`UNKNOWN` means required evidence is unavailable or insufficient.

`ASSUMED` means a hypothesis has not yet been supported by sufficient evidence.

```text
ASSUMED ≠ ROOT CAUSE
```

## 28. Final Production Rules

```text
UNKNOWN WORKLOAD IDENTITY = NO RESOURCE CHANGE

DESIRED ≠ RENDERED ≠ ACCOUNTED ≠ CONSUMED

REQUEST ≠ USAGE
REQUEST ≠ LIMIT

REQUEST → SCHEDULING / ACCOUNTING
LIMIT → RUNTIME CONSTRAINT

LOW UTILIZATION ≠ LOW RESOURCE RESERVATION

kubectl top ≠ SCHEDULER CAPACITY MODEL

APPLICATION CONTAINER RESOURCES
    ≠ COMPLETE POD RESOURCE MODEL

QoS CLASS ≠ ROOT CAUSE
QoS CLASS ≠ PriorityClass
QoS CLASS ≠ COMPLETE EVICTION ORDER

Guaranteed ≠ IMMUNE FROM FAILURE
Burstable ≠ BAD CONFIGURATION
BestEffort ≠ ZERO RESOURCES

EVICTED ≠ QoS ALONE PROVES ROOT CAUSE

INSUFFICIENT CPU SCHEDULING CAPACITY
    ≠ CPU THROTTLING

CPU LIMIT PRESENT ≠ CPU THROTTLING PROVEN
HIGH CPU ≠ CPU THROTTLING PROVEN

CPU THROTTLING SEMANTICS
    ≠ MEMORY ENFORCEMENT SEMANTICS

OOMKilled ≠ MEMORY LEAK PROVEN
CONTAINER OOM ≠ NODE-PRESSURE EVICTION

CURRENT CONTAINER HEALTHY
    ≠ PREVIOUS CONTAINER WAS HEALTHY

SUBMITTED CONFIGURATION
    ≠ NECESSARILY RENDERED CONFIGURATION

CURRENT LimitRange
    ≠ PROOF OF HISTORICAL POD DEFAULTING

QUOTA FAILURE ≠ SCHEDULER CAPACITY FAILURE

AGGREGATE FREE CAPACITY ≠ POD FIT

NODE HAS RESOURCES
    ≠ NODE IS ELIGIBLE FOR THIS POD

RESOURCE FAILURE
    ≠ INFRASTRUCTURE EXPANSION REQUIRED

REMEDIATION CAN DESTROY INCIDENT EVIDENCE

ASSUMED ≠ ROOT CAUSE

SYMPTOM ≠ FAILURE DOMAIN ≠ OWNER
```

## 29. Completion Check

Before considering the investigation complete, confirm:

- affected workload, Pod, UID, node, container, and incident timestamp;
- authoritative desired-state owner;
- desired and rendered resources;
- assigned QoS class;
- relevant `LimitRange` and `ResourceQuota`;
- scheduler state and events;
- eligible nodes and resource fit;
- runtime CPU and memory evidence;
- CPU throttling evidence status;
- memory failure classification;
- previous-container evidence;
- node-pressure and eviction evidence;
- Minimum Supported Blast Radius;
- Lowest Proven Healthy Layer;
- First Failed Transition;
- Evidence Confidence;
- Current Actionable Owner; and
- Requested Action.

If these cannot be answered, continue evidence collection before changing resource configuration.

## Next Tutorial

**Part 3.3 — Node Selectors & Node Affinity**
