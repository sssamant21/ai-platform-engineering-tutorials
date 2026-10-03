# Part 3.3 — Node Selectors & Node Affinity

# [SAFE-READ] Production Validation Lab

## Purpose

This lab validates node selectors, node affinity, node eligibility, scheduler evidence, and placement behavior without modifying production resources.

## Safety Boundary

This lab is observational only.

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

Do not change node labels, selectors, affinity, tolerations, requests, limits, replicas, node-pool configuration, or scheduler configuration.

```text
UNKNOWN TARGET = NO MUTATION
UNKNOWN WORKLOAD IDENTITY = NO PLACEMENT CHANGE
```

## Variables

Replace these placeholders only after confirming the target:

```text
<NAMESPACE>
<POD>
<WORKLOAD>
<NODE>
```

## Phase 1 — Establish Cluster and Workload Identity

```bash
kubectl config current-context
kubectl get pod <POD> -n <NAMESPACE> -o wide
```

Capture:

```text
Environment:
Cluster/context:
Namespace:
Workload:
Pod:
Pod UID:
Current node:
Pod phase:
Timestamp:
```

Retrieve the UID read-only:

```bash
kubectl get pod <POD> -n <NAMESPACE> -o jsonpath='{.metadata.uid}'
```

Do not continue if the target cannot be established confidently.

## Phase 2 — Capture Scheduler Evidence

```bash
kubectl describe pod <POD> -n <NAMESPACE>
```

Record:

```text
Scheduler events:
FailedScheduling present:
Reported reason:
First observed timestamp:
Latest observed timestamp:
```

Rule:

```text
PENDING ≠ NODE AFFINITY FAILURE PROVEN
```

## Phase 3 — Inspect Rendered Placement Configuration

```bash
kubectl get pod <POD> -n <NAMESPACE> -o yaml
```

Review:

```text
spec.nodeSelector
spec.affinity.nodeAffinity
spec.tolerations
spec.schedulerName
spec.nodeName
```

Optional focused reads:

```bash
kubectl get pod <POD> -n <NAMESPACE> -o jsonpath='{.spec.nodeSelector}'
kubectl get pod <POD> -n <NAMESPACE> -o jsonpath='{.spec.affinity.nodeAffinity}'
kubectl get pod <POD> -n <NAMESPACE> -o jsonpath='{.spec.schedulerName}'
```

Record:

```text
Rendered nodeSelector:
Required node affinity:
Preferred node affinity:
Scheduler name:
Assigned node:
```

```text
DESIRED ≠ RENDERED ≠ SCHEDULED
```

## Phase 4 — Evaluate nodeSelector

If `nodeSelector` exists, list node labels:

```bash
kubectl get nodes --show-labels
```

For known label keys, use focused output:

```bash
kubectl get nodes -L node-pool,workload,gpu,gpu_type
```

Determine which nodes satisfy every rendered `nodeSelector` requirement.

Record:

```text
nodeSelector present:
Required key/value pairs:
Matching nodes:
Nonmatching nodes:
```

```text
LABEL EXISTS SOMEWHERE ≠ REQUIRED ELIGIBLE NODE EXISTS
```

## Phase 5 — Evaluate Required Node Affinity

Inspect the complete required expression.

For each `nodeSelectorTerm`, capture:

```text
Term:
  Expression:
    key:
    operator:
    values:

Result:
```

Remember:

```text
nodeSelectorTerms → OR
matchExpressions WITHIN A TERM → AND
```

Do not declare a node eligible because only one expression matches.

```text
ONE EXPRESSION MATCHES ≠ COMPLETE TERM MATCH
```

## Phase 6 — Evaluate Preferred Node Affinity

Capture preferred rules and weights.

```text
Preferred rule:
Weight:
Matching nodes:
Nonmatching nodes:
```

Do not treat a preferred mismatch as a hard scheduling failure.

```text
PREFERRED AFFINITY → SCORING
PREFERRED AFFINITY ≠ GUARANTEED PLACEMENT
PREFERRED RULE NOT SATISFIED ≠ SCHEDULING FAILURE
```

## Phase 7 — Build the Placement Eligibility Funnel

Create an evidence table:

```text
All nodes:
Ready/schedulable candidates:
nodeSelector matches:
Required-affinity matches:
Other hard-constraint survivors:
Resource-fit candidates:
Final eligible candidates:
```

Reason through:

```text
ALL NODES
    ↓
Ready / schedulable
    ↓
nodeSelector
    ↓
required nodeAffinity
    ↓
other hard constraints
    ↓
resource fit
    ↓
ELIGIBLE NODES
```

```text
NODE HAS RESOURCES ≠ NODE IS ELIGIBLE FOR THIS POD
```

## Phase 8 — Check Node Readiness and Schedulability

```bash
kubectl get nodes -o wide
kubectl describe node <NODE>
```

Capture:

```text
Node:
Ready:
Unschedulable:
Relevant conditions:
Relevant labels:
Allocatable CPU:
Allocatable memory:
```

Do not modify the node.

## Phase 9 — Distinguish Eligibility From Resource Fit

If eligible placement nodes exist, inspect resource context:

```bash
kubectl describe node <NODE>
```

If metrics are available:

```bash
kubectl top node <NODE>
```

Treat `kubectl top` as runtime usage evidence, not scheduler accounting.

Classify:

```text
A. No nodes satisfy hard placement constraints

or

B. Placement-eligible nodes exist, but none satisfy resource fit

or

C. Placement and resource fit appear viable; another constraint requires investigation
```

```text
NO ELIGIBLE NODE ≠ NO RESOURCE-FIT NODE
LOW CLUSTER UTILIZATION ≠ POD HAS AN ELIGIBLE NODE
```

## Phase 10 — Check Other Scheduling Constraints

Read-only inspection may include:

```bash
kubectl get pod <POD> -n <NAMESPACE> -o yaml
kubectl describe pod <POD> -n <NAMESPACE>
```

Look for evidence involving:

```text
Taints / tolerations
Topology constraints
PVC / storage requirements
Node readiness
Node schedulability
Resource fit
Other scheduler constraints
```

Do not troubleshoot all of these deeply in this lab. Record them when they affect the placement conclusion.

```text
NODE AFFINITY MATCHES ≠ POD CAN DEFINITELY SCHEDULE
```

## Phase 11 — Check Scheduler Identity

```bash
kubectl get pod <POD> -n <NAMESPACE> -o jsonpath='{.spec.schedulerName}'
```

Record whether the default or another scheduler is named.

Do not assume that visible Pod affinity proves the complete scheduler behavior.

```text
VISIBLE POD PLACEMENT POLICY
    ≠
COMPLETE SCHEDULER BEHAVIOR PROVEN
```

Only escalate into scheduler-profile investigation when evidence supports it.

## Phase 12 — Investigate Label Mismatch Safely

If the workload requires a label that current nodes do not satisfy, record both sides exactly.

```text
Rendered requirement:
Current node labels:
Mismatch:
```

Do not immediately decide which configuration is wrong.

```text
LABEL MISMATCH PROVEN ≠ NODE LABEL IS WRONG
SEMANTICALLY SIMILAR LABEL ≠ MATCHING LABEL
```

Never fix this lab by running `kubectl label`.

## Phase 13 — Running Pod / IgnoredDuringExecution Check

For a running Pod, compare:

```bash
kubectl get pod <POD> -n <NAMESPACE> -o wide
kubectl get pod <POD> -n <NAMESPACE> -o yaml
kubectl get node <NODE> --show-labels
```

If current labels no longer satisfy affinity, do not conclude that the scheduler violated policy.

```text
CURRENT STATE ≠ STATE AT SCHEDULING TIME

NODE LABEL NO LONGER MATCHES
    ≠
RUNNING POD AUTOMATICALLY EVICTED

RUNNING POD ON NONMATCHING NODE
    ≠
SCHEDULER VIOLATION PROVEN
```

Record whether historical label state is known or unknown.

## Phase 14 — Build the Timeline

Capture available timestamps for:

```text
Pod creation
Scheduling attempts
FailedScheduling events
Deployment change
Node-pool change
Node-label change, if known
Node join/removal
Incident start
Recovery
```

Do not infer historical state solely from current labels.

## Phase 15 — Determine Blast Radius

Read-only discovery:

```bash
kubectl get pods -n <NAMESPACE> -o wide
kubectl get pods -A -o wide
```

Use broad cluster output only when authorized and operationally appropriate.

Classify the smallest supported scope:

```text
One Pod
One ReplicaSet
One Deployment
One namespace
One node pool
Multiple workloads
Cluster-wide
```

```text
OBSERVED BLAST RADIUS ≠ ASSUMED PLATFORM-WIDE IMPACT
```

## Phase 16 — Lowest Proven Healthy Layer

Document the evidence chain:

```text
Kubernetes API:
Nodes registered:
Nodes Ready:
Pod accepted:
Scheduler evaluating Pod:
Placement requirements:
Eligible node:
Resource fit:
```

Then record:

```text
Lowest Proven Healthy Layer:
First Failed Transition:
```

Example:

```text
Lowest Proven Healthy Layer:
Scheduler evaluates the Pod.

First Failed Transition:
Required placement → eligible node.
```

## Phase 17 — Evidence Confidence

Classify each important finding:

```text
PROVEN
SUPPORTED
UNKNOWN
ASSUMED
```

Example:

```text
PROVEN:
Scheduler event reports node-affinity mismatch.

PROVEN:
No current node satisfies the rendered required expression.

SUPPORTED:
A node-pool change correlates with the incident.

UNKNOWN:
Why desired configuration references the old label.

ASSUMED:
A particular platform component caused the mismatch.
```

```text
ASSUMED ≠ ROOT CAUSE
```

## Phase 18 — Determine Desired-State Ownership

Without changing anything, identify the likely authoritative configuration source from available evidence:

```text
TrueFoundry
Git / GitOps
Helm
Kustomize
Terraform
Kubernetes operator
Admission configuration
Node-pool automation
Other deployment automation
UNKNOWN
```

Record:

```text
Authoritative desired-state owner:
Evidence:
Confidence:
```

```text
OBSERVED DRIFT ≠ PERMISSION TO PATCH

DO NOT PATCH THE SYMPTOM
WHEN ANOTHER SYSTEM OWNS DESIRED STATE
```

## Phase 19 — Determine Current Actionable Owner

Separate the visible symptom from the team or system that can act on the proven failure boundary.

Record:

```text
Symptom:
Failure domain:
Authoritative owner:
Current Actionable Owner:
Requested Action:
```

```text
SYMPTOM ≠ FAILURE DOMAIN ≠ OWNER
```

## Phase 20 — Evidence Handoff Contract

Complete this before escalation:

```text
Environment:
Cluster:
Namespace:
Workload:
Pod:
Pod UID:
Timestamp:

Symptom:
Blast Radius:

Scheduler Events:
Scheduler Name:

Rendered nodeSelector:
Rendered Required Affinity:
Rendered Preferred Affinity:

Relevant Node Labels:
Candidate Nodes:
Eligible Nodes:

Resource Fit:
Other Scheduling Constraints:

Timeline:

Lowest Proven Healthy Layer:
First Failed Transition:

Evidence Confidence:

Authoritative Desired-State Owner:
Current Actionable Owner:

Requested Action:
```

## Phase 21 — Final Safety Validation

Before ending the investigation, confirm:

```text
[ ] Correct cluster/context was verified.
[ ] Correct namespace was verified.
[ ] Pod UID was captured.
[ ] Scheduler Events were preserved.
[ ] Rendered placement configuration was captured.
[ ] Relevant node labels were captured.
[ ] Required and preferred affinity were distinguished.
[ ] Eligibility and resource fit were distinguished.
[ ] Current state was not assumed to equal scheduling-time state.
[ ] Blast radius was evidence-based.
[ ] Desired-state ownership was considered.
[ ] No workload was modified.
[ ] No node label was modified.
[ ] No taint was modified.
[ ] No Pod was deleted or restarted.
[ ] No resource request/limit was changed.
[ ] No node-pool configuration was changed.
```

## Lab Completion Criteria

The lab is complete when you can answer:

1. What exact placement rules are rendered on the Pod?
2. Which rules are hard requirements and which are preferences?
3. Which nodes satisfy the hard placement requirements?
4. Are there zero eligible nodes, or eligible nodes without resource fit?
5. What do scheduler Events prove?
6. Does current node state represent scheduling-time state?
7. What is the Minimum Supported Blast Radius?
8. What is the Lowest Proven Healthy Layer?
9. What is the First Failed Transition?
10. Which conclusions are PROVEN, SUPPORTED, UNKNOWN, or ASSUMED?
11. Who owns the authoritative desired state?
12. What is the Current Actionable Owner and requested action?

## Core Lab Rules

```text
UNKNOWN TARGET = NO MUTATION
UNKNOWN WORKLOAD IDENTITY = NO PLACEMENT CHANGE

NODE HAS RESOURCES ≠ NODE IS ELIGIBLE
LOW CLUSTER UTILIZATION ≠ POD HAS AN ELIGIBLE NODE

REQUIRED AFFINITY → ELIGIBILITY
PREFERRED AFFINITY → SCORING

NO ELIGIBLE NODE ≠ NO RESOURCE-FIT NODE
PENDING ≠ NODE AFFINITY FAILURE PROVEN

LABEL MISMATCH PROVEN ≠ NODE LABEL IS WRONG
CURRENT STATE ≠ STATE AT SCHEDULING TIME

OBSERVED DRIFT ≠ PERMISSION TO PATCH
NODE LABEL CHANGE ≠ LOCAL POD FIX

REMEDIATION CAN DESTROY INCIDENT EVIDENCE
ASSUMED ≠ ROOT CAUSE
SYMPTOM ≠ FAILURE DOMAIN ≠ OWNER
```
