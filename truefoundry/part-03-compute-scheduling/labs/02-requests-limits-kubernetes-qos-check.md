# Part 3.2 — Requests, Limits & Kubernetes QoS

> **[SAFE-READ] Production Validation Lab**

## Purpose

This lab validates CPU and memory requests, limits, Kubernetes QoS classification, namespace resource policy, scheduling evidence, runtime resource consumption, and resource-related failure evidence for an existing Kubernetes workload.

The lab is observational. It must not change the workload, Pod, requests, limits, replicas, labels, taints, affinity, priority, node configuration, or node-pool configuration.

The central investigation question is:

> **What resource configuration was intended, what did Kubernetes actually render and account for, what QoS class was assigned, what happened at runtime, and where is the first proven resource-related failure?**

---

## Safety Rules

This lab permits read-only investigation such as:

```text
kubectl config current-context
kubectl get
kubectl describe
kubectl logs
kubectl top
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
kubectl label
kubectl taint
```

Do not modify:

```text
requests
limits
replicas
PriorityClass
node labels
taints
tolerations
affinity
node selectors
node-pool configuration
```

Production guardrail:

```text
UNKNOWN WORKLOAD IDENTITY = NO RESOURCE CHANGE
```

---

## Prerequisites

You need:

- read access to the target Kubernetes cluster;
- permission to inspect Pods, workloads, events, nodes, `LimitRange`, and `ResourceQuota`;
- access to logs if previous-container evidence is required;
- metrics access for `kubectl top` if the cluster exposes the metrics pipeline.

The absence of `kubectl top` data does not invalidate the lab. Record runtime metrics as unavailable and continue with other evidence.

---

## Variables

For the examples below, identify:

```text
<namespace>
<pod>
<container>
<node>
<workload>
```

Do not guess these values.

---

# Phase 1 — Validate Cluster and Workload Identity

## Step 1 — Confirm the Current Kubernetes Context

Run:

```bash
kubectl config current-context
```

Record:

```text
Environment:
Cluster/context:
Investigation timestamp:
```

Confirm that the context is the intended environment before continuing.

---

## Step 2 — Identify the Namespace and Pod

Run:

```bash
kubectl get pods -n <namespace> -o wide
```

Record:

```text
Namespace:
Pod:
Pod IP:
Node:
Pod status:
Age:
```

Do not continue with resource conclusions until the affected Pod is identified.

---

## Step 3 — Capture the Pod UID

Run:

```bash
kubectl get pod <pod> -n <namespace> -o jsonpath="{.metadata.uid}{'\n'}"
```

Record:

```text
Pod UID:
```

The UID helps distinguish the current Pod from an earlier Pod with the same name pattern.

---

## Step 4 — Identify the Workload Owner

Run:

```bash
kubectl get pod <pod> -n <namespace> -o jsonpath="{.metadata.ownerReferences}{'\n'}"
```

If additional ownership resolution is required, inspect the referenced object using read-only commands.

Record:

```text
Immediate owner:
Higher-level workload:
Authoritative desired-state system if known:
```

Do not assume the Pod itself is the authoritative configuration source.

---

# Phase 2 — Inventory the Complete Pod Resource Model

## Step 5 — Inspect the Pod

Run:

```bash
kubectl get pod <pod> -n <namespace> -o yaml
```

Do not modify or reapply this output.

Identify:

- application containers;
- sidecars;
- init containers;
- CPU requests;
- CPU limits;
- memory requests;
- memory limits;
- Pod-level resources if present;
- Pod overhead if present.

Record:

```text
Application containers:
Sidecars:
Init containers:
Pod-level resources present: YES / NO / UNKNOWN
Pod overhead present: YES / NO / UNKNOWN
```

Production rule:

```text
APPLICATION CONTAINER RESOURCES
    ≠ COMPLETE POD RESOURCE MODEL
```

---

## Step 6 — Capture Container Requests and Limits

For each relevant container, record:

```text
Container:
CPU request:
CPU limit:
Memory request:
Memory limit:
```

Do not inspect only the primary application container.

Compare the values across all relevant containers.

---

## Step 7 — Check for Pod-Level Resources

Inspect the Pod YAML for Pod-level resource configuration.

Record:

```text
Pod-level CPU request:
Pod-level CPU limit:
Pod-level memory request:
Pod-level memory limit:
```

If the fields are not present:

```text
Pod-level resources: NOT OBSERVED
```

Do not assume unsupported behavior solely because the fields are absent.

Record the Kubernetes version later before making version-sensitive conclusions.

---

## Step 8 — Check Pod Overhead

Inspect:

```bash
kubectl get pod <pod> -n <namespace> -o jsonpath="{.spec.overhead}{'\n'}"
```

Record:

```text
Pod overhead:
```

If empty, record:

```text
Pod overhead: NOT OBSERVED
```

---

# Phase 3 — Validate QoS Classification

## Step 9 — Inspect the Assigned QoS Class

Run:

```bash
kubectl get pod <pod> -n <namespace> -o jsonpath="{.status.qosClass}{'\n'}"
```

Record exactly one observed value:

```text
QoS class:
```

Typical values include:

```text
Guaranteed
Burstable
BestEffort
```

Do not classify workload health from QoS alone.

Production rules:

```text
QoS CLASS ≠ ROOT CAUSE

Guaranteed ≠ IMMUNE FROM FAILURE
Burstable ≠ BAD CONFIGURATION
BestEffort ≠ ZERO RESOURCES
```

---

## Step 10 — Compare QoS With Rendered Resources

Compare the observed QoS class with:

- container CPU requests;
- container CPU limits;
- container memory requests;
- container memory limits;
- applicable Pod-level resources.

Record:

```text
Observed QoS:
Resource configuration consistent with observed class:
YES / NO / REQUIRES VERSION-SPECIFIC VALIDATION
```

Do not override the observed `.status.qosClass` with a manual guess.

---

# Phase 4 — Validate Kubernetes Version

## Step 11 — Record Kubernetes Version

Run:

```bash
kubectl version
```

Record:

```text
Client version:
Server version:
```

Use the server version when evaluating version-sensitive resource semantics.

For Pod-level resources or Memory QoS behavior, record:

```text
Version-sensitive behavior requires vendor validation:
YES / NO
```

Do not assume every Kubernetes version exposes identical resource semantics.

---

# Phase 5 — Inspect Namespace Resource Policy

## Step 12 — Inspect LimitRange

Run:

```bash
kubectl get limitrange -n <namespace>
```

If present:

```bash
kubectl describe limitrange -n <namespace>
```

Record:

```text
LimitRange present:
Observed defaults:
Observed minimums:
Observed maximums:
Observed request/limit constraints:
```

Production rule:

```text
SUBMITTED CONFIGURATION
    ≠ NECESSARILY RENDERED CONFIGURATION
```

Do not conclude that the current `LimitRange` produced an older Pod's configuration without timeline evidence.

```text
CURRENT LimitRange
    ≠ PROOF OF HISTORICAL POD DEFAULTING
```

---

## Step 13 — Inspect ResourceQuota

Run:

```bash
kubectl get resourcequota -n <namespace>
```

If present:

```bash
kubectl describe resourcequota -n <namespace>
```

Record:

```text
ResourceQuota present:
CPU quota:
Memory quota:
Current usage:
Other relevant quota:
```

Production rule:

```text
QUOTA FAILURE ≠ SCHEDULER CAPACITY FAILURE
```

A quota/admission rejection and scheduler resource exhaustion require different investigations.

---

# Phase 6 — Establish Scheduler Evidence

## Step 14 — Inspect Pod Scheduling State

Run:

```bash
kubectl get pod <pod> -n <namespace> -o wide
```

and:

```bash
kubectl describe pod <pod> -n <namespace>
```

Record:

```text
Pod phase:
Node assigned:
PodScheduled condition:
Scheduling-related message:
```

Do not treat `Pending` as a root cause.

```text
PENDING IS A STATE, NOT A ROOT CAUSE
```

---

## Step 15 — Inspect Namespace Events

Run:

```bash
kubectl get events -n <namespace> --sort-by=.lastTimestamp
```

Look for evidence such as:

```text
FailedScheduling
Evicted
Preempted
resource-related admission errors
```

Record:

```text
Relevant event:
Reason:
Message:
First observed:
Last observed:
```

Do not infer missing historical events if retention has expired.

---

# Phase 7 — Validate Node Resource Evidence

## Step 16 — Confirm the Assigned Node

Run:

```bash
kubectl get pod <pod> -n <namespace> -o jsonpath="{.spec.nodeName}{'\n'}"
```

Record:

```text
Assigned node:
```

If the Pod is unscheduled, record:

```text
Assigned node: NONE
```

Do not invent a target node for an unscheduled Pod.

---

## Step 17 — Inspect Node Capacity and Allocatable

For an assigned or otherwise evidence-supported node, run:

```bash
kubectl describe node <node>
```

Record:

```text
CPU Capacity:
CPU Allocatable:
Memory Capacity:
Memory Allocatable:
```

Production rule:

```text
NODE CAPACITY ≠ NODE ALLOCATABLE
```

---

## Step 18 — Inspect Allocated Requests and Limits

From:

```bash
kubectl describe node <node>
```

review the allocated resource section.

Record:

```text
CPU requests:
CPU limits:
Memory requests:
Memory limits:
```

Do not compare scheduler fit only against current utilization.

---

## Step 19 — Record Node Conditions

From the same node evidence, record:

```text
MemoryPressure:
DiskPressure:
PIDPressure:
Ready:
```

Capture timestamps and messages where relevant.

---

# Phase 8 — Runtime Resource Evidence

## Step 20 — Inspect Pod Runtime Usage

If metrics are available:

```bash
kubectl top pod <pod> -n <namespace> --containers
```

Record:

```text
Container:
Current CPU:
Current memory:
Observation timestamp:
```

Production rule:

```text
kubectl top ≠ SCHEDULER CAPACITY MODEL
```

and:

```text
LOW kubectl top USAGE ≠ POD CAN SCHEDULE
```

Runtime consumption and scheduler accounting are different evidence classes.

---

## Step 21 — Inspect Node Runtime Usage

If metrics are available:

```bash
kubectl top node <node>
```

Record:

```text
Node CPU usage:
Node CPU percentage:
Node memory usage:
Node memory percentage:
Observation timestamp:
```

Do not use low runtime utilization as proof of scheduler headroom.

---

# Phase 9 — CPU Investigation

## Step 22 — Separate Scheduling CPU From Runtime CPU

Record:

```text
Pod CPU request:
Scheduler CPU-fit evidence:
Current CPU consumption:
CPU limit:
CPU throttling evidence:
```

Classify:

```text
Scheduling CPU issue:
PROVEN / SUPPORTED / UNKNOWN / NOT OBSERVED

Runtime CPU pressure:
PROVEN / SUPPORTED / UNKNOWN / NOT OBSERVED

CPU throttling:
PROVEN / SUPPORTED / UNKNOWN / NOT OBSERVED
```

Production rules:

```text
INSUFFICIENT CPU SCHEDULING CAPACITY
    ≠ CPU THROTTLING

CPU LIMIT PRESENT
    ≠ CPU THROTTLING PROVEN

HIGH CPU
    ≠ CPU THROTTLING PROVEN
```

`kubectl top` alone does not prove CPU throttling.

---

# Phase 10 — Memory Investigation

## Step 23 — Inspect Container State and Restarts

Run:

```bash
kubectl get pod <pod> -n <namespace> -o yaml
```

and:

```bash
kubectl describe pod <pod> -n <namespace>
```

For each relevant container, record:

```text
Container:
Restart count:
Current state:
Last state:
Termination reason:
Exit code:
Started at:
Finished at:
```

---

## Step 24 — Inspect Previous Logs

If the container restarted:

```bash
kubectl logs <pod> -n <namespace> -c <container> --previous
```

Record:

```text
Previous logs available: YES / NO
Relevant evidence:
Timestamp:
```

Production rule:

```text
CURRENT CONTAINER HEALTHY
    ≠ PREVIOUS CONTAINER WAS HEALTHY
```

---

## Step 25 — Classify the Memory Failure

Use the collected evidence to classify:

```text
Container OOM:
PROVEN / SUPPORTED / UNKNOWN / NOT OBSERVED

Node-pressure eviction:
PROVEN / SUPPORTED / UNKNOWN / NOT OBSERVED

Node-level memory pressure:
PROVEN / SUPPORTED / UNKNOWN / NOT OBSERVED

Application-controlled termination:
PROVEN / SUPPORTED / UNKNOWN / NOT OBSERVED

Memory leak:
PROVEN / SUPPORTED / UNKNOWN / ASSUMED
```

Production rules:

```text
OOMKilled ≠ MEMORY LEAK PROVEN

CONTAINER OOM
    ≠ NODE-PRESSURE EVICTION
```

Do not use `OOMKilled` alone as proof of an application memory leak.

---

# Phase 11 — Eviction and QoS Analysis

## Step 26 — Check for Eviction Evidence

Review:

```bash
kubectl describe pod <pod> -n <namespace>
```

and:

```bash
kubectl get events -n <namespace> --sort-by=.lastTimestamp
```

Record:

```text
Eviction observed:
Eviction reason:
Pressured resource:
Relevant node condition:
```

---

## Step 27 — Record Priority Evidence

Inspect:

```bash
kubectl get pod <pod> -n <namespace> -o yaml
```

Record:

```text
priorityClassName:
priority:
```

Production rule:

```text
QoS CLASS ≠ PriorityClass
```

Do not treat QoS as workload priority.

---

## Step 28 — Classify Eviction Evidence

Capture:

```text
QoS class:
Priority:
Resource requests:
Runtime usage:
Pressured resource:
Node conditions:
Eviction reason:
Timestamp:
```

Then classify the conclusion:

```text
QoS contributed to eviction analysis:
PROVEN / SUPPORTED / UNKNOWN

QoS alone proves root cause:
NO
```

Production rules:

```text
QoS CLASS ≠ COMPLETE EVICTION ORDER

EVICTED ≠ QoS ALONE PROVES ROOT CAUSE
```

---

# Phase 12 — Resource Fragmentation and Eligibility

## Step 29 — Avoid Aggregate-Capacity Conclusions

If investigating an unschedulable workload, do not add free CPU or memory across nodes and assume the Pod can schedule.

Production rule:

```text
AGGREGATE FREE CAPACITY ≠ POD FIT
```

A Pod must fit on an individual eligible node.

Record:

```text
Pod applicable CPU requirement:
Pod applicable memory requirement:
Eligible node identified:
Resource fit proven:
```

---

## Step 30 — Separate Eligibility From Capacity

Record whether node eligibility is affected by observed scheduling constraints.

Do not modify those constraints in this lab.

Use:

```text
NODE ELIGIBILITY
        ↓
RESOURCE FIT
        ↓
SCHEDULER DECISION
```

Production rule:

```text
NODE HAS RESOURCES
    ≠ NODE IS ELIGIBLE FOR THIS POD
```

Detailed selector, affinity, taint, toleration, and dedicated-pool investigation is covered in later Part 3 tutorials.

---

# Phase 13 — Establish Blast Radius

## Step 31 — Compare Nearby Workloads

Using read-only evidence, determine whether the symptom is limited to:

```text
one container
one Pod
one workload
one namespace
one node
one node pool
multiple workloads
cluster-wide
```

Record:

```text
Minimum Supported Blast Radius:
Evidence:
```

Do not describe a cluster-wide capacity incident when evidence supports only a single-container failure.

---

# Phase 14 — Build the Timeline

## Step 32 — Correlate Evidence

Build an incident timeline:

```text
T0 Configuration/deployment change:
T1 Pod admitted:
T2 Pod scheduled:
T3 Load changed:
T4 Resource behavior changed:
T5 Throttling/OOM/pressure evidence:
T6 Application impact:
T7 Restart/eviction:
T8 Recovery:
```

Record unknown timestamps explicitly as:

```text
UNKNOWN
```

Production rule:

```text
CORRELATION ≠ CAUSATION
```

---

# Phase 15 — Determine Failure Boundary

## Step 33 — Identify the Lowest Proven Healthy Layer

Use the lifecycle:

```text
Desired configuration
        ↓
Admission/defaulting
        ↓
Rendered Pod resources
        ↓
QoS classification
        ↓
Scheduler accounting
        ↓
Node placement
        ↓
Runtime enforcement
        ↓
Node-pressure/eviction behavior
```

Record:

```text
Lowest Proven Healthy Layer:
Evidence:
```

---

## Step 34 — Identify the First Failed Transition

Record:

```text
First Failed Transition:
Evidence:
```

Examples of evidence-supported transitions might include:

```text
Admission → rejected by quota

Scheduler accounting → no eligible node satisfies resource fit

Runtime enforcement → throttling evidence correlated with impact

Container runtime → OOM termination observed

Node health → MemoryPressure followed by eviction
```

Do not select a transition that is merely assumed.

---

# Phase 16 — Evidence Confidence

## Step 35 — Classify Important Conclusions

Use:

```text
PROVEN
SUPPORTED
UNKNOWN
ASSUMED
```

Record:

```text
Conclusion:
Evidence:
Confidence:
```

Production rule:

```text
ASSUMED ≠ ROOT CAUSE
```

---

# Phase 17 — Identify the Current Actionable Owner

## Step 36 — Determine Who Owns the Next Action

Possible ownership domains can include:

```text
Application / model workload
TrueFoundry / platform configuration
GitOps / deployment configuration
Kubernetes namespace policy
Kubernetes scheduling
Node / node-pool capacity
Runtime / operating system
Observability
```

Record:

```text
Current Actionable Owner:
Evidence supporting ownership:
Requested Action:
```

Production rule:

```text
SYMPTOM ≠ FAILURE DOMAIN ≠ OWNER
```

---

# Phase 18 — Validate Desired-State Ownership

## Step 37 — Identify the Authoritative Configuration Source

Determine whether resource configuration is controlled by:

```text
TrueFoundry
GitOps
Helm
Terraform
another deployment controller
UNKNOWN
```

Record:

```text
Authoritative desired-state owner:
```

Do not change the live Kubernetes workload during this lab.

```text
DO NOT PATCH THE SYMPTOM
WHEN ANOTHER SYSTEM OWNS DESIRED STATE
```

---

# Phase 19 — Evidence Handoff Contract

## Step 38 — Complete the Handoff

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

Separate observations from interpretation.

---

# Phase 20 — Final Safety Validation

## Step 39 — Confirm No Mutation Was Performed

Confirm:

```text
Workload changed: NO
Requests changed: NO
Limits changed: NO
Replicas changed: NO
Priority changed: NO
Pod deleted/restarted: NO
Node labels changed: NO
Node taints changed: NO
Affinity/selectors changed: NO
Node-pool configuration changed: NO
```

If any mutation occurred outside this lab, record it in the incident timeline because it may affect evidence interpretation.

---

# Acceptance Criteria

The lab is complete when you can answer:

- Which workload, Pod, UID, node, and container were investigated?
- What was the incident timestamp?
- What system owns authoritative desired state?
- What CPU and memory resources were desired?
- What CPU and memory resources were rendered?
- What QoS class did Kubernetes assign?
- Are Pod-level resources or Pod overhead relevant?
- Did `LimitRange` or `ResourceQuota` affect the investigation?
- What scheduler evidence exists?
- Which nodes were eligible?
- Was resource fit proven?
- What did runtime metrics show?
- Is CPU throttling proven, supported, unknown, or not observed?
- Was there an OOM, eviction, node-pressure event, or application-controlled termination?
- Was previous-container evidence captured?
- What is the Minimum Supported Blast Radius?
- What is the Lowest Proven Healthy Layer?
- What is the First Failed Transition?
- What is the evidence confidence?
- Who is the Current Actionable Owner?
- What evidence-supported action is required next?
- Was the investigation completed without mutating production state?

---

# Final Production Rules

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

---

## Lab Result

If all acceptance criteria are satisfied, record:

```text
Part 3.2 Requests, Limits & Kubernetes QoS
[SAFE-READ] Production Validation Lab: PASS
```

Do not perform remediation as part of this validation lab.
