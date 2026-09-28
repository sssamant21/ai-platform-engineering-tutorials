# Part 3.1 --- CPU & Memory Resource Management

## Purpose

CPU and memory problems are frequently diagnosed from the wrong layer. A
Pod can remain Pending while node utilization appears low. An
application can become slow while CPU usage appears high without CPU
throttling being the proven cause. A container can be reported as
`OOMKilled` without proving an application memory leak or a cluster-wide
memory shortage.

This tutorial establishes an evidence-driven resource model for
TrueFoundry-managed Kubernetes workloads. It separates desired
configuration, rendered Kubernetes state, scheduler accounting, runtime
enforcement, observed consumption, and application behavior.

The central production question is:

> **What CPU and memory did the workload require, what did Kubernetes
> actually render and account for, which nodes were eligible, what
> resources were allocatable there, what did the workload consume after
> placement, and where is the first proven resource-related failure?**

## Learning Objectives

After completing this tutorial, you should be able to:

-   explain Kubernetes CPU and memory resource units;
-   distinguish resource requests from limits and observed usage;
-   distinguish node capacity from node allocatable resources;
-   explain why low current utilization does not prove scheduling
    capacity;
-   identify the effective resource requirement of a Pod rather than
    inspecting only one application container;
-   distinguish aggregate cluster capacity from capacity that can
    satisfy one Pod;
-   separate scheduling failures from runtime CPU and memory failures;
-   distinguish CPU throttling, container OOM, node memory pressure, and
    node-pressure eviction;
-   collect production-safe resource evidence;
-   determine the Minimum Supported Blast Radius;
-   identify the Lowest Proven Healthy Layer and First Failed
    Transition;
-   locate the authoritative resource configuration before remediation.

## 1. Production Resource Lifecycle

``` text
TrueFoundry Workload Configuration
        ↓
Rendered Kubernetes Workload
        ↓
Pod-Level Resources (when applicable)
        +
Container Resources
        +
Init / Sidecar Requirements
        +
Pod Overhead (when applicable)
        ↓
Effective Pod Resource Requirement
        ↓
Scheduler
        ↓
Eligible Node Set
        ↓
Node Allocatable Resources
        ↓
Scheduling Decision
        ↓
Container Runtime
        ↓
Runtime Resource Enforcement
        ↓
Observed CPU / Memory Consumption
        ↓
Application Behavior
```

Four operational states:

``` text
DESIRED → RENDERED → SCHEDULED → CONSUMED
```

-   **Desired** --- intended configuration in the authoritative
    deployment system.
-   **Rendered** --- Kubernetes resource configuration actually
    produced.
-   **Scheduled** --- requirements and constraints Kubernetes evaluates
    for placement.
-   **Consumed** --- CPU and memory observed at runtime.

``` text
DESIRED ≠ RENDERED ≠ SCHEDULED ≠ CONSUMED
```

## 2. Kubernetes CPU Fundamentals

Common CPU quantities:

``` text
1 CPU     = 1000m
0.5 CPU   = 500m
0.25 CPU  = 250m
```

``` yaml
resources:
  requests:
    cpu: "500m"
  limits:
    cpu: "1"
```

``` text
CPU REQUEST ≠ CPU USAGE
CPU REQUEST ≠ CPU LIMIT
CPU LIMIT ≠ GUARANTEED CPU CONSUMPTION
```

CPU is generally treated as a compressible resource. When a CPU limit is
enforced, runtime behavior can include throttling:

``` text
CPU LIMIT REACHED
        ↓
CPU TIME MAY BE THROTTLED
        ≠
MEMORY OOM TERMINATION
```

``` text
HIGH CPU ≠ CPU THROTTLING PROVEN
```

## 3. Kubernetes Memory Fundamentals

``` yaml
resources:
  requests:
    memory: "2Gi"
  limits:
    memory: "4Gi"
```

Memory quantities commonly use `Ki`, `Mi`, `Gi`, or decimal units such
as `k`, `M`, and `G`.

A critical unit warning:

``` text
400m memory ≠ 400Mi
```

``` text
MEMORY REQUEST ≠ MEMORY LIMIT ≠ CURRENT MEMORY USAGE
```

Memory-limit pressure can cause kernel memory enforcement to terminate
processes:

``` text
MEMORY LIMIT PRESSURE
        ↓
KERNEL OOM ENFORCEMENT MAY OCCUR
```

Do not assume that reaching a displayed value creates an immediate
deterministic termination sequence; validate the actual termination and
runtime evidence.

## 4. Requests and Limits

``` text
REQUEST → scheduling / resource-accounting input
LIMIT   → runtime resource constraint
```

``` yaml
resources:
  requests:
    cpu: "500m"
    memory: "2Gi"
  limits:
    cpu: "2"
    memory: "4Gi"
```

The scheduler does not simply inspect instantaneous utilization and
choose an idle-looking node.

``` text
SCHEDULER DECISION ≠ BASED ON CURRENT kubectl top USAGE
```

Detailed QoS behavior is covered in Part 3.2.

## 5. Effective Pod Resource Requirements

A Pod can include an application container, sidecars, init containers,
and other platform/runtime components. Depending on Kubernetes version
and feature state, Pod-level resources and Pod overhead can also
participate in the effective resource model.

Use:

``` text
CHECK CLUSTER VERSION / FEATURE STATE
        ↓
CHECK POD-LEVEL RESOURCES WHEN APPLICABLE
        ↓
CHECK CONTAINER RESOURCES
        ↓
CHECK INIT / SIDECAR REQUIREMENTS
        ↓
CHECK POD OVERHEAD WHEN APPLICABLE
        ↓
DETERMINE EFFECTIVE POD REQUIREMENT
```

``` text
APPLICATION CONTAINER REQUEST ≠ EFFECTIVE POD SCHEDULING REQUIREMENT
```

Follow the resource-calculation semantics applicable to the actual
Kubernetes version and enabled features.

## 6. Node Capacity and Node Allocatable

``` text
Infrastructure VM Capacity
        ↓
Kubernetes Node Capacity
        ↓
Kubernetes Node Allocatable
        ↓
Already Requested Resources
        ↓
Remaining Request Headroom
        ↓
Eligible Capacity for This Pod
```

Example:

``` text
Infrastructure node:
CPU:     16
Memory:  64 GiB

Kubernetes allocatable:
CPU:     15.2
Memory:  58 GiB
```

``` text
VM SIZE
    ≠ NODE CAPACITY
    ≠ NODE ALLOCATABLE
    ≠ UNREQUESTED CAPACITY
    ≠ SCHEDULABLE CAPACITY FOR THIS POD
```

## 7. Requests Versus Observed Utilization

Consider:

``` text
Node allocatable CPU:       8000m
Existing CPU requests:      7500m
Observed CPU consumption:    900m
New Pod CPU request:        1500m
```

Request headroom is:

``` text
8000m - 7500m = 500m
```

The new `1500m` request cannot fit on that node based merely on low
instantaneous utilization.

``` text
CURRENT LOW USAGE ≠ SCHEDULABLE CAPACITY
LOW UTILIZATION ≠ LOW RESOURCE RESERVATION
kubectl top ≠ SCHEDULER CAPACITY MODEL
```

`kubectl top` is runtime-consumption evidence. Scheduler events, Pod
requests, node allocatable resources, placement constraints, and
existing request accounting answer scheduling questions.

## 8. Resource Fragmentation

Suppose:

``` text
Node A request headroom: 1500m
Node B request headroom: 1500m
Node C request headroom: 1500m

Cluster aggregate:       4500m
New Pod request:         3000m
```

No individual eligible node may be able to satisfy the Pod.

``` text
TOTAL CLUSTER CAPACITY ≠ SCHEDULABLE CAPACITY FOR THIS POD
RESOURCE HEADROOM CAN EXIST WHILE PLACEMENT CAPACITY DOES NOT
```

## 9. Eligibility Before Capacity

Before asking whether a node has enough CPU or memory, determine whether
the node is actually eligible for the Pod.

Eligibility can be affected by:

-   node readiness;
-   unschedulable state;
-   `nodeSelector`;
-   node affinity;
-   taints and tolerations;
-   topology constraints;
-   storage constraints.

``` text
NODE HAS CAPACITY ≠ POD CAN SCHEDULE THERE
```

These mechanisms are covered in greater depth later in Part 3.

## 10. Pending Is a State, Not a Root Cause

``` text
Pod Pending
    ↓
Pod Conditions
    ↓
Scheduler Events
    ↓
Eligible Node Set
    ↓
Placement Constraints
    ↓
Resource Requirements
    ↓
Node Allocatable
```

``` text
PENDING IS A STATE, NOT A ROOT CAUSE
POD PENDING ≠ INSUFFICIENT CPU/MEMORY PROVEN
```

A Pending Pod can reflect resource shortages, placement rules, taints,
storage dependencies, node state, or other constraints.

## 11. Scheduling CPU Versus Runtime CPU

Scheduling path:

``` text
CPU Request → Scheduler → Eligible Node → Node Allocatable CPU → Placement
```

Runtime path:

``` text
Application CPU Demand
        ↓
Runtime CPU Enforcement
        ↓
CPU Time Available
        ↓
Possible Throttling
        ↓
Application Latency
```

``` text
INSUFFICIENT CPU FOR SCHEDULING ≠ CPU THROTTLING
APPLICATION SLOW ≠ CPU SHORTAGE PROVEN
```

## 12. Memory Incident Classification

Avoid:

``` text
OOMKilled → increase memory
```

Instead:

``` text
Container Restart
        ↓
Inspect Termination State
        ↓
Inspect Previous Container Evidence
        ↓
Inspect Configured Request / Limit
        ↓
Inspect Observed Memory
        ↓
Inspect Node Pressure
        ↓
Inspect Kubernetes Events
        ↓
Classify Failure
```

Possible classifications include container/cgroup OOM, node memory
pressure, node-pressure eviction, application memory growth, a large
transient allocation, resource mis-sizing, or another termination cause.

``` text
CONTAINER OOM ≠ NODE MEMORY PRESSURE ≠ NODE-PRESSURE EVICTION
OOMKilled ≠ MEMORY LEAK PROVEN
```

## 13. Preserve Previous-Container Evidence

Useful evidence includes restart count, last termination state,
termination reason, exit code, timestamps, events, and previous logs.

``` bash
kubectl logs <pod> -n <namespace> -c <container> --previous
```

``` text
CURRENT CONTAINER HEALTH ≠ PREVIOUS FAILURE EVIDENCE
```

## 14. Node Resource Evidence

Inspect:

``` text
Ready
MemoryPressure
DiskPressure
PIDPressure
Unschedulable
Capacity
Allocatable
Allocated requests
Allocated limits
```

Useful read-only commands include:

``` bash
kubectl describe node <node>
```

and:

``` bash
kubectl get node <node>   -o jsonpath='{.status.capacity}{"\n"}{.status.allocatable}{"\n"}'
```

``` text
NODE MemoryPressure ≠ CONTAINER MEMORY LIMIT EXCEEDED
```

## 15. Production-Safe Resource Evidence

Start by proving identity:

``` text
Environment
Cluster
Namespace
Workload
Pod
Pod UID
Container
Node
Timestamp
```

Useful observational commands include:

``` bash
kubectl config current-context
kubectl get pod -n <namespace>
kubectl describe pod <pod> -n <namespace>
kubectl get node
kubectl describe node <node>
```

When a metrics pipeline is available:

``` bash
kubectl top pod -n <namespace>
kubectl top node
```

Metrics are runtime evidence, not a substitute for scheduler accounting.

``` text
UNKNOWN WORKLOAD IDENTITY = NO RESOURCE CHANGE
```

## 16. Resource Misconfiguration Versus Infrastructure Shortage

Suppose:

``` text
Workload CPU request:               16 CPU
Largest eligible node allocatable:   8 CPU
```

This supports a scheduling incompatibility, but it does not
independently determine the correct remediation.

Potential remediation domains include workload sizing, infrastructure
sizing, placement configuration, or architecture.

``` text
INSUFFICIENT CAPACITY EVENT ≠ INFRASTRUCTURE MUST BE EXPANDED
```

## 17. Minimum Supported Blast Radius

Classify the smallest scope supported by evidence:

``` text
Container
Pod
Replica
Workload
Namespace
Node
Node pool
Cluster
Multiple workloads
```

``` text
MINIMUM SUPPORTED BLAST RADIUS
```

One affected replica with healthy peers and a healthy node does not
prove a cluster-wide resource problem.

## 18. Timeline Correlation

Capture relevant times:

``` text
Incident start
Pod creation
Scheduling attempt
Container start
Restart
OOM / termination
Node-pressure transition
Deployment change
Resource-configuration change
Scaling event
Traffic change
```

``` text
TIMING CORRELATION ≠ ROOT CAUSE PROOF
```

## 19. Protect Incident Evidence

Actions such as restarting, deleting a Pod, scaling, changing requests
or limits, moving a workload, or resizing a node pool can alter or
destroy incident evidence.

``` text
REMEDIATION CAN DESTROY INCIDENT EVIDENCE
```

When service-restoration urgency permits:

``` text
Collect Minimum Required Evidence
        ↓
Classify Failure
        ↓
Identify Current Actionable Owner
        ↓
Perform Approved Remediation
```

## 20. Authoritative Resource Configuration

``` text
Observed Pod
    ↓
Owning Controller
    ↓
Rendered Workload
    ↓
TrueFoundry / GitOps / Deployment Source
    ↓
Authoritative Configuration
```

``` text
POD ≠ AUTHORITATIVE RESOURCE CONFIGURATION
Observed Drift ≠ Permission to Patch
DO NOT PATCH THE SYMPTOM WHEN ANOTHER SYSTEM OWNS DESIRED STATE
```

## 21. Production Scenarios

### Scenario 1 --- Pending Pod with Proven CPU Constraint

``` text
Pod Pending
→ Scheduler evidence reports insufficient CPU
→ Verify effective CPU request
→ Determine eligible nodes
→ Compare request against allocatable/request headroom
→ Confirm First Failed Transition
```

Do not automatically conclude that the cluster must be expanded.

### Scenario 2 --- Node Looks Idle but Pod Is Pending

``` text
kubectl top → low CPU utilization

Node allocatable:  8000m
Existing requests: 7500m
New Pod request:   1500m
```

Only `500m` request headroom remains.

``` text
USAGE ≠ REQUEST ACCOUNTING
```

### Scenario 3 --- Aggregate Capacity Exists but the Pod Cannot Fit

``` text
Node A: 1500m headroom
Node B: 1500m headroom
Node C: 1500m headroom
Aggregate: 4500m
Pod request: 3000m
```

``` text
AGGREGATE CAPACITY ≠ POD PLACEMENT CAPACITY
```

### Scenario 4 --- Application Slow with High CPU

Investigate CPU request, CPU limit, observed demand, throttling
evidence, node contention, application behavior, and traffic/concurrency
changes.

``` text
HIGH CPU ≠ CPU THROTTLING PROVEN
```

### Scenario 5 --- Container Restart with Memory Evidence

Inspect last termination state, previous logs, request, limit, observed
memory, node pressure, eviction events, and the incident timeline.

``` text
OOMKilled ≠ MEMORY LEAK PROVEN
```

### Scenario 6 --- Capacity Exists on an Ineligible Node

If a node has resource headroom but does not satisfy placement
constraints:

``` text
NODE HAS CAPACITY ≠ POD CAN SCHEDULE THERE
```

## 22. Production Anti-Patterns

Avoid these shortcuts:

``` text
Pod Pending → immediately increase node count
Application slow → immediately increase CPU
Container restarted → immediately increase memory
Node CPU low → scheduler must have CPU capacity
Cluster aggregate CPU sufficient → this Pod can schedule
OOMKilled → application has a memory leak
Patch observed Pod → permanent incident resolution
```

Each skips evidence, ownership, or both.

## 23. Evidence Handoff Contract

Use a handoff such as:

``` text
Environment:
Cluster:
Namespace:
Workload:
Pod:
Pod UID:
Container:
Node:

Incident Window:
Observed Symptom:

CPU Request:
CPU Limit:
Memory Request:
Memory Limit:
Effective Pod Requirement:

Node Capacity:
Node Allocatable:
Allocated Requests:

Observed CPU:
Observed Memory:

Scheduling State:
Scheduler Evidence:
Node Pressure Conditions:

Lowest Proven Healthy Layer:
First Failed Transition:
Minimum Supported Blast Radius:

Evidence Confidence:
PROVEN / SUPPORTED / UNKNOWN / ASSUMED

Authoritative Configuration Owner:
Current Actionable Owner:
Requested Action:
```

## 24. Evidence Confidence

Classify important conclusions as:

``` text
PROVEN
SUPPORTED
UNKNOWN
ASSUMED
```

Examples:

-   **PROVEN** --- scheduler event reports insufficient CPU for the
    affected Pod.
-   **SUPPORTED** --- a resource configuration change immediately
    precedes failures for the same workload.
-   **UNKNOWN** --- whether the configured CPU request accurately
    reflects application requirements.
-   **ASSUMED** --- the node pool must be enlarged because the Pod is
    Pending.

``` text
ASSUMED ≠ ROOT CAUSE
```

## 25. Production Investigation Workflow

``` text
VERIFY DEPLOYMENT IDENTITY
        ↓
DEFINE INCIDENT WINDOW
        ↓
DETERMINE MINIMUM SUPPORTED BLAST RADIUS
        ↓
PROVE DESIRED RESOURCE CONFIGURATION
        ↓
PROVE RENDERED RESOURCE CONFIGURATION
        ↓
DETERMINE EFFECTIVE POD REQUIREMENT
        ↓
PROVE SCHEDULING STATE
        ↓
DETERMINE ELIGIBLE NODE SET
        ↓
PROVE NODE ALLOCATABLE CAPACITY
        ↓
COMPARE RESOURCE REQUEST ACCOUNTING
        ↓
SEPARATE SCHEDULING FROM RUNTIME
        ↓
CORRELATE OBSERVED CPU / MEMORY
        ↓
CHECK THROTTLING / OOM / NODE PRESSURE
        ↓
FIND LOWEST PROVEN HEALTHY LAYER
        ↓
FIND FIRST FAILED TRANSITION
        ↓
IDENTIFY AUTHORITATIVE CONFIGURATION
        ↓
IDENTIFY CURRENT ACTIONABLE OWNER
        ↓
PERFORM APPROVED REMEDIATION
        ↓
VALIDATE RESOURCE STATE
        ↓
VALIDATE APPLICATION RECOVERY
```

Do not ask only, "Does the cluster have enough CPU and memory?"

Ask:

> What did this workload require, which nodes were actually eligible,
> what resources were allocatable there, what did the scheduler account
> for, what happened after placement, and what evidence proves the first
> failure?

## 26. Final Production Rules

``` text
REQUEST ≠ USAGE
REQUEST ≠ LIMIT
DESIRED ≠ RENDERED ≠ SCHEDULED ≠ CONSUMED
NODE CAPACITY ≠ NODE ALLOCATABLE
LOW UTILIZATION ≠ LOW RESOURCE RESERVATION
kubectl top ≠ SCHEDULER CAPACITY MODEL
TOTAL CLUSTER CAPACITY ≠ CAPACITY FOR THIS POD
NODE HAS CAPACITY ≠ POD CAN SCHEDULE THERE
APPLICATION CONTAINER REQUEST ≠ EFFECTIVE POD REQUIREMENT
PENDING IS A STATE, NOT A ROOT CAUSE
POD PENDING ≠ INSUFFICIENT CPU/MEMORY PROVEN
CPU THROTTLING ≠ MEMORY OOM
CONTAINER OOM ≠ NODE-PRESSURE EVICTION
OOMKilled ≠ MEMORY LEAK PROVEN
INSUFFICIENT CAPACITY EVENT ≠ INFRASTRUCTURE MUST BE EXPANDED
POD ≠ AUTHORITATIVE RESOURCE CONFIGURATION
REMEDIATION CAN DESTROY INCIDENT EVIDENCE
ASSUMED ≠ ROOT CAUSE
SYMPTOM ≠ FAILURE DOMAIN ≠ OWNER
```

## 27. Completion Check

After Part 3.1, you should be able to answer:

1.  What CPU and memory resources were intended?
2.  What resources were actually rendered into Kubernetes?
3.  What is the effective Pod resource requirement?
4.  What state is the Pod currently in?
5.  Which nodes are actually eligible?
6.  What CPU and memory are allocatable on those nodes?
7.  What resources are already requested?
8.  What CPU and memory are currently being consumed?
9.  Is the problem scheduling-related or runtime-related?
10. Is there evidence of CPU throttling, OOM, node pressure, or
    eviction?
11. What is the Minimum Supported Blast Radius?
12. What is the Lowest Proven Healthy Layer?
13. What is the First Failed Transition?
14. Which system owns the authoritative resource configuration?
15. Who is the Current Actionable Owner?
16. What evidence will prove recovery after remediation?

If those questions cannot be answered, the resource investigation is not
yet complete.

## Next Tutorial

**Part 3.2 --- Requests, Limits & Kubernetes QoS**

Part 3.2 builds on this resource model and examines requests, limits,
Kubernetes QoS classes, CPU throttling, memory enforcement, and
resource-pressure behavior in greater depth.
