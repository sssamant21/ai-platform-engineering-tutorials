# Part 3.6 — Workload Scheduling Troubleshooting

## 1. Objective

When a Kubernetes workload remains Pending, do not immediately add nodes, increase resources, remove taints, or modify affinity.

The investigation must answer:

```text
WHY DOES THIS POD CURRENTLY HAVE NO VALID PLACEMENT?
```

Use evidence to determine:

```text
Workload Identity
      ↓
Scheduling Readiness
      ↓
Rendered Scheduling Contract
      ↓
Scheduler Evidence
      ↓
Candidate-Node Reduction
      ↓
First Failed Transition
      ↓
Failure Domain
      ↓
Current Actionable Owner
```

Core principle:

```text
POD PENDING = SYMPTOM
NOT ROOT CAUSE
```

---

# Phase 1 — Identify

## 2. Establish Production Identity

Start with read-only checks:

```bash
kubectl config current-context
kubectl get pod <pod> -n <namespace> -o wide
```

Capture:

```text
Environment:
Cluster:
Namespace:
Workload:
Pod:
Pod UID:
Timestamp:
```

Identify the owner:

```bash
kubectl get pod <pod> -n <namespace> \
  -o jsonpath="{range .metadata.ownerReferences[*]}{.kind}{'\t'}{.name}{'\n'}{end}"
```

Production rule:

```text
UNKNOWN WORKLOAD IDENTITY = NO MUTATION
```

Never troubleshoot an assumed namespace, cluster, or Pod.

## 3. Establish Minimum Supported Blast Radius

Determine what is actually affected:

```text
One Pod?
One workload?
One namespace?
One node?
One node pool?
One zone?
One workload class?
Multiple pools?
Cluster-wide?
```

Do not turn one Pending Pod into a cluster-wide scheduling incident without supporting evidence.

```text
OBSERVED IMPACT ≠ ASSUMED BLAST RADIUS
```

---

# Phase 2 — Prove Scheduling State

## 4. Confirm the Current Pod State

```bash
kubectl get pod <pod> -n <namespace> -o wide
```

Capture the phase, assigned node, creation timestamp, and restart count.

```text
POD PENDING ≠ SCHEDULER FAILURE
```

Pending is a lifecycle state, not a root-cause analysis.

## 5. Determine Scheduling Readiness

Inspect the rendered Pod:

```bash
kubectl get pod <pod> -n <namespace> -o yaml
```

Check:

```text
schedulerName
schedulingGates
nodeName
```

A scheduling gate can intentionally prevent a Pod from entering normal scheduling. Where applicable, Kubernetes can expose a `SchedulingGated` reason.

```text
POD PENDING ≠ SCHEDULER IS CURRENTLY TRYING TO PLACE IT
SCHEDULING READINESS ≠ SCHEDULING FEASIBILITY
```

First prove that scheduling should occur. Then determine whether placement is feasible.

## 6. Check Whether Normal Scheduler Placement Applies

Normally kube-scheduler selects the node. A Pod with `.spec.nodeName` can instead be assigned directly to a node and bypass normal scheduler selection.

```text
POD ASSIGNED TO NODE ≠ KUBE-SCHEDULER SELECTED THAT NODE
```

This distinction is important during storage and taint investigations.

## 7. Capture Scheduler Events Immediately

Run early:

```bash
kubectl describe pod <pod> -n <namespace>
```

Preserve:

```text
Reason:
Message:
First timestamp:
Last timestamp:
Count:
```

A scheduler message can contain multiple exclusion populations, for example insufficient memory, untolerated taints, and affinity mismatches. Do not collapse the evidence into a generic "capacity issue".

```text
SCHEDULER SUMMARY ≠ SINGLE ROOT CAUSE
SCHEDULER REASON ≠ UNDERLYING ROOT CAUSE
```

A scheduler reason explains why placement failed against the evaluated state. It does not automatically explain why the system reached that state.

## 8. Preserve Evidence Before Remediation

Before Pod deletion, rollout restart, manual scaling, node replacement, label changes, taint changes, affinity changes, or resource-request changes, capture the incident evidence.

```text
REMEDIATION CAN DESTROY INCIDENT EVIDENCE
POD RECREATION CAN DESTROY ORIGINAL SCHEDULING EVIDENCE
MANUAL SCALE-UP CAN DESTROY AUTOSCALER FAILURE EVIDENCE
```

## 9. Inspect the Rendered Pod

The scheduler evaluates the actual Pod, not necessarily the source configuration someone expected.

Capture:

```text
resource requests
resource limits
nodeSelector
nodeAffinity
tolerations
schedulerName
schedulingGates
topologySpreadConstraints
PVC requirements
nodeName
```

Model:

```text
Desired Configuration
       ↓
Platform / GitOps / Template
       ↓
Admission / Defaulting
       ↓
Rendered Pod
       ↓
Scheduler
```

```text
DESIRED ≠ RENDERED ≠ SCHEDULED
SOURCE MANIFEST ≠ RENDERED POD RESOURCE CONTRACT
```

LimitRange/defaulting or other admission behavior can make the actual Pod different from the source manifest.

## 10. Check Scheduler Identity

```bash
kubectl get pod <pod> -n <namespace> \
  -o jsonpath="{.spec.schedulerName}{'\n'}"
```

The common scheduler is `default-scheduler`. If a custom scheduler/profile is involved, scheduler configuration can add constraints such as `addedAffinity`.

```text
VISIBLE POD AFFINITY ≠ COMPLETE EFFECTIVE AFFINITY
WHEN CUSTOM SCHEDULER PROFILES ARE USED
```

Treat this as an advanced path when scheduler identity makes it relevant.

---

# Phase 3 — Reduce Candidate Nodes

## 11. Use a Candidate-Node Funnel

For incident investigation, model placement as progressive candidate reduction:

```text
ALL NODES
    ↓
Registered
    ↓
Ready / Schedulable
    ↓
nodeSelector Compatible
    ↓
Required-Affinity Compatible
    ↓
Taint-Compatible
    ↓
Topology-Compatible
    ↓
Storage-Compatible
    ↓
Resource-Fit
    ↓
FEASIBLE NODES
    ↓
Scoring
    ↓
Selected Node
```

This is an **SRE diagnostic model**, not a claim about the exact internal execution order of every kube-scheduler plugin.

The central question is:

```text
WHERE DID THE CANDIDATE SET BECOME ZERO?
```

## 12. Count Candidates

Do not stop at a generic statement such as "affinity mismatch". Record candidate counts at each useful transition.

Example:

```text
Registered Nodes                 20
Ready/Schedulable                19
Placement-Compatible              8
Taint-Compatible                  6
Topology-Compatible               4
Storage-Compatible                4
Resource-Fit                      0
Final Feasible                    0
```

Then record:

```text
Lowest Proven Healthy Layer:
Storage compatibility

First Failed Transition:
Storage-compatible → resource-fit
```

## 13. Maintain an Exclusion Ledger

For difficult incidents, record why nodes leave the candidate population:

```text
Node A → required affinity
Node B → untolerated taint
Node C → insufficient memory
Node D → topology
Node E → storage
Node F → feasible
```

```text
ONE NODE'S STATE ≠ CLUSTER SCHEDULABILITY FOR THE POD
```

A node may fail multiple constraints. Avoid double-counting nodes when summarizing candidate populations.

## 14. Validate Node Health

```bash
kubectl get nodes
kubectl describe node <node>
```

Inspect:

```text
Ready
Unschedulable
MemoryPressure
DiskPressure
PIDPressure
NetworkUnavailable
condition-related taints
```

```text
NODE EXISTS ≠ NODE READY
NODE READY ≠ NODE ELIGIBLE
NODE ELIGIBLE ≠ RESOURCE FIT
```

## 15. Validate nodeSelector

```bash
kubectl get pod <pod> -n <namespace> \
  -o jsonpath="{.spec.nodeSelector}{'\n'}"

kubectl get nodes --show-labels
```

If the Pod requires `node-pool=ai` and no Ready node has that label:

```text
First Failed Transition:
Ready Nodes → nodeSelector-compatible Nodes
```

Do not automatically patch a node:

```text
LABEL MISMATCH PROVEN ≠ MANUAL LABEL PATCH AUTHORIZED
```

## 16. Validate Node Affinity

Hard requirement:

```text
requiredDuringSchedulingIgnoredDuringExecution
```

Preference:

```text
preferredDuringSchedulingIgnoredDuringExecution
```

Rules:

```text
REQUIRED AFFINITY → ELIGIBILITY
PREFERRED AFFINITY → SCORING
PREFERRED AFFINITY ≠ HARD PLACEMENT REQUIREMENT
```

Remember that scheduler profiles can add effective affinity not visible directly in the Pod specification.

## 17. Validate Taints and Tolerations

Inspect node taints and Pod tolerations:

```bash
kubectl describe node <node>
kubectl get pod <pod> -n <namespace> -o yaml
```

Evaluate all relevant taints.

```text
TAINT ≠ TOLERATION
TOLERATION ≠ PLACEMENT REQUIREMENT
ONE TAINT TOLERATED ≠ ALL RELEVANT TAINTS TOLERATED
```

A toleration permits compatibility; it does not attract the workload to that node.

## 18. Validate Topology

Investigate separately:

```text
Node affinity
Pod affinity
Pod anti-affinity
Topology spread constraints
Zone restrictions
Storage topology
```

```text
TOTAL CLUSTER CAPACITY ≠ CAPACITY IN THE REQUIRED TOPOLOGY
```

Advanced topology-spread behavior can depend on Kubernetes version and rendered configuration. Inspect the actual cluster and workload behavior before concluding.

## 19. Validate Storage Constraints

For stateful workloads:

```bash
kubectl get pvc -n <namespace>
kubectl describe pvc <pvc> -n <namespace>
kubectl get pv <pv> -o yaml
```

Inspect:

```text
PVC state
StorageClass
volumeBindingMode
PV node affinity
zone/topology restrictions
Pod placement requirements
```

```text
PVC PENDING ≠ STORAGE FAILURE PROVEN
```

With `WaitForFirstConsumer`, volume binding/provisioning can intentionally wait for scheduling context.

## 20. Watch for nodeName + WaitForFirstConsumer

Direct node assignment bypasses normal scheduler participation. Combining `.spec.nodeName` with a StorageClass using `WaitForFirstConsumer` can prevent the expected scheduler-assisted volume-binding workflow from progressing.

Treat this as an advanced storage/scheduling edge case.

## 21. Validate Resource Fit

For each relevant node compare:

```text
Pod Request
       vs
Node Allocatable
       vs
Existing Requested Resources
```

Example:

```text
Node allocatable memory:       60 Gi
Existing requested memory:     52 Gi
New Pod request:               16 Gi
Runtime memory usage:          24 Gi
```

Runtime usage may look low, but the new request still does not fit according to scheduler accounting.

```text
REQUEST ≠ USAGE
kubectl top ≠ SCHEDULER CAPACITY MODEL
LOW RUNTIME UTILIZATION ≠ SCHEDULABLE CAPACITY
```

## 22. Do Not Use Aggregate Capacity as Pod Fit

Example:

```text
Node A → 4 CPU available
Node B → 4 CPU available
Pod request → 8 CPU
```

The cluster may have 8 CPU in aggregate, but no individual node fits the Pod.

```text
AGGREGATE CLUSTER CAPACITY ≠ POD FIT
```

## 23. Account for Fragmentation

Example:

```text
Node A
CPU       PASS
Memory    FAIL

Node B
CPU       FAIL
Memory    PASS

Node C
CPU       PASS
Memory    PASS
Affinity  FAIL
```

Result:

```text
Feasible Nodes = 0
```

```text
LOW CLUSTER UTILIZATION ≠ SCHEDULABLE CAPACITY EXISTS FOR THIS POD
```

## 24. Keep QoS in Context

QoS is useful operational evidence but should not replace the actual scheduling investigation.

```text
QoS CLASS ≠ SCHEDULING ROOT CAUSE
```

Use scheduler Events, resource requests, placement constraints, and candidate nodes first.

---

# Phase 4 — Investigate the Capacity Path

## 25. Dedicated Node Pools

For dedicated capacity:

```text
Expected Pool
    ↓
Provisioned Nodes
    ↓
Registered Nodes
    ↓
Ready Nodes
    ↓
Placement-Compatible
    ↓
Taint-Compatible
    ↓
Topology/Storage-Compatible
    ↓
Resource-Fit
    ↓
Feasible Nodes
```

Record each count.

```text
POOL SIZE ≠ READY NODE COUNT ≠ FEASIBLE NODE COUNT
```

## 26. Validate Pool Consistency

Compare expected pool nodes for:

```text
labels
taints
instance type
zone
capacity
allocatable
special resources
readiness
conditions
container runtime
Kubernetes version
```

```text
SAME NODE POOL ≠ IDENTICAL EFFECTIVE NODE STATE
NODE BELONGS TO EXPECTED POOL ≠ NODE HAS EXPECTED EFFECTIVE CONFIGURATION
```

## 27. Do Not Scale First

Before recommending more nodes ask:

```text
WOULD ADDING A CORRECTLY CONFIGURED NODE MAKE THIS POD SCHEDULABLE?
```

If the answer is no because of an incorrect selector, required affinity, missing toleration, scheduling gate, storage topology, unsupported topology, or incorrect rendered configuration, adding nodes may accomplish nothing.

```text
MORE NODES ≠ MORE FEASIBLE NODES
```

## 28. Separate Autoscaling from Scheduling

If correctly configured additional capacity would help:

```text
Unschedulable Pod
       ↓
Autoscaler observes demand
       ↓
Scale decision
       ↓
Provisioning request
       ↓
Machine created
       ↓
Node bootstrap
       ↓
Node registration
       ↓
Node Ready
       ↓
Expected labels / taints / resources
       ↓
Scheduler reevaluation
       ↓
Pod scheduled
```

Every transition is independently investigable.

```text
SCHEDULING SYMPTOM ≠ SCHEDULER FAILURE
```

## 29. Classify Scale-Up Outcome

Use:

```text
A — Scale-up not required
B — Scale-up required but not triggered
C — Scale-up triggered but provisioning failed
D — Capacity created but Pod still cannot schedule
```

For outcome D investigate wrong labels, wrong taints, wrong instance type, wrong topology, insufficient node size, storage incompatibility, or workload constraint mismatch.

```text
NODE CREATED ≠ USEFUL CAPACITY CREATED
```

## 30. Separate Infrastructure States

Do not collapse:

```text
Desired capacity
      ↓
Provisioned machine
      ↓
Registered Node
      ↓
Ready Node
      ↓
Correctly configured Node
      ↓
Feasible Node
```

```text
PROVISIONED ≠ REGISTERED
REGISTERED ≠ READY
READY ≠ CORRECTLY CONFIGURED
CORRECTLY CONFIGURED ≠ FEASIBLE
```

## 31. Build a Time-to-Capacity Timeline

Capture:

```text
Pod Pending:
Autoscaler Detection:
Scale Decision:
Provisioning Request:
Machine Created:
Node Registered:
Node Ready:
Pod Scheduled:
```

This distinguishes detection, decision, provisioning, bootstrap, registration, readiness, and scheduling delays.

Without timestamps, normal capacity startup can be mistaken for a scheduler outage.

---

# Phase 5 — Conclude, Hand Off & Validate

## 32. Account for Historical State

During RCA:

```text
CURRENT LABELS ≠ LABELS AT FAILURE TIME
CURRENT TAINTS ≠ TAINTS AT FAILURE TIME
CURRENT CAPACITY ≠ CAPACITY AT FAILURE TIME
CURRENT NODE HEALTH ≠ NODE HEALTH AT FAILURE TIME
```

Use timestamps, Events, monitoring, autoscaler logs, provisioning records, and configuration history.

```text
CURRENT STATE ≠ HISTORICAL INCIDENT STATE
```

## 33. Identify Lowest Proven Healthy Layer

Example:

```text
Rendered Pod              HEALTHY
Scheduling readiness      HEALTHY
nodeSelector              HEALTHY
Required affinity         HEALTHY
Taint compatibility       HEALTHY
Storage compatibility     HEALTHY
Resource fit              FAILED
```

Record:

```text
Lowest Proven Healthy Layer:
Storage compatibility

First Failed Transition:
Storage-compatible → resource-fit
```

## 34. First Failed Transition Is Not Automatically Final RCA

Example:

```text
First Failed Transition:
Provisioning request → machine creation

Immediate Failure:
Cloud capacity unavailable
```

Further investigation may still be required to understand the underlying cause.

```text
FIRST FAILED TRANSITION ≠ FINAL ROOT CAUSE AUTOMATICALLY
```

But once the transition is strongly proven:

```text
ONCE FIRST FAILED TRANSITION IS PROVEN,
INVESTIGATE THAT FAILURE DOMAIN
```

Do not repeatedly re-investigate already-proven healthy layers without new evidence.

## 35. Assign Evidence Confidence

Use:

```text
PROVEN
SUPPORTED
UNKNOWN
ASSUMED
```

Example:

```text
PROVEN
Pod requests 32 GiB memory.

PROVEN
No eligible Node can currently fit 32 GiB.

SUPPORTED
Dedicated pool capacity is insufficient.

UNKNOWN
Why autoscaling did not add capacity.

ASSUMED
TrueFoundry caused the scheduling failure.
```

```text
ASSUMED ≠ ROOT CAUSE
```

## 36. Identify Authoritative Desired-State Ownership

Before modifying anything determine who owns:

```text
requests / limits
nodeSelector
affinity
tolerations
topology rules
node labels
node taints
node-pool size
instance type
autoscaling
bootstrap configuration
storage topology
```

Potential authoritative owners include application configuration, TrueFoundry, GitOps, Terraform/IaC, Kubernetes platform, cloud infrastructure, autoscaler/provisioner, and storage platform.

```text
OBSERVED DRIFT ≠ PERMISSION TO PATCH
```

## 37. Determine Current Actionable Owner

Example:

```text
Symptom:
Pod Pending

First Failed Transition:
Taint-compatible → resource-fit

Failure Domain:
Dedicated node-pool capacity

Evidence Confidence:
PROVEN

Current Actionable Owner:
Infrastructure capacity owner

Requested Action:
Validate supported capacity expansion.
```

```text
SYMPTOM ≠ FAILURE DOMAIN ≠ OWNER
```

## 38. TrueFoundry Context

For a TrueFoundry-managed workload:

```text
TrueFoundry Deployment Intent
       ↓
Rendered Kubernetes Workload
       ↓
Rendered Pod
       ↓
Scheduler
       ↓
Node / Dedicated Pool
       ↓
Autoscaler
       ↓
Cloud Infrastructure
```

A scheduling error can be visible through TrueFoundry while originating elsewhere.

```text
ERROR VISIBLE THROUGH TRUEFOUNDRY ≠ TRUEFOUNDRY ROOT CAUSE
```

## 39. Evidence Handoff Contract

A production handoff should contain:

```text
Environment:
Cluster:
Namespace:

Workload:
Workload Owner:
Pod:
Pod UID:
Timestamp:

Symptom:
Minimum Supported Blast Radius:

Pod Phase:
Scheduler Name:
Scheduling Gates:
nodeName:

Scheduler Event Reason:
Scheduler Event Message:
First Event:
Last Event:
Event Count:

Rendered Requests:
Rendered Limits:
Rendered nodeSelector:
Rendered Required Affinity:
Rendered Preferred Affinity:
Rendered Tolerations:
Rendered Topology:
Rendered Storage Requirements:

Registered Nodes:
Ready/Schedulable Nodes:
Placement-Compatible Nodes:
Taint-Compatible Nodes:
Topology-Compatible Nodes:
Storage-Compatible Nodes:
Resource-Fit Nodes:
Final Feasible Nodes:

Exclusion Ledger:

Intended Node Pool:
Expected Pool Size:
Provisioned Nodes:
Registered Pool Nodes:
Ready Pool Nodes:

Autoscaling Enabled:
Autoscaler Evidence:
Provisioning Evidence:

Pod Pending:
Autoscaler Detection:
Provisioning Request:
Machine Created:
Node Registered:
Node Ready:
Pod Scheduled:

Lowest Proven Healthy Layer:
First Failed Transition:

Evidence Confidence:
Failure Domain:
Authoritative Desired-State Owner:
Current Actionable Owner:

Requested Action:
```

## 40. Post-Remediation Validation

Do not stop simply because the Pod entered Running.

Validate:

```text
Pod scheduled?
Correct Node?
Correct node pool?
Pod Ready?
Containers healthy?
PVC attached/mounted?
Expected endpoint healthy?
Replacement workload stable?
Scheduler Events clean?
Capacity stable?
```

```text
POD SCHEDULED ≠ INCIDENT FULLY RESOLVED
SERVICE RESTORED ≠ ROOT CAUSE REMEDIATED
```

For example, manually adding a node may restore service while an autoscaling defect remains unresolved.

---

# 41. Production Decision Model

```text
IDENTIFY
   ↓
Correct cluster / namespace / workload / Pod?
   ↓
Establish blast radius
   ↓
PROVE SCHEDULING STATE
   ↓
Pending?
   ↓
Scheduling ready?
   ↓
Normal scheduler path?
   ↓
Capture Events
   ↓
Capture rendered Pod
   ↓
REDUCE CANDIDATES
   ↓
Registered
   ↓
Ready / schedulable
   ↓
Selector
   ↓
Required affinity
   ↓
Taints / tolerations
   ↓
Topology
   ↓
Storage
   ↓
Resource fit
   ↓
Feasible Nodes?
   │
   ├── YES
   │     ↓
   │  Scheduler-specific investigation
   │
   └── NO
         ↓
   Where did candidates reach zero?
         ↓
   Would correct new capacity help?
         │
         ├── NO
         │     ↓
         │  Configuration / topology /
         │  storage / placement investigation
         │
         └── YES
               ↓
             Autoscaler
               ↓
             Provisioning
               ↓
             Registration
               ↓
             Readiness
               ↓
             Effective node configuration
               ↓
             Scheduling
               ↓
CONCLUDE
   ↓
Lowest Proven Healthy Layer
   ↓
First Failed Transition
   ↓
Evidence Confidence
   ↓
Failure Domain
   ↓
Desired-State Owner
   ↓
Current Actionable Owner
   ↓
Requested Action
   ↓
Approved Remediation
   ↓
Post-Remediation Validation
```

# 42. Production Rules

```text
UNKNOWN WORKLOAD IDENTITY = NO MUTATION

POD PENDING = SYMPTOM, NOT ROOT CAUSE

POD PENDING ≠ SCHEDULER FAILURE

SCHEDULING READINESS ≠ SCHEDULING FEASIBILITY

SCHEDULER SUMMARY ≠ SINGLE ROOT CAUSE

SCHEDULER REASON ≠ UNDERLYING ROOT CAUSE

DESIRED ≠ RENDERED ≠ SCHEDULED

SOURCE MANIFEST ≠ RENDERED POD RESOURCE CONTRACT

VISIBLE POD AFFINITY ≠ COMPLETE EFFECTIVE AFFINITY
WHEN CUSTOM SCHEDULER PROFILES ARE USED

NODE EXISTS ≠ NODE READY

NODE READY ≠ NODE ELIGIBLE

NODE ELIGIBLE ≠ RESOURCE FIT

PREFERRED AFFINITY ≠ HARD PLACEMENT REQUIREMENT

TOLERATION ≠ PLACEMENT REQUIREMENT

ONE TAINT TOLERATED ≠ ALL RELEVANT TAINTS TOLERATED

PVC PENDING ≠ STORAGE FAILURE PROVEN

REQUEST ≠ USAGE

kubectl top ≠ SCHEDULER CAPACITY MODEL

LOW RUNTIME UTILIZATION ≠ SCHEDULABLE CAPACITY

AGGREGATE CLUSTER CAPACITY ≠ POD FIT

LOW CLUSTER UTILIZATION ≠ SCHEDULABLE CAPACITY EXISTS FOR THIS POD

ONE NODE'S STATE ≠ CLUSTER SCHEDULABILITY FOR THE POD

POOL SIZE ≠ READY NODE COUNT ≠ FEASIBLE NODE COUNT

SAME NODE POOL ≠ IDENTICAL EFFECTIVE NODE STATE

MORE NODES ≠ MORE FEASIBLE NODES

NODE CREATED ≠ USEFUL CAPACITY CREATED

PROVISIONED ≠ REGISTERED ≠ READY ≠ FEASIBLE

CURRENT STATE ≠ HISTORICAL INCIDENT STATE

OBSERVED DRIFT ≠ PERMISSION TO PATCH

REMEDIATION CAN DESTROY INCIDENT EVIDENCE

FIRST FAILED TRANSITION ≠ FINAL ROOT CAUSE AUTOMATICALLY

ASSUMED ≠ ROOT CAUSE

ERROR VISIBLE THROUGH TRUEFOUNDRY ≠ TRUEFOUNDRY ROOT CAUSE

SYMPTOM ≠ FAILURE DOMAIN ≠ OWNER

POD SCHEDULED ≠ INCIDENT FULLY RESOLVED

SERVICE RESTORED ≠ ROOT CAUSE REMEDIATED
```

# 43. Part 3 Integration

Part 3 forms one continuous operational model:

```text
3.1 CPU & Memory Resource Management
          ↓
3.2 Requests, Limits & Kubernetes QoS
          ↓
3.3 Node Selectors & Node Affinity
          ↓
3.4 Taints & Tolerations
          ↓
3.5 Dedicated Node Pools
          ↓
3.6 Workload Scheduling Troubleshooting
```

The operational philosophy is:

```text
DON'T GUESS THE FIX
        ↓
PRESERVE THE EVIDENCE
        ↓
PROVE THE SCHEDULING CONTRACT
        ↓
REDUCE THE CANDIDATE SET
        ↓
FIND WHERE CANDIDATES REACHED ZERO
        ↓
IDENTIFY THE FIRST FAILED TRANSITION
        ↓
IDENTIFY THE FAILURE DOMAIN
        ↓
IDENTIFY THE AUTHORITATIVE OWNER
        ↓
REMEDIATE THROUGH DESIRED STATE
        ↓
VALIDATE SERVICE RESTORATION
        ↓
VALIDATE ROOT-CAUSE REMEDIATION
```

---

## Safety Note

The diagnostic commands in this tutorial are intended to be read-only. Production changes to Pods, node labels, taints, affinity, resource requests, node pools, autoscaling, or storage configuration must follow the authoritative desired-state and approved change process.
