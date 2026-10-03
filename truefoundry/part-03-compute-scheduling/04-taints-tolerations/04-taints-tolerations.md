# Part 3.4 --- Taints & Tolerations

**Track:** TrueFoundry --- Compute & Scheduling\
**Part:** 3.4\
**Audience:** Platform Engineers, SREs, DevOps Engineers, Kubernetes
Operators\
**Focus:** Node taints, Pod tolerations, scheduling compatibility,
eviction behavior, and production troubleshooting

------------------------------------------------------------------------

## 1. Purpose

Kubernetes taints and tolerations control whether Pods are compatible
with nodes that carry specific taints.

``` text
NODE SELECTOR / NODE AFFINITY → ATTRACT OR REQUIRE PLACEMENT
TAINT                       → REPEL INCOMPATIBLE PODS
TOLERATION                  → ALLOW A POD THROUGH THAT TAINT
```

``` text
TOLERATION ≠ PLACEMENT REQUIREMENT
POD TOLERATES NODE ≠ POD MUST RUN ON NODE
```

## 2. Production Scheduling Model

``` text
ALL NODES
    ↓
Ready / schedulable candidates
    ↓
nodeSelector
    ↓
required nodeAffinity
    ↓
NODE TAINTS
    ↓
POD TOLERATIONS
    ↓
Remaining unmatched taints/effects
    ↓
Storage / topology / other hard constraints
    ↓
Resource fit
    ↓
ELIGIBLE NODES
    ↓
Scheduler scoring
    ↓
SELECTED NODE
```

``` text
NODE MATCHES AFFINITY ≠ NODE ACCEPTS THE POD
TOLERATION MATCH ≠ COMPLETE NODE ELIGIBILITY
```

## 3. What Is a Taint?

A taint is associated with a Kubernetes node and is represented
conceptually as:

``` text
key=value:effect
```

Example:

``` text
dedicated=ai:NoSchedule
```

A taint contains `key`, `value`, and `effect`.

Read-only inspection:

``` bash
kubectl describe node <node>
kubectl get node <node> -o yaml
```

Inspect `spec.taints`. During production investigation, record the exact
key, value, and effect.

## 4. What Is a Toleration?

A toleration is configured on a Pod.

``` yaml
tolerations:
  - key: dedicated
    operator: Equal
    value: ai
    effect: NoSchedule
```

If the toleration matches, that specific taint no longer excludes the
Pod on that basis. Other scheduling requirements still apply.

## 5. Taint Effects

Important node-taint effects are `NoSchedule`, `PreferNoSchedule`, and
`NoExecute`.

### 5.1 NoSchedule

A Pod without an appropriate matching toleration cannot normally be
newly scheduled onto the node through the scheduler.

``` text
Candidate Node
      ↓
NoSchedule taint
      ↓
Matching toleration?
   ┌──────┴──────┐
   NO            YES
   ↓              ↓
Excluded      Continue evaluation
```

A matching toleration only removes this particular scheduling
restriction.

### 5.2 PreferNoSchedule

`PreferNoSchedule` is a soft preference. Kubernetes attempts to avoid
placing a Pod that does not tolerate the taint on that node, but
placement can still occur.

``` text
PreferNoSchedule ≠ NoSchedule
```

### 5.3 NoExecute

`NoExecute` affects placement and can also affect Pods already running
on a node.

``` text
POD NEVER SCHEDULED ≠ POD SCHEDULED THEN EVICTED
```

The second path requires timeline and eviction analysis.

## 6. tolerationSeconds

A matching `NoExecute` toleration can specify how long a Pod tolerates
the taint.

``` yaml
tolerations:
  - key: example
    operator: Equal
    value: maintenance
    effect: NoExecute
    tolerationSeconds: 300
```

If the taint remains after the configured duration, eviction can follow.
A matching `NoExecute` toleration without `tolerationSeconds` can
tolerate the taint indefinitely while the match remains valid.

``` text
CURRENT TAINT + CURRENT TOLERATION ≠ COMPLETE EVICTION HISTORY
```

Always correlate timestamps.

## 7. Toleration Matching

Traditional toleration operators include `Equal` and `Exists`.

``` yaml
- key: dedicated
  operator: Equal
  value: ai
  effect: NoSchedule
```

``` yaml
- key: dedicated
  operator: Exists
  effect: NoSchedule
```

Modern Kubernetes can also provide numeric comparison capabilities such
as `Gt` and `Lt` under version- and feature-dependent toleration
functionality. Verify the actual Kubernetes version and feature state
before relying on them.

For incident investigation, capture:

``` text
key
operator
value
effect
tolerationSeconds
```

``` text
TOLERATION EXISTS ≠ RELEVANT TAINT IS TOLERATED
```

## 8. Multiple Taints

A node can contain several taints.

``` text
dedicated=ai:NoSchedule
environment=production:NoSchedule
maintenance=true:PreferNoSchedule
```

Do not conclude that a node is eligible after finding one matching
toleration.

``` text
ONE TAINT TOLERATED ≠ ALL RELEVANT TAINTS TOLERATED
```

Conceptually evaluate all remaining unmatched taints and their effects.

## 9. Toleration Does Not Attract a Pod

Suppose GPU nodes carry:

``` text
dedicated=gpu:NoSchedule
```

A matching toleration means the taint does not exclude the Pod. It does
not mean "run this Pod on GPU nodes."

Placement intent can come from `nodeSelector`, node affinity, resource
requirements, and other scheduling constraints.

``` text
TOLERATION ≠ PLACEMENT REQUIREMENT
```

## 10. Affinity with Taints and Tolerations

A common dedicated-node pattern combines a node label and taint.

``` text
Label:
node-pool=gpu

Taint:
dedicated=gpu:NoSchedule
```

The intended workload can use required placement plus a matching
toleration.

``` text
AFFINITY              → TARGET / REQUIRE THE POOL
TAINT + TOLERATION    → REPEL OTHERS / ALLOW COMPATIBLE WORKLOAD
```

``` text
TAINT / TOLERATION ≠ SECURITY BOUNDARY
```

## 11. Missing vs Overly Broad Tolerations

A missing toleration can exclude an intended workload. An overly broad
toleration can make a general workload unexpectedly eligible for
specialized or expensive capacity.

``` text
MISSING TOLERATION CAN CAUSE EXCLUSION
OVERLY BROAD TOLERATION CAN CAUSE UNINTENDED ELIGIBILITY
```

## 12. Toleration Is Not Authorization

``` text
TOLERATION ≠ AUTHORIZATION
TOLERATION ≠ SECURITY BOUNDARY
TOLERATION ≠ RESOURCE ENTITLEMENT
```

A Pod can tolerate a GPU-node taint without necessarily requesting or
being able to use a GPU.

## 13. Node-Condition Taints

Some taints originate from Kubernetes node-health conditions rather than
deliberate workload-isolation design. Important examples relate to
NotReady, Unreachable, MemoryPressure, DiskPressure, PIDPressure, and
NetworkUnavailable.

``` text
NODE CONDITION
      ↓
Kubernetes control-plane behavior
      ↓
CONDITION-RELATED TAINT
      ↓
Scheduling / eviction consequences
```

``` text
TAINT EXISTS ≠ MANUAL CONFIGURATION ERROR
TAINT MAY BE A MECHANISM / CONSEQUENCE ≠ UNDERLYING ROOT CAUSE
```

## 14. Do Not Fix Node Health by Hiding the Taint

For condition-related taints, investigate why the node is unhealthy
instead of immediately removing the taint.

Potential remediation domains include memory, disk, node capacity,
container runtime, networking, cloud infrastructure, and node lifecycle.

``` text
REMOVE SCHEDULING SYMPTOM ≠ REMEDIATE NODE FAILURE
```

## 15. Automatically Added Tolerations

Kubernetes can automatically add certain tolerations. A particularly
important operational case involves:

``` text
node.kubernetes.io/not-ready
node.kubernetes.io/unreachable
```

Ordinary Pods can receive default temporary tolerations for these
conditions.

``` text
TOLERATION PRESENT ≠ USER EXPLICITLY CONFIGURED IT
```

During RCA, determine whether a toleration came from desired workload
configuration, platform configuration, admission/defaulting, Kubernetes
behavior, or controller behavior.

## 16. DaemonSet Considerations

DaemonSet Pods can receive automatic tolerations associated with
node-condition behavior.

``` text
DAEMONSET TOLERATION BEHAVIOR ≠ ORDINARY APPLICATION POD BEHAVIOR
```

Inspect `metadata.ownerReferences` before using one workload's
tolerations as a model for another.

## 17. Scheduler Evidence First

For a Pending Pod:

``` bash
kubectl describe pod <pod> -n <namespace>
```

Inspect scheduler Events.

``` text
WHAT DOES THE SCHEDULER PROVE?
PENDING ≠ UNTOLERATED TAINT PROVEN
```

Do not start with an assumption that the taint is wrong.

## 18. Inspect Rendered Pod Tolerations

``` bash
kubectl get pod <pod> -n <namespace> -o yaml
kubectl get pod <pod> -n <namespace> -o jsonpath='{.spec.tolerations}'
```

Inspect `spec.tolerations`.

``` text
DESIRED TOLERATION ≠ RENDERED TOLERATION
```

The rendered Pod is evidence, but it may not be the authoritative
desired-state owner.

## 19. Inspect Actual Node Taints

``` bash
kubectl describe node <node>
kubectl get node <node> -o yaml
```

Inspect `spec.taints` and compare exact node taints with exact Pod
tolerations.

## 20. Production Eligibility Funnel

``` text
ALL NODES
    ↓
Ready / schedulable
    ↓
nodeSelector
    ↓
required nodeAffinity
    ↓
taint / toleration compatibility
    ↓
storage / topology / other constraints
    ↓
resource fit
    ↓
ELIGIBLE NODES
```

Ask:

``` text
HOW MANY NODES SURVIVE EACH FILTER?
```

Distinguish:

``` text
NO PLACEMENT-COMPATIBLE NODE
          ≠
NO TAINT-COMPATIBLE NODE
          ≠
NO RESOURCE-FIT NODE
```

## 21. Taint Failure vs Capacity Failure

If candidate nodes survive affinity but none survive taint/toleration
evaluation, investigate scheduling compatibility.

If nodes survive taint/toleration evaluation but none satisfy resource
requirements, investigate resource fit and capacity.

``` text
UNTOLERATED TAINT ≠ INSUFFICIENT CAPACITY
```

## 22. NoExecute Timeline Investigation

For a previously running Pod, establish:

``` text
Pod created
    ↓
Pod scheduled
    ↓
Pod running
    ↓
Node condition / taint change
    ↓
NoExecute evaluation
    ↓
Toleration duration
    ↓
Eviction
    ↓
Replacement / recovery
```

Capture when the Pod was scheduled, when the node condition and taint
changed, whether `NoExecute` was involved, whether a matching toleration
existed, whether `tolerationSeconds` was configured, and when eviction
occurred.

``` text
CURRENT STATE ≠ STATE AT INCIDENT TIME
```

## 23. Pod Eviction Requires Mechanism Identification

A Pod disappearing from a node can result from `NoExecute`,
node-pressure eviction, node failure, controller replacement, rollout,
manual deletion, or other lifecycle behavior.

``` text
POD EVICTED ≠ RESOURCE PRESSURE PROVEN
POD DISAPPEARED ≠ NoExecute PROVEN
```

Use Events, timestamps, workload history, and node evidence.

## 24. Direct Node Assignment Edge Case

If `.spec.nodeName` is explicitly assigned, the normal scheduler
node-selection path can be bypassed.

Inspect:

``` text
spec.nodeName
schedulerName
workload/controller behavior
binding history where available
```

``` text
POD ON NoSchedule-TAINTED NODE ≠ SCHEDULER VIOLATION PROVEN
```

`NoExecute` behavior remains separately relevant.

## 25. TrueFoundry Operational Context

``` text
TrueFoundry / Deployment Intent
             ↓
Desired Kubernetes Configuration
             ↓
Admission / Platform Processing
             ↓
Rendered Pod
             ↓
nodeSelector / affinity
             ↓
tolerations
             ↓
Scheduler
             ↓
Node labels + taints
             ↓
Eligible node set
             ↓
Resource fit
             ↓
Placement
```

``` text
ERROR VISIBLE THROUGH TRUEFOUNDRY ≠ TRUEFOUNDRY ROOT CAUSE
POD ≠ AUTHORITATIVE DESIRED-STATE OWNER
```

## 26. Desired-State Ownership

Tolerations or taints can be controlled by TrueFoundry, GitOps, Helm,
Kustomize, Terraform, Kubernetes operators, admission policy, managed
Kubernetes behavior, node provisioning systems, node-pool automation, or
Kubernetes control-plane behavior.

Before remediation, identify the authoritative owner.

``` text
OBSERVED DRIFT ≠ PERMISSION TO PATCH
```

``` text
DO NOT PATCH THE SYMPTOM
WHEN ANOTHER SYSTEM OWNS DESIRED STATE
```

## 27. Taint Changes Have Broad Blast Radius

Before considering a taint change, determine which nodes carry it, which
workloads tolerate it, which workloads intentionally do not tolerate it,
whether it protects dedicated capacity, whether it is condition-derived,
who owns it, whether automation will restore it, whether removing it
admits unintended workloads, and whether `NoExecute` could affect
running Pods.

``` text
TAINT CHANGE ≠ LOCAL POD FIX
```

Production validation should remain read-only unless an approved change
procedure explicitly authorizes mutation.

## 28. Workload Identity

Establish:

``` text
Environment
    ↓
Cluster / Context
    ↓
Namespace
    ↓
Workload
    ↓
Pod UID
    ↓
Node
    ↓
Scheduler
    ↓
Timestamp
```

``` text
UNKNOWN WORKLOAD IDENTITY = NO TAINT / TOLERATION CHANGE
```

## 29. Evidence Preservation

Before remediation, preserve:

``` text
Environment
Cluster/context
Namespace
Workload
Owner reference
Pod
Pod UID
Timestamp

Pod YAML
Pod Events

Rendered:
  nodeSelector
  affinity
  tolerations
  schedulerName
  nodeName

Node:
  labels
  taints
  conditions
  readiness
  schedulability
  allocatable resources

Desired workload configuration
Relevant historical timestamps
```

``` text
REMEDIATION CAN DESTROY INCIDENT EVIDENCE
```

## 30. Minimum Supported Blast Radius

Determine the smallest evidence-supported impact: one Pod, one workload,
one namespace, one node, one node pool, multiple workloads, or
cluster-wide.

``` text
OBSERVED BLAST RADIUS ≠ ASSUMED PLATFORM-WIDE IMPACT
```

## 31. Lowest Proven Healthy Layer

Example:

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
Required placement produced candidate nodes.

First Failed Transition:
Candidate nodes → taint/toleration compatibility.
```

## 32. Evidence Confidence

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
Candidate nodes currently carry that taint.

PROVEN
Rendered Pod lacks a matching toleration.

SUPPORTED
A deployment change correlates with incident start.

UNKNOWN
Why desired workload configuration omitted the toleration.

ASSUMED
TrueFoundry removed the toleration.
```

``` text
ASSUMED ≠ ROOT CAUSE
```

## 33. Evidence Handoff Contract

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

## 34. Production Troubleshooting Flow

``` text
IDENTITY
    ↓
POD STATE
    ↓
SCHEDULER / EVICTION EVIDENCE
    ↓
RENDERED PLACEMENT
    ↓
CANDIDATE NODES
    ↓
NODE TAINTS
    ↓
POD TOLERATIONS
    ↓
MATCH EACH RELEVANT TAINT
    ↓
REMAINING EFFECTS
    ↓
OTHER CONSTRAINTS
    ↓
RESOURCE FIT
    ↓
TIMELINE
    ↓
BLAST RADIUS
    ↓
LOWEST PROVEN HEALTHY LAYER
    ↓
FIRST FAILED TRANSITION
    ↓
DESIRED-STATE OWNER
    ↓
CURRENT ACTIONABLE OWNER
    ↓
SAFE REMEDIATION
```

## 35. Production Rules

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
MISSING TOLERATION CAN CAUSE EXCLUSION
OVERLY BROAD TOLERATION CAN CAUSE UNINTENDED ELIGIBILITY
TOLERATION ≠ AUTHORIZATION
TOLERATION ≠ SECURITY BOUNDARY
POD ON NoSchedule-TAINTED NODE ≠ SCHEDULER VIOLATION PROVEN
TAINT CHANGE ≠ LOCAL POD FIX
DESIRED ≠ RENDERED ≠ NODE STATE ≠ OBSERVED OUTCOME
POD ≠ AUTHORITATIVE DESIRED-STATE OWNER
OBSERVED DRIFT ≠ PERMISSION TO PATCH
ERROR VISIBLE THROUGH TRUEFOUNDRY ≠ TRUEFOUNDRY ROOT CAUSE
REMEDIATION CAN DESTROY INCIDENT EVIDENCE
ASSUMED ≠ ROOT CAUSE
SYMPTOM ≠ FAILURE DOMAIN ≠ OWNER
```

## 36. Scope Note

This tutorial focuses on **node taints and Pod tolerations**.

Modern Kubernetes also includes newer device-level taint concepts
associated with Dynamic Resource Allocation. Device-specific and
GPU-specific scheduling behavior is intentionally deferred to the GPU
infrastructure portion of this learning path.

## 37. Operational Takeaway

When a workload is Pending, unexpectedly placed, or evicted, do not
begin by changing taints or tolerations.

Establish identity, preserve scheduler and eviction evidence, inspect
the rendered Pod, compare exact node taints with exact Pod tolerations,
determine the remaining eligible-node set, distinguish placement
restrictions from resource capacity, reconstruct the timeline, and
identify the authoritative desired-state owner.

The objective is not merely to make the Pod schedule. It is to identify
the **first failed scheduling or runtime transition** and correct the
configuration or infrastructure at the layer that owns it.
