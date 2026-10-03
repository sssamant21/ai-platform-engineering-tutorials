# Part 3.4 --- Taints & Tolerations --- Hands-On Lab

**Lab Type:** `[SAFE-READ]` Production Validation Lab\
**Track:** TrueFoundry --- Compute & Scheduling\
**Part:** 3.4\
**Audience:** Platform Engineers, SREs, DevOps Engineers, Kubernetes
Operators

------------------------------------------------------------------------

## 1. Lab Objective

Use read-only Kubernetes evidence to determine whether taints and
tolerations contribute to a workload scheduling or eviction problem.

By the end of the lab, you should be able to:

-   establish exact workload identity;
-   preserve scheduler and Pod evidence;
-   inspect rendered Pod tolerations;
-   inspect node taints and conditions;
-   compare exact taints with exact tolerations;
-   distinguish `NoSchedule`, `PreferNoSchedule`, and `NoExecute`;
-   distinguish placement incompatibility from resource-capacity
    problems;
-   investigate `NoExecute` behavior using a timeline;
-   recognize automatically added tolerations and workload-owner
    context;
-   determine the Minimum Supported Blast Radius;
-   identify the Lowest Proven Healthy Layer and First Failed
    Transition;
-   classify evidence as `PROVEN`, `SUPPORTED`, `UNKNOWN`, or `ASSUMED`;
-   identify the authoritative desired-state owner and Current
    Actionable Owner.

------------------------------------------------------------------------

## 2. Safety Boundary

This is a strict `[SAFE-READ]` lab.

Do **not** use this lab to modify Pods, workloads, nodes, node pools,
taints, tolerations, selectors, affinity, resources, replicas, or
scheduler configuration.

Do not run:

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

Do not remove a taint merely because it appears in scheduler evidence.

Do not add a toleration merely to make a Pending Pod schedule.

Do not modify a node-condition taint to hide an underlying node-health
problem.

Production rule:

``` text
UNKNOWN WORKLOAD IDENTITY = NO TAINT / TOLERATION CHANGE
```

------------------------------------------------------------------------

## 3. Required Inputs

Before starting, identify:

``` text
Environment:
Cluster:
Namespace:
Workload:
Pod:
Approximate incident time:
Observed symptom:
```

Set local shell placeholders mentally or replace placeholders in each
command carefully.

This lab intentionally does not use mutation commands to create test
resources.

------------------------------------------------------------------------

# Phase 1 --- Establish Cluster and Workload Identity

## 4. Confirm Current Context

``` bash
kubectl config current-context
```

Record:

``` text
Cluster / Context:
Environment:
Timestamp:
```

Do not continue if the context is not the intended cluster.

------------------------------------------------------------------------

## 5. Confirm Namespace and Pod

``` bash
kubectl get pod <pod> -n <namespace> -o wide
```

Capture:

``` text
Pod:
Namespace:
Status:
Node:
Pod IP:
Age:
```

Then capture the Pod UID:

``` bash
kubectl get pod <pod> -n <namespace> -o jsonpath='{.metadata.uid}'
```

Record:

``` text
Pod UID:
```

The Pod UID matters because a Pod with the same name can be recreated.

------------------------------------------------------------------------

## 6. Identify the Workload Owner

``` bash
kubectl get pod <pod> -n <namespace> -o jsonpath='{.metadata.ownerReferences}'
```

If more context is required:

``` bash
kubectl get pod <pod> -n <namespace> -o yaml
```

Record:

``` text
Owner Kind:
Owner Name:
```

Production rule:

``` text
POD ≠ AUTHORITATIVE DESIRED-STATE OWNER
```

------------------------------------------------------------------------

# Phase 2 --- Capture Scheduler and Lifecycle Evidence

## 7. Describe the Pod

``` bash
kubectl describe pod <pod> -n <namespace>
```

Capture:

-   Pod status;
-   Node assignment;
-   Conditions;
-   Events;
-   scheduler messages;
-   eviction-related messages;
-   restart/lifecycle information where present.

Record the exact scheduler Event rather than paraphrasing it.

Ask:

``` text
WHAT DOES THE SCHEDULER PROVE?
```

Production rule:

``` text
PENDING ≠ UNTOLERATED TAINT PROVEN
```

------------------------------------------------------------------------

## 8. Review Namespace Events

``` bash
kubectl get events -n <namespace> --sort-by=.lastTimestamp
```

Use this only as supporting evidence because Events can age out.

Record relevant:

``` text
Timestamp:
Reason:
Object:
Message:
```

Do not treat missing historical Events as proof that an event never
occurred.

------------------------------------------------------------------------

# Phase 3 --- Inspect Rendered Placement Configuration

## 9. Capture Rendered nodeSelector

``` bash
kubectl get pod <pod> -n <namespace> -o jsonpath='{.spec.nodeSelector}'
```

Record:

``` text
Rendered nodeSelector:
```

------------------------------------------------------------------------

## 10. Capture Rendered Node Affinity

``` bash
kubectl get pod <pod> -n <namespace> -o jsonpath='{.spec.affinity.nodeAffinity}'
```

Record:

``` text
Required node affinity:
Preferred node affinity:
```

Part 3.3 established:

``` text
NODE MATCHES AFFINITY ≠ NODE ACCEPTS THE POD
```

A node can satisfy affinity and still be excluded by an untolerated
taint.

------------------------------------------------------------------------

## 11. Capture Rendered Tolerations

``` bash
kubectl get pod <pod> -n <namespace> -o jsonpath='{.spec.tolerations}'
```

For easier manual review, use:

``` bash
kubectl get pod <pod> -n <namespace> -o yaml
```

Record each relevant toleration:

``` text
Key:
Operator:
Value:
Effect:
tolerationSeconds:
```

Production rule:

``` text
TOLERATION EXISTS ≠ RELEVANT TAINT IS TOLERATED
```

------------------------------------------------------------------------

## 12. Capture Scheduler and Direct Node Assignment

``` bash
kubectl get pod <pod> -n <namespace> -o jsonpath='{.spec.schedulerName}'
```

Then:

``` bash
kubectl get pod <pod> -n <namespace> -o jsonpath='{.spec.nodeName}'
```

Record:

``` text
schedulerName:
spec.nodeName:
```

If `spec.nodeName` was explicitly assigned before normal scheduling,
normal scheduler node selection can be bypassed.

Therefore:

``` text
POD ON NoSchedule-TAINTED NODE ≠ SCHEDULER VIOLATION PROVEN
```

------------------------------------------------------------------------

# Phase 4 --- Inspect Node State

## 13. List Nodes

``` bash
kubectl get nodes -o wide
```

Record candidate nodes relevant to the workload.

Do not assume every cluster node is eligible.

------------------------------------------------------------------------

## 14. Inspect Node Labels

For a candidate node:

``` bash
kubectl get node <node> --show-labels
```

For structured output:

``` bash
kubectl get node <node> -o yaml
```

Record labels relevant to:

``` text
nodeSelector
required nodeAffinity
node pool
zone / topology
specialized compute
```

------------------------------------------------------------------------

## 15. Inspect Node Taints

``` bash
kubectl describe node <node>
```

Look for:

``` text
Taints:
```

You can also inspect:

``` bash
kubectl get node <node> -o jsonpath='{.spec.taints}'
```

Record every relevant taint:

``` text
Node:
Key:
Value:
Effect:
```

Do not stop after finding the first taint.

Production rule:

``` text
ONE TAINT TOLERATED ≠ ALL RELEVANT TAINTS TOLERATED
```

------------------------------------------------------------------------

## 16. Inspect Node Conditions

``` bash
kubectl get node <node> -o jsonpath='{.status.conditions}'
```

For easier interpretation:

``` bash
kubectl describe node <node>
```

Inspect conditions related to:

``` text
Ready
MemoryPressure
DiskPressure
PIDPressure
NetworkUnavailable
```

Record:

``` text
Condition:
Status:
Reason:
Message:
Last transition:
```

Production rule:

``` text
TAINT EXISTS ≠ MANUAL CONFIGURATION ERROR
```

A taint can reflect node-health behavior rather than intentional
workload placement.

------------------------------------------------------------------------

## 17. Inspect Node Schedulability

``` bash
kubectl get node <node> -o jsonpath='{.spec.unschedulable}'
```

Also review:

``` bash
kubectl describe node <node>
```

Do not confuse a node being unschedulable with a taint/toleration
mismatch.

------------------------------------------------------------------------

# Phase 5 --- Compare Taints and Tolerations

## 18. Build a Match Table

For each candidate node, manually build:

``` text
NODE:
------------------------------------------------
Taint 1:
  key:
  value:
  effect:

Matching Pod toleration:
  key:
  operator:
  value:
  effect:
  tolerationSeconds:

Result:
  MATCH / NO MATCH / UNKNOWN
------------------------------------------------
```

Repeat for every relevant taint.

The correct question is:

``` text
DOES A RENDERED POD TOLERATION
MATCH THIS SPECIFIC NODE TAINT?
```

Not:

``` text
Does the Pod have tolerations?
```

------------------------------------------------------------------------

## 19. Evaluate Remaining Taints

Conceptually:

``` text
ALL NODE TAINTS
       ↓
Compare against Pod tolerations
       ↓
Matched taints
       ↓
Remaining unmatched taints
       ↓
Evaluate effects
```

Classify remaining effects:

``` text
NoSchedule
PreferNoSchedule
NoExecute
```

Remember:

``` text
PreferNoSchedule ≠ NoSchedule
NoSchedule ≠ NoExecute
```

------------------------------------------------------------------------

# Phase 6 --- Build the Eligible-Node Funnel

## 20. Count Nodes Through Each Stage

Build:

``` text
ALL NODES:
    count =

Ready / schedulable:
    count =

nodeSelector-compatible:
    count =

required-affinity-compatible:
    count =

taint/toleration-compatible:
    count =

other hard constraints:
    count =

resource-fit:
    count =

FINAL ELIGIBLE NODES:
    count =
```

The objective is to identify the first stage where the candidate set
becomes zero or unexpectedly small.

------------------------------------------------------------------------

## 21. Distinguish Taint Failure From Capacity Failure

Example:

``` text
10 total nodes
4 satisfy required placement
0 survive taint/toleration evaluation
```

This supports investigation of taint/toleration compatibility.

Compare:

``` text
10 total nodes
4 satisfy required placement
4 survive taint/toleration evaluation
0 satisfy resource requirements
```

This supports resource-fit/capacity investigation.

Production rule:

``` text
UNTOLERATED TAINT ≠ INSUFFICIENT CAPACITY
```

------------------------------------------------------------------------

# Phase 7 --- Check Resource Fit Without Confusing Runtime Usage

## 22. Inspect Pod Requests

``` bash
kubectl get pod <pod> -n <namespace> -o yaml
```

Review resource requests for all relevant containers and init
containers.

Do not use `kubectl top` as the scheduler's resource-fit model.

From Part 3.1 and Part 3.2:

``` text
REQUEST ≠ USAGE
kubectl top ≠ SCHEDULER CAPACITY MODEL
```

------------------------------------------------------------------------

## 23. Inspect Node Allocatable and Allocated Resources

``` bash
kubectl describe node <node>
```

Review:

``` text
Capacity
Allocatable
Allocated resources
```

Use this only after placement and taint compatibility have been
evaluated.

``` text
NODE HAS RESOURCES ≠ NODE IS ELIGIBLE
```

------------------------------------------------------------------------

# Phase 8 --- NoExecute Investigation

## 24. Determine Whether NoExecute Is Involved

From node taints, identify whether a relevant taint uses:

``` text
NoExecute
```

Record:

``` text
NoExecute involved: YES / NO / UNKNOWN
```

If no, continue to the next phase.

------------------------------------------------------------------------

## 25. Check the Matching NoExecute Toleration

Inspect the rendered Pod tolerations again:

``` bash
kubectl get pod <pod> -n <namespace> -o yaml
```

Record:

``` text
Matching toleration:
tolerationSeconds:
```

Classify:

``` text
No matching toleration
Matching toleration with no tolerationSeconds
Matching toleration with tolerationSeconds
UNKNOWN
```

------------------------------------------------------------------------

## 26. Build the NoExecute Timeline

Record:

``` text
Pod creation:
Pod scheduling:
Pod became Running:
Node condition transition:
Taint observed:
NoExecute observed:
Eviction evidence:
Pod termination:
Replacement Pod:
Recovery:
```

Production rule:

``` text
CURRENT TAINT + CURRENT TOLERATION ≠ COMPLETE EVICTION HISTORY
CURRENT STATE ≠ STATE AT INCIDENT TIME
```

Do not infer historical behavior solely from current node state.

------------------------------------------------------------------------

# Phase 9 --- Automatic Toleration and Workload-Type Checks

## 27. Check for not-ready / unreachable Tolerations

Inspect:

``` bash
kubectl get pod <pod> -n <namespace> -o yaml
```

Look for tolerations involving:

``` text
node.kubernetes.io/not-ready
node.kubernetes.io/unreachable
```

If present, do not automatically attribute them to application
configuration.

Production rule:

``` text
TOLERATION PRESENT ≠ USER EXPLICITLY CONFIGURED IT
```

------------------------------------------------------------------------

## 28. Check Whether the Pod Is Owned by a DaemonSet

Use the owner evidence collected earlier.

If the owner is a DaemonSet:

``` text
DAEMONSET TOLERATION BEHAVIOR ≠ ORDINARY APPLICATION POD BEHAVIOR
```

Record:

``` text
Workload type:
Automatic/controller-related toleration behavior relevant: YES / NO / UNKNOWN
```

------------------------------------------------------------------------

# Phase 10 --- Node-Condition Root-Cause Separation

## 29. Determine Whether the Taint Is Condition-Related

Compare node taints with node conditions.

Classify the taint:

``` text
Dedicated-placement taint
Condition-related taint
Platform-managed taint
Unknown
```

If condition-related, record the underlying node evidence.

Example:

``` text
Taint:
node.kubernetes.io/memory-pressure

Relevant condition:
MemoryPressure=True

Underlying cause:
UNKNOWN until node resource evidence is investigated
```

Production rule:

``` text
TAINT ≠ UNDERLYING NODE ROOT CAUSE
```

And:

``` text
REMOVE SCHEDULING SYMPTOM ≠ REMEDIATE NODE FAILURE
```

------------------------------------------------------------------------

# Phase 11 --- Dedicated Node Pool Validation

## 30. Validate the Intended Placement Pattern

For a dedicated pool, record:

``` text
Expected node label:
Expected node affinity / selector:
Expected node taint:
Expected Pod toleration:
```

Then compare with rendered/live state.

Expected pattern:

``` text
LABEL / AFFINITY
      ↓
TARGET INTENDED NODE POOL

TAINT
      ↓
REPEL INCOMPATIBLE PODS

TOLERATION
      ↓
ALLOW INTENDED POD THROUGH THAT TAINT
```

Do not treat a toleration as an attraction mechanism.

------------------------------------------------------------------------

## 31. Check for Overly Broad Tolerations

Review whether the Pod tolerates more nodes than intended.

Ask:

``` text
Could this toleration make the workload eligible
for specialized or expensive node pools unexpectedly?
```

Record:

``` text
Overly broad toleration risk:
PROVEN / SUPPORTED / UNKNOWN / NOT OBSERVED
```

Production rule:

``` text
OVERLY BROAD TOLERATION CAN CAUSE UNINTENDED ELIGIBILITY
```

------------------------------------------------------------------------

# Phase 12 --- Desired-State Ownership

## 32. Determine Where the Placement Policy Comes From

Identify whether the authoritative configuration is controlled by:

``` text
TrueFoundry
GitOps
Helm
Kustomize
Terraform
Kubernetes operator
Admission policy
Managed Kubernetes behavior
Node provisioning system
Node-pool automation
Kubernetes control plane
Other
Unknown
```

Record:

``` text
Authoritative Desired-State Owner:
Evidence:
```

Production rules:

``` text
POD ≠ AUTHORITATIVE DESIRED-STATE OWNER
OBSERVED DRIFT ≠ PERMISSION TO PATCH
```

and:

``` text
DO NOT PATCH THE SYMPTOM
WHEN ANOTHER SYSTEM OWNS DESIRED STATE
```

------------------------------------------------------------------------

# Phase 13 --- Determine Blast Radius

## 33. Check Related Pods

Read-only examples:

``` bash
kubectl get pods -n <namespace> -o wide
```

If appropriate and authorized for the investigation:

``` bash
kubectl get pods -A -o wide
```

Do not infer cluster-wide impact from one Pod.

Classify:

``` text
One Pod
One workload
One namespace
One node
One node pool
Multiple workloads
Cluster-wide
UNKNOWN
```

Production rule:

``` text
OBSERVED BLAST RADIUS ≠ ASSUMED PLATFORM-WIDE IMPACT
```

------------------------------------------------------------------------

# Phase 14 --- Lowest Proven Healthy Layer

## 34. Identify the First Failed Transition

Use evidence to construct a chain such as:

``` text
Kubernetes API                    HEALTHY
       ↓
Nodes registered                  HEALTHY
       ↓
Nodes Ready                       HEALTHY
       ↓
Pod accepted                      HEALTHY
       ↓
Scheduler evaluating Pod          HEALTHY
       ↓
Required affinity candidates      HEALTHY
       ↓
Taint/toleration compatibility    FAILED
```

Record:

``` text
Lowest Proven Healthy Layer:

First Failed Transition:
```

Do not use a generic statement such as:

``` text
Root cause: Kubernetes scheduling issue
```

when a more precise boundary is available.

------------------------------------------------------------------------

# Phase 15 --- Evidence Confidence

## 35. Classify Every Important Finding

Use:

``` text
PROVEN
SUPPORTED
UNKNOWN
ASSUMED
```

Example:

``` text
PROVEN
Scheduler reports an untolerated taint.

PROVEN
Candidate nodes currently carry the taint.

PROVEN
Rendered Pod lacks a matching toleration.

SUPPORTED
A workload change correlates with incident start.

UNKNOWN
Why desired configuration omitted the toleration.

ASSUMED
TrueFoundry removed the toleration.
```

Production rule:

``` text
ASSUMED ≠ ROOT CAUSE
```

------------------------------------------------------------------------

# Phase 16 --- Current Actionable Owner

## 36. Separate Failure Domain From Owner

Determine:

``` text
Observed symptom:
First Failed Transition:
Failure domain:
Authoritative Desired-State Owner:
Current Actionable Owner:
```

Examples of possible actionable owners include:

``` text
Application/workload team
TrueFoundry/platform team
Kubernetes platform team
Cloud/infrastructure team
Node-pool automation owner
GitOps/IaC owner
Unknown pending additional evidence
```

Production rule:

``` text
SYMPTOM ≠ FAILURE DOMAIN ≠ OWNER
```

An error surfaced in TrueFoundry does not prove TrueFoundry is the
failure domain.

------------------------------------------------------------------------

# Phase 17 --- Evidence Handoff Contract

## 37. Complete the Incident Handoff

``` text
Environment:
Cluster:
Namespace:
Workload:
Workload Owner:
Pod:
Pod UID:
Node:
Timestamp:

Symptom:
Blast Radius:

Scheduler Events:

Rendered nodeSelector:
Rendered Required Affinity:
Rendered Tolerations:
Scheduler Name:
spec.nodeName:

Candidate Nodes:

Relevant Node Labels:
Relevant Node Taints:
Relevant Node Conditions:

Taint/Toleration Match Result:

Eligible Nodes After Placement:
Eligible Nodes After Taints:
Resource Fit:

NoExecute Involved:
tolerationSeconds:
Eviction Evidence:

Timeline:

Lowest Proven Healthy Layer:
First Failed Transition:

Evidence Confidence:

Authoritative Desired-State Owner:
Current Actionable Owner:

Requested Action:
```

The `Requested Action` should identify what the receiving owner needs to
investigate or change. It should not prescribe an unverified mutation.

------------------------------------------------------------------------

# Phase 18 --- Final Safety Validation

## 38. Before Recommending Any Change

Confirm:

``` text
[ ] Correct cluster/context verified
[ ] Correct namespace verified
[ ] Pod UID captured
[ ] Workload owner identified
[ ] Scheduler evidence preserved
[ ] Rendered tolerations captured
[ ] Candidate node taints captured
[ ] Node conditions captured
[ ] Every relevant taint evaluated
[ ] NoSchedule vs NoExecute distinguished
[ ] Resource fit separated from taint compatibility
[ ] Timeline reconstructed where required
[ ] Automatic tolerations considered
[ ] DaemonSet behavior considered where applicable
[ ] Blast radius established
[ ] Lowest Proven Healthy Layer identified
[ ] First Failed Transition identified
[ ] Evidence confidence assigned
[ ] Desired-state owner identified
[ ] Current Actionable Owner identified
```

Only after this evidence exists should a separate approved remediation
procedure be considered.

------------------------------------------------------------------------

## 39. Lab Completion Criteria

The lab is complete when you can answer all of the following with
evidence:

1.  Which exact workload and Pod were investigated?
2.  What did the scheduler or lifecycle evidence report?
3.  Which nodes were placement-compatible?
4.  Which taints existed on those nodes?
5.  Which rendered tolerations existed on the Pod?
6.  Which taints matched or did not match?
7.  Was `NoSchedule`, `PreferNoSchedule`, or `NoExecute` relevant?
8.  Did the candidate set become zero because of taints or because of
    resource fit?
9.  Was a node-condition taint involved?
10. Was an automatically added toleration relevant?
11. Was the workload a DaemonSet or another controller type?
12. Was `spec.nodeName` relevant?
13. What was the Minimum Supported Blast Radius?
14. What was the Lowest Proven Healthy Layer?
15. What was the First Failed Transition?
16. Which findings were `PROVEN`, `SUPPORTED`, `UNKNOWN`, or `ASSUMED`?
17. Who owns the authoritative desired state?
18. Who is the Current Actionable Owner?
19. What evidence should be handed to that owner?
20. What requested action follows from the evidence?

------------------------------------------------------------------------

## 40. Production Rules Reinforced by This Lab

``` text
UNKNOWN WORKLOAD IDENTITY = NO TAINT / TOLERATION CHANGE
NODE MATCHES AFFINITY ≠ NODE ACCEPTS THE POD
TOLERATION ≠ PLACEMENT REQUIREMENT
POD TOLERATES NODE ≠ POD MUST RUN ON NODE
TOLERATION EXISTS ≠ RELEVANT TAINT IS TOLERATED
ONE TAINT TOLERATED ≠ ALL RELEVANT TAINTS TOLERATED
TOLERATION MATCH ≠ COMPLETE NODE ELIGIBILITY
PreferNoSchedule ≠ NoSchedule
NoSchedule ≠ NoExecute
PENDING ≠ UNTOLERATED TAINT PROVEN
POD NEVER SCHEDULED ≠ POD SCHEDULED THEN EVICTED
UNTOLERATED TAINT ≠ INSUFFICIENT CAPACITY
TOLERATION PRESENT ≠ USER EXPLICITLY CONFIGURED IT
DAEMONSET TOLERATION BEHAVIOR ≠ ORDINARY APPLICATION POD BEHAVIOR
TAINT EXISTS ≠ MANUAL CONFIGURATION ERROR
TAINT ≠ UNDERLYING NODE ROOT CAUSE
REMOVE SCHEDULING SYMPTOM ≠ REMEDIATE NODE FAILURE
CURRENT STATE ≠ STATE AT INCIDENT TIME
POD EVICTED ≠ RESOURCE PRESSURE PROVEN
POD DISAPPEARED ≠ NoExecute PROVEN
OVERLY BROAD TOLERATION CAN CAUSE UNINTENDED ELIGIBILITY
TOLERATION ≠ AUTHORIZATION
TOLERATION ≠ SECURITY BOUNDARY
POD ON NoSchedule-TAINTED NODE ≠ SCHEDULER VIOLATION PROVEN
TAINT CHANGE ≠ LOCAL POD FIX
REQUEST ≠ USAGE
kubectl top ≠ SCHEDULER CAPACITY MODEL
NODE HAS RESOURCES ≠ NODE IS ELIGIBLE
POD ≠ AUTHORITATIVE DESIRED-STATE OWNER
OBSERVED DRIFT ≠ PERMISSION TO PATCH
ERROR VISIBLE THROUGH TRUEFOUNDRY ≠ TRUEFOUNDRY ROOT CAUSE
REMEDIATION CAN DESTROY INCIDENT EVIDENCE
ASSUMED ≠ ROOT CAUSE
SYMPTOM ≠ FAILURE DOMAIN ≠ OWNER
```

------------------------------------------------------------------------

## 41. Final Operational Principle

The purpose of taint/toleration troubleshooting is not to find the
fastest command that makes a Pod schedule.

The objective is to determine:

``` text
WHAT WAS THE FIRST FAILED TRANSITION?
WHY DID THE NODE BECOME INELIGIBLE OR THE POD BECOME EVICTABLE?
WHO OWNS THE AUTHORITATIVE STATE?
WHAT EVIDENCE SUPPORTS THE REMEDIATION?
```

Preserve evidence first. Change production state only through an
approved remediation process after the failure domain and ownership are
established.
