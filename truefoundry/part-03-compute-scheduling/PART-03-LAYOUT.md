# Part 3 --- Compute & Scheduling --- Master Layout

## Part Objective

Part 3 provides a production-focused understanding of CPU/memory
resource management and Kubernetes scheduling for TrueFoundry-managed
workloads.

The part follows a workload from declared resource intent through
scheduler eligibility and node placement to runtime resource behavior.
The objective is to prevent common incident-analysis mistakes such as
treating every Pending Pod as a capacity problem or treating current CPU
usage as proof of schedulable capacity.

## Scope Boundary

Part 3 covers:

-   Kubernetes CPU and memory resource units
-   requests and limits
-   node capacity and allocatable resources
-   scheduler resource accounting
-   Kubernetes QoS
-   node labels
-   node selectors
-   node affinity
-   taints and tolerations
-   dedicated node pools
-   workload placement
-   Pending/unschedulable workload investigation
-   CPU throttling, OOM, and resource-pressure context where needed to
    distinguish scheduling from runtime failures

The following subjects are intentionally deferred:

-   GPU infrastructure and GPU scheduling --- Part 4
-   application deployment mechanics --- Part 5
-   model-serving resource configuration --- Parts 6--7
-   autoscaling and performance tuning --- Part 8
-   advanced networking --- Part 9
-   broader security architecture --- Part 10
-   observability architecture --- Part 11
-   production capacity planning --- Part 12
-   broader incident scenarios and runbooks --- Parts 13--14

This boundary avoids duplication while preserving the evidence needed
for production scheduling investigations.

------------------------------------------------------------------------

## 3.1 --- CPU & Memory Resource Management

### Objective

Explain Kubernetes CPU and memory resource semantics and establish the
production evidence model used to distinguish configured resources, node
capacity, allocatable resources, and observed workload consumption.

### Core Topics

-   CPU units and millicores
-   memory units
-   resource requests
-   resource limits
-   node capacity
-   node allocatable
-   scheduler resource accounting
-   observed resource usage
-   resource pressure
-   workload-controller vs Pod resource configuration
-   init containers and sidecars where they affect effective resource
    requirements
-   resource evidence and timestamps

### SRE Focus

Maintain the distinctions:

``` text
NODE CAPACITY ≠ NODE ALLOCATABLE
```

``` text
REQUESTED RESOURCES ≠ ACTUAL RESOURCE USAGE
```

``` text
CURRENT LOW USAGE ≠ SCHEDULABLE CAPACITY
```

Validate the workload's rendered resource configuration before
concluding that the cluster is undersized.

### Lab

`labs/01-cpu-memory-resource-management-check.md`

The production validation portion must be `[SAFE-READ]`.

------------------------------------------------------------------------

## 3.2 --- Requests, Limits & Kubernetes QoS

### Objective

Explain how resource requests and limits affect scheduling and runtime
behavior, and how Kubernetes QoS classification contributes to
resource-pressure behavior.

### Core Topics

-   CPU requests
-   CPU limits
-   memory requests
-   memory limits
-   scheduler use of requests
-   CPU throttling
-   memory limit enforcement
-   OOM behavior
-   QoS classes
-   Guaranteed
-   Burstable
-   BestEffort
-   resource-pressure and eviction context
-   effective Pod resource requirements

### SRE Focus

Differentiate:

``` text
SCHEDULING REQUEST
        ≠
RUNTIME USAGE
        ≠
RUNTIME LIMIT
```

and:

``` text
POD SCHEDULED ≠ POD HAS SUFFICIENT RUNTIME HEADROOM
```

A workload may schedule successfully and later experience throttling or
memory failure. Those are different transitions and must be investigated
separately.

### Lab

`labs/02-requests-limits-kubernetes-qos-check.md`

------------------------------------------------------------------------

## 3.3 --- Node Selectors & Node Affinity

### Objective

Explain label-based node placement and how required and preferred
scheduling constraints influence the set of nodes eligible for a
workload.

### Core Topics

-   node labels
-   `nodeSelector`
-   node affinity
-   required node affinity
-   preferred node affinity
-   scheduler eligibility
-   multiple placement constraints
-   label drift
-   workload-controller ownership
-   topology considerations
-   placement evidence

### SRE Focus

Use the model:

``` text
NODE EXISTS
    ≠
NODE ELIGIBLE FOR THIS POD
```

and:

``` text
MATCHING LABEL EXISTS SOMEWHERE
    ≠
ELIGIBLE NODE HAS REQUIRED CAPACITY
```

Do not patch labels or affinity during production diagnosis simply to
make a Pod schedule.

### Lab

`labs/03-node-selectors-node-affinity-check.md`

------------------------------------------------------------------------

## 3.4 --- Taints & Tolerations

### Objective

Explain how taints repel Pods, how tolerations permit scheduling
consideration, and how taints interact with other scheduler constraints.

### Core Topics

-   taints
-   tolerations
-   `NoSchedule`
-   `PreferNoSchedule`
-   `NoExecute`
-   toleration matching
-   existing vs new Pods
-   placement eligibility
-   node isolation
-   interaction with node affinity/selectors
-   scheduler events

### SRE Focus

Maintain:

``` text
TOLERATION EXISTS ≠ POD MUST SCHEDULE TO THAT NODE
```

``` text
TAINT BLOCKS POD ≠ TAINT IS ROOT CAUSE OF EVERY PENDING POD
```

Tolerations remove a repelling condition; they do not guarantee
placement or provide capacity.

### Lab

`labs/04-taints-tolerations-check.md`

------------------------------------------------------------------------

## 3.5 --- Dedicated Node Pools

### Objective

Explain dedicated node pools as an operational placement and isolation
pattern without treating any one cloud provider's implementation as
universal TrueFoundry behavior.

### Core Topics

-   node-pool identity
-   labels
-   taints
-   tolerations
-   affinity/selectors
-   workload isolation
-   capacity boundaries
-   scaling boundaries
-   maintenance implications
-   failure domains
-   cost considerations
-   cloud-provider implementation differences

### SRE Focus

Establish the complete placement contract:

``` text
WORKLOAD REQUIREMENTS
        ↓
NODE-POOL LABELS / TAINTS
        ↓
ELIGIBLE NODES
        ↓
ALLOCATABLE CAPACITY
        ↓
SCHEDULING
```

A healthy node pool is not proof that the affected Pod is eligible to
use it.

### Lab

`labs/05-dedicated-node-pools-check.md`

------------------------------------------------------------------------

## 3.6 --- Workload Scheduling Troubleshooting

### Objective

Combine the Part 3 concepts into an evidence-driven production method
for diagnosing Pending and unschedulable workloads.

### Core Topics

-   Pod scheduling lifecycle
-   scheduler events
-   Pod conditions
-   unschedulable reasons
-   requests vs allocatable capacity
-   node readiness
-   cordon/unschedulable state
-   selectors and affinity
-   taints and tolerations
-   topology constraints
-   PVC/storage scheduling dependencies where relevant
-   preemption context
-   autoscaler signals where relevant
-   desired-state ownership
-   blast radius
-   remediation ownership

### SRE Focus

Use the investigation chain:

``` text
Verify Workload Identity
        ↓
Confirm Pod Exists
        ↓
Confirm Scheduling State
        ↓
Read Scheduler Evidence
        ↓
Determine Eligible Node Set
        ↓
Evaluate Placement Constraints
        ↓
Evaluate Allocatable Capacity
        ↓
Identify First Failed Transition
        ↓
Identify Authoritative Owner
```

Do not infer the reason for Pending state from the phase alone.

### Lab

`labs/06-workload-scheduling-troubleshooting-check.md`

------------------------------------------------------------------------

# Standard Tutorial Workflow

Every Part 3 tutorial follows:

``` text
Draft
  ↓
Technical/Vendor Validation
  ↓
Production/SRE Review
  ↓
Revised Final
  ↓
Canonical
  ↓
Hands-On Lab
  ↓
Repository Validation
```

## Draft

Create the full teaching flow and operational model.

## Technical/Vendor Validation

Validate version-sensitive TrueFoundry statements against current
official TrueFoundry documentation.

Validate Kubernetes scheduling and resource behavior against current
official Kubernetes documentation.

Avoid presenting tutorial-created SRE methods as Kubernetes or
TrueFoundry product terminology.

## Production/SRE Review

Review for:

-   production safety
-   scheduling accuracy
-   resource-accounting accuracy
-   failure-domain clarity
-   evidence quality
-   blast-radius analysis
-   ownership boundaries
-   authoritative remediation
-   recovery validation

## Revised Final

Incorporate technical and SRE review findings.

## Canonical

Use the explicit canonical filename defined by the Part 3 naming
convention.

Do not replace tutorial filenames with generic `README.md`.

## Hands-On Lab

Create the matching lab under `labs/`.

Production validation labs are `[SAFE-READ]` unless explicitly
identified as an approved mutation exercise.

## Repository Validation

Validate:

-   expected file paths
-   naming convention
-   no accidental files
-   tutorial/lab pairing
-   links and navigation where applicable
-   Git diff
-   staged scope
-   clean commit
-   push to `main`

------------------------------------------------------------------------

# Naming Convention

## Tutorial

``` text
NN-topic-name/NN-topic-name.md
```

Example:

``` text
01-cpu-memory-resource-management/
└── 01-cpu-memory-resource-management.md
```

## Lab

``` text
labs/NN-topic-name-check.md
```

Example:

``` text
labs/01-cpu-memory-resource-management-check.md
```

------------------------------------------------------------------------

# Production Safety Standard

For production investigation:

``` text
OBSERVE BEFORE MUTATE
```

and:

``` text
UNKNOWN TARGET = NO MUTATION
```

Safe evidence normally includes:

``` text
kubectl config current-context
kubectl get
kubectl describe
kubectl logs
kubectl top
```

where the exact command is read-only, does not expose sensitive data,
and the metrics pipeline is available when `kubectl top` is used.

Mutating commands must not be included in `[SAFE-READ]` production
validation.

In particular, production labs must not instruct unapproved changes to:

-   requests or limits
-   replicas
-   node labels
-   node taints
-   Pod tolerations
-   node selectors
-   node affinity
-   node-pool size or configuration

------------------------------------------------------------------------

# Part 3 Operational Model

The recurring compute and scheduling path is:

``` text
TrueFoundry Workload Configuration
          ↓
Rendered Kubernetes Workload
          ↓
Pod Resource Requests + Placement Constraints
          ↓
Scheduler
          ↓
Eligible Node Set
          ↓
Node Allocatable Capacity
          ↓
Scheduling Decision
          ↓
Node / Container Runtime
          ↓
Runtime Resource Behavior
          ↓
Application Health
```

For an incident:

``` text
Verify Deployment Identity
        ↓
Determine Minimum Supported Blast Radius
        ↓
Collect Timestamped Scheduler Evidence
        ↓
Confirm Rendered Resource Requirements
        ↓
Determine Eligible Nodes
        ↓
Compare Constraints and Allocatable Capacity
        ↓
Find Lowest Proven Healthy Layer
        ↓
Find First Failed Transition
        ↓
Classify Failure Domain
        ↓
Define Next Required Action
        ↓
Identify Current Actionable Owner
        ↓
Remediate Through Authoritative Source
        ↓
Validate End-to-End Recovery
```

------------------------------------------------------------------------

# Part 3 Completion Gate

Part 3 is complete when tutorials 3.1 through 3.6 each have:

``` text
Draft                         ✅
Technical/Vendor Validation  ✅
Production/SRE Review         ✅
Revised Final                 ✅
Canonical                     ✅
Hands-On Lab                  ✅
Repository Validation         ✅
```

After 3.6 repository validation:

1.  update `truefoundry/STATUS-TRACKER.md` once for the completed Part
    3;
2.  validate the final Part 3 repository diff;
3.  commit the completed tracker state;
4.  push and verify `main`;
5.  proceed to Part 4 only after Part 3 is closed.
