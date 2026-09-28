# TrueFoundry Part 3 --- Compute & Scheduling

## Purpose

Part 3 builds on the Kubernetes integration foundation established in
Part 2 and focuses on how CPU and memory requirements influence
Kubernetes workload placement and runtime behavior.

The goal is to teach how TrueFoundry-managed workloads translate compute
intent into Kubernetes scheduling constraints, how the scheduler
evaluates placement, and how an SRE distinguishes resource configuration
problems from cluster-capacity and scheduling failures.

## Audience

This part is intended for:

-   Platform Engineers
-   Site Reliability Engineers (SREs)
-   DevOps Engineers
-   Kubernetes Administrators
-   AI/ML Platform Engineers
-   Infrastructure Engineers supporting TrueFoundry

## Prerequisites

Readers should understand the concepts established in Parts 1 and 2,
especially:

-   TrueFoundry Control Plane vs Compute Plane
-   workload identity and target cluster/namespace
-   desired, rendered, and observed state
-   Kubernetes Pods and workload controllers
-   ServiceAccounts and authorization boundaries
-   Kubernetes events
-   Lowest Proven Healthy Layer
-   First Failed Transition
-   Minimum Supported Blast Radius
-   Current Actionable Owner
-   `UNKNOWN TARGET = NO MUTATION`
-   `[SAFE-READ]` production validation

Basic Kubernetes scheduling familiarity is helpful but not required.

## Part 3 Learning Path

  -----------------------------------------------------------------------
  \#                      Tutorial                Production Focus
  ----------------------- ----------------------- -----------------------
  3.1                     CPU & Memory Resource   Resource units,
                          Management              workload demand, node
                                                  allocatable capacity,
                                                  pressure, and runtime
                                                  impact

  3.2                     Requests, Limits &      Scheduling requests,
                          Kubernetes QoS          runtime limits, QoS
                                                  classes, throttling,
                                                  OOM behavior, and
                                                  eviction context

  3.3                     Node Selectors & Node   Label-based placement,
                          Affinity                required/preferred
                                                  affinity, eligibility,
                                                  and placement failures

  3.4                     Taints & Tolerations    Repelling workloads,
                                                  toleration semantics,
                                                  placement constraints,
                                                  and operational
                                                  validation

  3.5                     Dedicated Node Pools    Isolation, node-pool
                                                  identity, capacity
                                                  boundaries, workload
                                                  targeting, and
                                                  operational tradeoffs

  3.6                     Workload Scheduling     Evidence-driven
                          Troubleshooting         diagnosis of Pending
                                                  and unschedulable
                                                  workloads across the
                                                  complete scheduling
                                                  path
  -----------------------------------------------------------------------

## Repository Structure

``` text
part-03-compute-scheduling/
├── README.md
├── PART-03-LAYOUT.md
├── 01-cpu-memory-resource-management/
│   └── 01-cpu-memory-resource-management.md
├── 02-requests-limits-kubernetes-qos/
│   └── 02-requests-limits-kubernetes-qos.md
├── 03-node-selectors-node-affinity/
│   └── 03-node-selectors-node-affinity.md
├── 04-taints-tolerations/
│   └── 04-taints-tolerations.md
├── 05-dedicated-node-pools/
│   └── 05-dedicated-node-pools.md
├── 06-workload-scheduling-troubleshooting/
│   └── 06-workload-scheduling-troubleshooting.md
└── labs/
    ├── 01-cpu-memory-resource-management-check.md
    ├── 02-requests-limits-kubernetes-qos-check.md
    ├── 03-node-selectors-node-affinity-check.md
    ├── 04-taints-tolerations-check.md
    ├── 05-dedicated-node-pools-check.md
    └── 06-workload-scheduling-troubleshooting-check.md
```

Tutorial directories are created as part of the Part 3 skeleton.
Canonical tutorial and lab files are added only when their workflow
reaches the appropriate stage.

## Review Workflow

Every tutorial follows the same stage-gated workflow:

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
[SAFE-READ] Hands-On Lab
  ↓
Repository Validation
```

Only one stage is advanced at a time.

Technical/vendor validation should use current official TrueFoundry
documentation and current upstream Kubernetes documentation where
required.

## Production/SRE Principles

Part 3 carries forward the operational rules established in Parts 1 and
2:

``` text
UNKNOWN TARGET = NO MUTATION
```

``` text
REQUESTED RESOURCES ≠ ACTUAL RESOURCE USAGE
```

``` text
NODE CAPACITY ≠ NODE ALLOCATABLE
```

``` text
TOTAL CLUSTER CAPACITY ≠ SCHEDULABLE CAPACITY FOR THIS POD
```

``` text
POD PENDING ≠ INSUFFICIENT CPU OR MEMORY
```

``` text
SCHEDULING FAILURE ≠ RUNTIME RESOURCE FAILURE
```

Troubleshooting should establish:

``` text
Workload Identity
  ↓
Rendered Pod Requirements
  ↓
Eligible Nodes
  ↓
Scheduler Constraints
  ↓
Available Allocatable Resources
  ↓
Scheduling Decision
  ↓
Runtime Resource Behavior
```

and then:

``` text
Evidence
  ↓
Lowest Proven Healthy Layer
  ↓
First Failed Transition
  ↓
Failure Domain
  ↓
Next Required Action
  ↓
Current Actionable Owner
```

## Hands-On Lab Policy

Production-oriented validation labs are marked:

``` text
[SAFE-READ]
```

Unless a lab explicitly states otherwise, production validation uses
observational commands such as:

``` bash
kubectl config current-context
kubectl get
kubectl describe
kubectl logs
kubectl top
```

`kubectl top` is used only when the cluster has a functioning metrics
pipeline and its output is treated as observed usage rather than
scheduling capacity.

Read-only JSONPath queries may be used where needed.

Production validation labs must not instruct the reader to perform
unapproved mutations such as:

``` text
kubectl create
kubectl apply
kubectl delete
kubectl edit
kubectl patch
kubectl scale
kubectl rollout restart
kubectl taint
kubectl label
```

Labs must not change workload resources, node labels, taints,
tolerations, affinity, or node-pool configuration in production.

Mutation exercises required to demonstrate scheduling behavior must be
clearly separated from `[SAFE-READ]` production validation and
identified as non-production or approved-change exercises.

## Scope Boundary

Part 3 covers general CPU/memory compute and Kubernetes scheduling
behavior.

GPU-specific topics are intentionally deferred to Part 4, including:

-   NVIDIA drivers
-   NVIDIA device plugin
-   GPU Operator
-   CUDA runtime
-   GPU resource requests
-   GPU node pools
-   GPU device visibility
-   GPU scheduling troubleshooting

Application deployment mechanics remain in Part 5, and
autoscaling/performance tuning remains in Part 8.

## Completion Criteria

Part 3 is complete only when all six tutorials have completed:

-   Draft
-   Technical/Vendor Validation
-   Production/SRE Review
-   Revised Final
-   Canonical
-   Hands-On Lab
-   Repository Validation

`truefoundry/STATUS-TRACKER.md` is updated once at the end of Part 3
after all six tutorials and labs have passed repository validation.

## Outcome

After completing Part 3, the reader should be able to explain how
Kubernetes evaluates CPU/memory resource requests and placement
constraints, determine why a workload can or cannot schedule onto a
node, distinguish scheduler failures from runtime resource failures, and
collect production-safe evidence before routing remediation to the
authoritative owner.
