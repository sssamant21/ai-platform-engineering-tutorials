# Part 3.3 — Node Selectors & Node Affinity

## Purpose

CPU and memory availability alone do not determine whether Kubernetes can schedule a Pod. A node may have significant free resources but remain unavailable because it does not satisfy workload placement requirements.

This tutorial covers node labels, `nodeSelector`, required and preferred node affinity, expression logic, eligibility versus resource fit, label drift, scheduler evidence, production ownership, and safe remediation.

```text
NODE HAS RESOURCES
    ≠
NODE IS ELIGIBLE FOR THIS POD
```

## Production Scheduling Model

```text
ALL NODES
    ↓
Ready / schedulable candidates
    ↓
nodeSelector
    ↓
required nodeAffinity
    ↓
Other hard scheduling constraints
    ↓
Resource fit
    ↓
ELIGIBLE NODES
    ↓
Scheduler scoring
    ↓
preferred nodeAffinity + other scoring factors
    ↓
SELECTED NODE
```

```text
TOTAL CLUSTER CAPACITY
    ≠
SCHEDULABLE CAPACITY FOR THIS POD
```

## Establish Identity First

```bash
kubectl config current-context
kubectl get pod <pod> -n <namespace> -o wide
```

Capture environment, cluster, namespace, workload, Pod, Pod UID, node, scheduler, and timestamp.

```text
UNKNOWN WORKLOAD IDENTITY = NO PLACEMENT CHANGE
```

## Node Labels

```bash
kubectl get nodes --show-labels
kubectl get nodes -L node-pool,workload,gpu,gpu_type
```

Example labels include `node-pool=compute`, `environment=production`, `workload=inference`, `gpu=true`, and `gpu_type=A10G`.

```text
LABEL EXISTS ≠ WORKLOAD USES THAT LABEL
```

## nodeSelector

```yaml
spec:
  nodeSelector:
    node-pool: compute
```

All specified selector labels must match the node.

```text
FREE CPU/MEMORY ≠ PLACEMENT ELIGIBILITY
```

## Node Affinity

Important forms are:

```text
requiredDuringSchedulingIgnoredDuringExecution
preferredDuringSchedulingIgnoredDuringExecution
```

### Required Node Affinity

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: node-pool
              operator: In
              values:
                - compute
```

```text
REQUIRED AFFINITY → ELIGIBILITY / FILTERING
```

### Preferred Node Affinity

```yaml
affinity:
  nodeAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
            - key: node-pool
              operator: In
              values:
                - compute
```

Preferred affinity contributes to scheduler scoring.

```text
PREFERRED AFFINITY → SCORING PREFERENCE
PREFERRED AFFINITY ≠ GUARANTEED PLACEMENT
PREFERRED RULE NOT SATISFIED ≠ SCHEDULING FAILURE
```

## nodeSelector and Node Affinity Together

If both are configured, applicable requirements must be satisfied.

```text
nodeSelector
       AND
required nodeAffinity
       ↓
Node remains a candidate
```

```text
ONE MATCHING RULE ≠ COMPLETE PLACEMENT ELIGIBILITY
```

## Affinity Operators and Logic

Common operators include `In`, `NotIn`, `Exists`, `DoesNotExist`, `Gt`, and `Lt`.

Multiple `matchExpressions` within one `nodeSelectorTerm` are ANDed. Multiple `nodeSelectorTerms` are ORed.

```text
nodeSelectorTerms → OR
matchExpressions WITHIN A TERM → AND
ONE EXPRESSION MATCHES ≠ COMPLETE TERM MATCH
```

## Eligibility Before Capacity

Low aggregate utilization does not prove that a Pod has an eligible node.

```text
LOW CLUSTER UTILIZATION ≠ POD HAS AN ELIGIBLE NODE
```

Distinguish:

```text
NO ELIGIBLE NODE
    ≠
ELIGIBLE NODES EXIST BUT NONE HAVE RESOURCE FIT
```

The first points toward placement constraints. The second points toward capacity/resource fit.

## Start With Scheduler Evidence

```bash
kubectl describe pod <pod> -n <namespace>
```

Inspect scheduler Events before deciding which constraint failed.

```text
PENDING ≠ NODE AFFINITY FAILURE PROVEN
```

## Inspect the Rendered Pod

```bash
kubectl get pod <pod> -n <namespace> -o yaml
```

Inspect:

```text
spec.nodeSelector
spec.affinity
spec.tolerations
spec.schedulerName
spec.nodeName
```

```text
DESIRED PLACEMENT ≠ RENDERED PLACEMENT ≠ SCHEDULED STATE
```

## Inspect Candidate Nodes

```bash
kubectl get nodes --show-labels
kubectl get nodes -L node-pool,workload,gpu,gpu_type
```

Use this investigation order:

```text
Pod requirements
    ↓
Node labels
    ↓
Eligible node set
    ↓
Resource fit
    ↓
Other constraints
```

## Label Mismatch and Drift

If a Pod requires `node-pool=inference` while nodes expose `node-pool=inference-v2`, a mismatch is proven. Which side is wrong is not yet proven.

```text
LABEL MISMATCH PROVEN ≠ NODE LABEL IS WRONG
SEMANTICALLY SIMILAR LABEL ≠ MATCHING LABEL
```

## IgnoredDuringExecution

If relevant labels change after a Pod has already been scheduled, the running Pod is not automatically evicted merely because the affinity rule no longer matches.

```text
NODE LABEL NO LONGER MATCHES
    ≠
RUNNING POD AUTOMATICALLY EVICTED

RUNNING POD ON NONMATCHING NODE
    ≠
SCHEDULER VIOLATION PROVEN
```

Timeline matters:

```text
CURRENT STATE ≠ STATE AT SCHEDULING TIME
```

## Scheduler Identity

Inspect `spec.schedulerName`. Scheduler profiles can introduce additional behavior.

```text
VISIBLE POD PLACEMENT POLICY
    ≠
COMPLETE SCHEDULER BEHAVIOR PROVEN
```

Investigate custom scheduler/profile behavior only when evidence supports it.

## Placement Is More Than Affinity

Scheduling can also depend on CPU/memory fit, taints and tolerations, topology, storage, node readiness, node schedulability, and other constraints.

```text
NODE AFFINITY MATCHES ≠ POD CAN DEFINITELY SCHEDULE
```

Taints and tolerations are covered in Part 3.4.

## TrueFoundry Operational Context

```text
TrueFoundry / Deployment Intent
            ↓
Desired Workload Configuration
            ↓
Rendered Kubernetes Pod
            ↓
nodeSelector / nodeAffinity
            ↓
Scheduler Evaluation
            ↓
Node Label State
            ↓
Node Eligibility
            ↓
Resource Fit
            ↓
Placement
```

```text
POD ≠ AUTHORITATIVE DESIRED-STATE OWNER
```

## Desired-State Ownership

Placement configuration may be controlled by TrueFoundry, GitOps, Helm, Kustomize, Terraform, Kubernetes operators, admission configuration, node-pool automation, or another deployment system.

```text
OBSERVED DRIFT ≠ PERMISSION TO PATCH

DO NOT PATCH THE SYMPTOM
WHEN ANOTHER SYSTEM OWNS DESIRED STATE
```

## Node Label Changes Have Blast Radius

```text
NODE LABEL CHANGE ≠ LOCAL POD FIX
```

Before changing a label, establish who owns it, which nodes use it, which workloads reference it, whether it participates in workload isolation, and whether automation enforces it.

Placement labels may also be security-relevant configuration.

## Evidence Before Remediation

Capture environment, context, namespace, workload, Pod name/UID, timestamp, Pod YAML, scheduler Events, scheduler name, node selector, required/preferred affinity, tolerations, relevant node labels, node readiness/schedulability, allocatable resources, and desired workload configuration.

```text
REMEDIATION CAN DESTROY INCIDENT EVIDENCE
```

## Blast Radius and Failure Boundary

Determine whether impact is one Pod, ReplicaSet, Deployment, namespace, node pool, multiple workloads, or cluster-wide.

```text
OBSERVED BLAST RADIUS ≠ ASSUMED PLATFORM-WIDE IMPACT
```

Example failure chain:

```text
Kubernetes API reachable             HEALTHY
        ↓
Nodes registered                     HEALTHY
        ↓
Nodes Ready                          HEALTHY
        ↓
Pod accepted                         HEALTHY
        ↓
Scheduler evaluates Pod              HEALTHY
        ↓
Required placement → eligible node   FAILED
```

Record the **Lowest Proven Healthy Layer** and **First Failed Transition**.

## Evidence Confidence

Classify findings as:

```text
PROVEN
SUPPORTED
UNKNOWN
ASSUMED
```

```text
ASSUMED ≠ ROOT CAUSE
SYMPTOM ≠ FAILURE DOMAIN ≠ OWNER
```

## Production Troubleshooting Workflow

1. Establish environment and cluster.
2. Confirm namespace and workload.
3. Capture Pod UID and timestamp.
4. Confirm Pod state.
5. Read scheduler Events.
6. Inspect `schedulerName`.
7. Inspect rendered `nodeSelector`.
8. Inspect required node affinity.
9. Inspect preferred node affinity.
10. Evaluate AND/OR expression logic.
11. Inspect relevant node labels.
12. Determine nodes satisfying hard placement rules.
13. Check node readiness and schedulability.
14. Check other hard scheduling constraints.
15. Check resource fit on eligible nodes.
16. Distinguish eligibility failure from capacity failure.
17. Determine blast radius.
18. Build the incident timeline.
19. Identify Lowest Proven Healthy Layer.
20. Identify First Failed Transition.
21. Classify evidence confidence.
22. Determine authoritative desired-state ownership.
23. Identify Current Actionable Owner.
24. Preserve evidence.
25. Define the requested remediation action.

## Evidence Handoff Contract

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

## Final Production Rules

```text
UNKNOWN WORKLOAD IDENTITY = NO PLACEMENT CHANGE

NODE HAS RESOURCES ≠ NODE IS ELIGIBLE
TOTAL CLUSTER CAPACITY ≠ SCHEDULABLE CAPACITY FOR THIS POD
LOW CLUSTER UTILIZATION ≠ POD HAS AN ELIGIBLE NODE

LABEL EXISTS ≠ WORKLOAD USES THAT LABEL
LABEL EXISTS SOMEWHERE ≠ REQUIRED ELIGIBLE NODE EXISTS
SEMANTICALLY SIMILAR LABEL ≠ MATCHING LABEL

REQUIRED AFFINITY → ELIGIBILITY
PREFERRED AFFINITY → SCORING
PREFERRED AFFINITY ≠ GUARANTEED PLACEMENT
PREFERRED RULE NOT SATISFIED ≠ SCHEDULING FAILURE

nodeSelectorTerms → OR
matchExpressions WITHIN A TERM → AND
ONE EXPRESSION MATCHES ≠ COMPLETE TERM MATCH

NO ELIGIBLE NODE ≠ NO RESOURCE-FIT NODE
PENDING ≠ NODE AFFINITY FAILURE PROVEN
NODE AFFINITY MATCHES ≠ POD CAN DEFINITELY SCHEDULE

LABEL MISMATCH PROVEN ≠ NODE LABEL IS WRONG
DESIRED ≠ RENDERED ≠ SCHEDULED
CURRENT STATE ≠ STATE AT SCHEDULING TIME

NODE LABEL NO LONGER MATCHES ≠ RUNNING POD AUTOMATICALLY EVICTED
RUNNING POD ON NONMATCHING NODE ≠ SCHEDULER VIOLATION PROVEN

VISIBLE POD PLACEMENT POLICY ≠ COMPLETE SCHEDULER BEHAVIOR PROVEN
NODE LABEL CHANGE ≠ LOCAL POD FIX
OBSERVED DRIFT ≠ PERMISSION TO PATCH
POD ≠ AUTHORITATIVE DESIRED-STATE OWNER

REMEDIATION CAN DESTROY INCIDENT EVIDENCE
ASSUMED ≠ ROOT CAUSE
SYMPTOM ≠ FAILURE DOMAIN ≠ OWNER
```

## Completion Check

You should now be able to distinguish node eligibility from resource capacity, interpret `nodeSelector` and required/preferred node affinity, evaluate affinity expression logic, use scheduler evidence to troubleshoot Pending Pods, recognize label drift without prematurely assigning root cause, and identify the authoritative desired-state owner before remediation.

## Next Tutorial

**Part 3.4 — Taints & Tolerations**
