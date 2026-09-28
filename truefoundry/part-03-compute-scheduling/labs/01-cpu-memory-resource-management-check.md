# Part 3.1 Lab --- CPU & Memory Resource Management Check

> **\[SAFE-READ\] Production Validation Lab**

## Purpose

This lab validates CPU and memory configuration, scheduling evidence,
node capacity, runtime consumption, and resource-related failure
evidence for an existing Kubernetes workload.

The lab is observational. It must not change the workload, Pod, node,
requests, limits, replicas, labels, taints, affinity, or node-pool
configuration.

The central question is:

> **What CPU and memory did the affected workload require, what did
> Kubernetes render and account for, which node received the Pod, what
> resources were allocatable there, what is being consumed, and where is
> the first proven resource-related failure?**

## Safety Rules

This lab permits read-only investigation such as:

``` text
kubectl config current-context
kubectl get
kubectl describe
kubectl logs
kubectl top
```

Do not use this lab to run:

``` text
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

Do not modify requests, limits, replicas, node labels, taints,
tolerations, affinity, or node-pool configuration.

``` text
UNKNOWN TARGET = NO MUTATION
UNKNOWN WORKLOAD IDENTITY = NO RESOURCE CHANGE
OBSERVE BEFORE MUTATE
```

## Variables

Set these from the actual incident or validation target:

``` text
NAMESPACE=<namespace>
POD=<pod>
CONTAINER=<container>
NODE=<node-observed-from-pod>
```

Do not guess these values.

## Check 1 --- Verify Kubernetes Context

``` bash
kubectl config current-context
```

Record:

``` text
Expected environment:
Expected cluster:
Observed context:
Match: YES / NO
Timestamp:
```

Stop if the context is not the intended target.

## Check 2 --- Verify Namespace and Pod Identity

``` bash
kubectl get pod -n <namespace>
```

Then inspect the target Pod:

``` bash
kubectl get pod <pod> -n <namespace> -o wide
```

Record:

``` text
Namespace:
Pod:
Status:
Node:
Pod IP:
Age:
```

Obtain stable Pod identity metadata:

``` bash
kubectl get pod <pod> -n <namespace> \
  -o jsonpath='{.metadata.uid}{"\n"}{.metadata.creationTimestamp}{"\n"}'
```

Record:

``` text
Pod UID:
Creation timestamp:
```

Rule:

``` text
POD NAME ≠ COMPLETE POD IDENTITY
```

## Check 3 --- Identify the Owning Workload

``` bash
kubectl get pod <pod> -n <namespace> \
  -o jsonpath='{range .metadata.ownerReferences[*]}{.kind}{"\t"}{.name}{"\t"}{.uid}{"\n"}{end}'
```

Record the immediate owner.

If the immediate owner is a ReplicaSet, inspect its owner reference:

``` bash
kubectl get replicaset <replicaset> -n <namespace> \
  -o jsonpath='{range .metadata.ownerReferences[*]}{.kind}{"\t"}{.name}{"\t"}{.uid}{"\n"}{end}'
```

Do not assume the Pod itself is the authoritative resource
configuration.

``` text
POD ≠ AUTHORITATIVE RESOURCE CONFIGURATION
```

## Check 4 --- Inventory Pod Containers

List regular containers:

``` bash
kubectl get pod <pod> -n <namespace> \
  -o jsonpath='{range .spec.containers[*]}{.name}{"\n"}{end}'
```

List init containers:

``` bash
kubectl get pod <pod> -n <namespace> \
  -o jsonpath='{range .spec.initContainers[*]}{.name}{"\n"}{end}'
```

Record:

``` text
Application container(s):
Sidecar/other regular container(s):
Init container(s):
```

Do not assume the application container represents the complete Pod
resource requirement.

``` text
APPLICATION CONTAINER REQUEST ≠ EFFECTIVE POD REQUIREMENT
```

## Check 5 --- Inspect Rendered Container Requests and Limits

``` bash
kubectl get pod <pod> -n <namespace> \
  -o jsonpath='{range .spec.containers[*]}{"CONTAINER="}{.name}{"\nREQUESTS="}{.resources.requests}{"\nLIMITS="}{.resources.limits}{"\n\n"}{end}'
```

Inspect init-container resources:

``` bash
kubectl get pod <pod> -n <namespace> \
  -o jsonpath='{range .spec.initContainers[*]}{"INIT="}{.name}{"\nREQUESTS="}{.resources.requests}{"\nLIMITS="}{.resources.limits}{"\n\n"}{end}'
```

Record for every relevant container:

``` text
Container:
CPU request:
CPU limit:
Memory request:
Memory limit:
```

For init containers:

``` text
Init container:
CPU request:
CPU limit:
Memory request:
Memory limit:
```

Do not manually derive the effective Pod requirement using an
oversimplified formula if the cluster uses resource features whose
semantics have not been verified.

## Check 6 --- Check Pod-Level Resources When Applicable

Inspect whether the Pod contains Pod-level resource configuration:

``` bash
kubectl get pod <pod> -n <namespace> \
  -o jsonpath='{.spec.resources}{"\n"}'
```

If the field is empty, record:

``` text
Pod-level resources observed: NO / NOT PRESENT
```

If populated, record the metadata-level resource quantities without
changing them.

Also record:

``` bash
kubectl version
```

The effective resource calculation must follow the behavior of the
actual Kubernetes version and enabled feature state.

``` text
CLUSTER VERSION / FEATURE STATE MATTERS
```

## Check 7 --- Inspect Pod Overhead

``` bash
kubectl get pod <pod> -n <namespace> \
  -o jsonpath='{.spec.overhead}{"\n"}'
```

Record:

``` text
Pod overhead present: YES / NO
CPU overhead:
Memory overhead:
```

Do not assume overhead is present.

## Check 8 --- Determine Scheduling State

``` bash
kubectl get pod <pod> -n <namespace> \
  -o jsonpath='{.status.phase}{"\n"}'
```

Inspect Pod conditions:

``` bash
kubectl get pod <pod> -n <namespace> \
  -o jsonpath='{range .status.conditions[*]}{.type}{"\t"}{.status}{"\t"}{.reason}{"\t"}{.message}{"\n"}{end}'
```

Record:

``` text
Phase:
PodScheduled:
Reason:
Message:
```

Rule:

``` text
PENDING IS A STATE, NOT A ROOT CAUSE
```

## Check 9 --- Inspect Scheduler and Pod Events

``` bash
kubectl describe pod <pod> -n <namespace>
```

Focus on:

``` text
Events
FailedScheduling
Scheduled
Preemption
Insufficient cpu
Insufficient memory
taint-related messages
affinity/selector messages
storage-related messages
```

Record exact observed scheduler evidence:

``` text
Scheduler event timestamp:
Reason:
Message:
```

Do not classify a Pending Pod as a CPU or memory shortage unless
evidence supports it.

``` text
POD PENDING ≠ INSUFFICIENT CPU/MEMORY PROVEN
```

## Check 10 --- Confirm Node Assignment

``` bash
kubectl get pod <pod> -n <namespace> \
  -o jsonpath='{.spec.nodeName}{"\n"}'
```

If the Pod is already scheduled, use the returned node for subsequent
node checks.

If `.spec.nodeName` is empty, do not invent a node target. Use scheduler
evidence and later Part 3 placement methods to evaluate eligible nodes.

``` text
UNKNOWN NODE TARGET = NO NODE-SPECIFIC CONCLUSION
```

## Check 11 --- Inspect Node Capacity and Allocatable

For a scheduled Pod:

``` bash
kubectl get node <node> \
  -o jsonpath='{.status.capacity.cpu}{"\t"}{.status.capacity.memory}{"\n"}{.status.allocatable.cpu}{"\t"}{.status.allocatable.memory}{"\n"}'
```

Record:

``` text
Node:
CPU capacity:
CPU allocatable:
Memory capacity:
Memory allocatable:
```

Rule:

``` text
NODE CAPACITY ≠ NODE ALLOCATABLE
```

## Check 12 --- Inspect Node Conditions

``` bash
kubectl get node <node> \
  -o jsonpath='{range .status.conditions[*]}{.type}{"\t"}{.status}{"\t"}{.reason}{"\t"}{.lastTransitionTime}{"\n"}{end}'
```

Record at minimum:

``` text
Ready:
MemoryPressure:
DiskPressure:
PIDPressure:
```

Also inspect whether Kubernetes marks the node unschedulable:

``` bash
kubectl get node <node> \
  -o jsonpath='{.spec.unschedulable}{"\n"}'
```

Do not equate node pressure with a container-limit failure.

``` text
NODE MemoryPressure ≠ CONTAINER MEMORY LIMIT EXCEEDED
```

## Check 13 --- Inspect Allocated Requests and Limits

``` bash
kubectl describe node <node>
```

Review the `Allocated resources` section.

Record:

``` text
CPU requests:
CPU limits:
Memory requests:
Memory limits:
```

Do not compare only current utilization with node allocatable resources
when investigating scheduler capacity.

``` text
LOW UTILIZATION ≠ LOW RESOURCE RESERVATION
```

## Check 14 --- Observe Runtime Usage

If the metrics pipeline is available:

``` bash
kubectl top pod <pod> -n <namespace> --containers
```

and:

``` bash
kubectl top node <node>
```

Record:

``` text
Container CPU observed:
Container memory observed:
Node CPU observed:
Node memory observed:
Timestamp:
```

If metrics are unavailable, record:

``` text
Runtime metrics: UNKNOWN / UNAVAILABLE
```

Do not convert missing metrics into an assumption.

``` text
kubectl top ≠ SCHEDULER CAPACITY MODEL
```

## Check 15 --- Inspect Container Runtime State

``` bash
kubectl get pod <pod> -n <namespace> \
  -o jsonpath='{range .status.containerStatuses[*]}{"CONTAINER="}{.name}{"\nREADY="}{.ready}{"\nRESTARTS="}{.restartCount}{"\nCURRENT="}{.state}{"\nLAST="}{.lastState}{"\n\n"}{end}'
```

Record:

``` text
Container:
Ready:
Restart count:
Current state:
Last state:
Termination reason:
Exit code:
Finished time:
```

Rule:

``` text
CURRENT CONTAINER HEALTH ≠ PREVIOUS FAILURE EVIDENCE
```

## Check 16 --- Inspect Previous Logs When a Restart Occurred

Only when a previous container instance exists and logs are relevant:

``` bash
kubectl logs <pod> -n <namespace> -c <container> --previous
```

Avoid broad or unnecessary log collection. Do not expose credentials or
sensitive application data in incident tickets.

Record only relevant evidence:

``` text
Previous log evidence:
Timestamp:
Error/signature:
```

## Check 17 --- Classify Memory Evidence

If memory-related termination is suspected, compare:

``` text
Container last termination reason
Memory request
Memory limit
Observed memory
Node MemoryPressure
Eviction events
Restart timeline
```

Possible classification:

``` text
PROVEN container/cgroup OOM
SUPPORTED container memory issue
PROVEN node memory pressure
PROVEN node-pressure eviction
UNKNOWN
```

Do not write:

``` text
OOMKilled = memory leak
```

Use:

``` text
OOMKilled ≠ MEMORY LEAK PROVEN
```

## Check 18 --- Separate Scheduling CPU from Runtime CPU

If the Pod could not schedule, focus on:

``` text
CPU request
Effective Pod requirement
Eligible nodes
Node allocatable
Existing request accounting
Scheduler evidence
```

If the Pod scheduled but application performance is poor, focus on:

``` text
Observed CPU
CPU limit
Throttling evidence from the available observability stack
Node contention
Application behavior
Traffic/concurrency
```

Do not combine these into one diagnosis.

``` text
INSUFFICIENT CPU FOR SCHEDULING ≠ CPU THROTTLING
```

## Check 19 --- Determine Minimum Supported Blast Radius

Compare the affected target with peer resources.

Examples:

``` bash
kubectl get pod -n <namespace> -o wide
```

and, where the owning workload is known:

``` bash
kubectl get deployment <deployment> -n <namespace>
```

or the applicable workload controller.

Classify only the smallest scope supported by evidence:

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

Record:

``` text
Minimum Supported Blast Radius:
Evidence:
```

## Check 20 --- Build the Incident Timeline

Record:

``` text
Incident start:
Pod creation:
Scheduling event:
Container start:
Restart:
OOM/termination:
Node-pressure transition:
Deployment/configuration change:
Scaling event:
Traffic change:
```

Use:

``` text
TIMING CORRELATION ≠ ROOT CAUSE PROOF
```

## Check 21 --- Identify the Lowest Proven Healthy Layer

Use the resource path:

``` text
Desired Configuration
        ↓
Rendered Kubernetes Resources
        ↓
Effective Pod Requirement
        ↓
Scheduler Evaluation
        ↓
Eligible Node Capacity
        ↓
Placement
        ↓
Runtime Enforcement
        ↓
Observed Consumption
        ↓
Application Behavior
```

Record:

``` text
Lowest Proven Healthy Layer:
Evidence:
Timestamp:
```

## Check 22 --- Identify the First Failed Transition

Examples:

``` text
Rendered Resources → Scheduler Evaluation
Scheduler Evaluation → Eligible Node Capacity
Placement → Runtime Enforcement
Runtime Enforcement → Application Behavior
```

Record:

``` text
First Failed Transition:
Evidence:
Timestamp:
```

Do not skip directly from symptom to root cause.

## Check 23 --- Identify Authoritative Configuration Ownership

From the observed Pod, trace:

``` text
Pod
→ Owning Controller
→ Deployment/Workload Definition
→ TrueFoundry / GitOps / Deployment Source
→ Authoritative Configuration
```

Record:

``` text
Observed Kubernetes object:
Owning controller:
Authoritative configuration source:
Authoritative owner:
```

If the authoritative source is unknown, record:

``` text
UNKNOWN
```

Do not patch the observed Pod as a diagnostic shortcut.

``` text
Observed Drift ≠ Permission to Patch
```

## Check 24 --- Assign Evidence Confidence

For every important conclusion, use:

``` text
PROVEN
SUPPORTED
UNKNOWN
ASSUMED
```

Example:

``` text
Claim:
Evidence:
Confidence:
```

Never promote:

``` text
ASSUMED → ROOT CAUSE
```

## Check 25 --- Complete the Evidence Handoff

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

## Check 26 --- Validate No Mutation Was Performed

Before closing the lab, confirm:

``` text
Requests changed: NO
Limits changed: NO
Replicas changed: NO
Pod deleted/restarted: NO
Node labels changed: NO
Node taints changed: NO
Affinity changed: NO
Node-pool configuration changed: NO
Secrets exposed: NO
```

If any state-changing action was required during a real incident, it
belongs to a separately approved remediation procedure, not this
`[SAFE-READ]` validation lab.

## Acceptance Criteria

The lab passes when the engineer can demonstrate:

``` text
[ ] Correct cluster/context proven
[ ] Namespace and Pod identity proven
[ ] Pod UID recorded
[ ] Owning workload identified
[ ] All relevant containers inventoried
[ ] Rendered requests and limits captured
[ ] Pod-level resources checked when applicable
[ ] Pod overhead checked
[ ] Scheduling state captured
[ ] Scheduler evidence captured
[ ] Node assignment proven when scheduled
[ ] Node capacity and allocatable captured
[ ] Node pressure conditions captured
[ ] Allocated request/limit evidence reviewed
[ ] Runtime metrics captured or explicitly marked unavailable
[ ] Restart/termination evidence reviewed
[ ] Previous logs reviewed when applicable
[ ] Scheduling and runtime CPU paths separated
[ ] Container OOM and node-pressure paths separated
[ ] Minimum Supported Blast Radius established
[ ] Incident timeline established
[ ] Lowest Proven Healthy Layer identified
[ ] First Failed Transition identified
[ ] Authoritative configuration owner identified or marked UNKNOWN
[ ] Evidence confidence assigned
[ ] Current Actionable Owner identified
[ ] No production mutation performed
```

## Final Lab Principle

``` text
DO NOT ASK ONLY:

"How much CPU and memory is being used?"

PROVE:

WHAT WAS REQUESTED
→ WHAT WAS RENDERED
→ WHAT THE SCHEDULER ACCOUNTED FOR
→ WHICH NODES WERE ELIGIBLE
→ WHAT WAS ALLOCATABLE
→ WHAT WAS CONSUMED
→ WHERE THE FIRST FAILURE OCCURRED
→ WHO OWNS THE AUTHORITATIVE REMEDIATION
```

And always retain:

``` text
REQUEST ≠ USAGE
NODE CAPACITY ≠ NODE ALLOCATABLE
kubectl top ≠ SCHEDULER CAPACITY MODEL
PENDING IS A STATE, NOT A ROOT CAUSE
OOMKilled ≠ MEMORY LEAK PROVEN
OBSERVE BEFORE MUTATE
ASSUMED ≠ ROOT CAUSE
SYMPTOM ≠ FAILURE DOMAIN ≠ OWNER
```
