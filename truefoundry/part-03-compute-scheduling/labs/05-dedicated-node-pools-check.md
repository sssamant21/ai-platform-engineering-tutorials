# Part 3.5 Lab --- Dedicated Node Pools

**Lab classification:** `[SAFE-READ]`

## Purpose

This lab validates dedicated node-pool scheduling and capacity using
read-only Kubernetes evidence.

You will investigate:

-   workload and cluster identity;
-   intended node-pool identity;
-   rendered `nodeSelector` and node affinity;
-   taints and tolerations;
-   node readiness and conditions;
-   pool consistency and drift;
-   topology constraints;
-   resource requests and per-node fit;
-   scheduler Events;
-   scheduling gates;
-   feasible-node reduction;
-   autoscaling and provisioning evidence available through Kubernetes;
-   blast radius;
-   Lowest Proven Healthy Layer;
-   First Failed Transition;
-   desired-state ownership;
-   Current Actionable Owner.

The objective is not to change scheduling. The objective is to prove
where the scheduling or capacity chain first fails.

------------------------------------------------------------------------

# 1. Safety Rules

This lab is strictly read-only.

## Allowed

``` bash
kubectl config current-context
kubectl get
kubectl describe
kubectl top
```

`kubectl top` is optional and is used only for runtime utilization
context. It is not a substitute for scheduler resource accounting.

Read-only JSONPath output is also allowed.

## Do Not Run

Do not run any mutation command during this lab, including:

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

Do not manually scale a node pool, recycle a node, change labels, change
taints, or alter workload placement while collecting incident evidence.

``` text
REMEDIATION CAN DESTROY INCIDENT EVIDENCE
```

------------------------------------------------------------------------

# 2. Lab Variables

Use values appropriate for your environment.

For Windows CMD:

``` cmd
set NS=<namespace>
set POD=<pod-name>
set NODE=<node-name>
```

Optional:

``` cmd
set WORKLOAD=<workload-name>
```

For shells that do not use Windows CMD syntax, substitute the equivalent
variables or use literal values.

Do not proceed until the environment, cluster, namespace, workload, and
Pod are confirmed.

------------------------------------------------------------------------

# 3. Confirm Kubernetes Context

Run:

``` bash
kubectl config current-context
```

Record:

``` text
Environment:
Cluster / Context:
Timestamp:
```

Expected principle:

``` text
UNKNOWN CLUSTER IDENTITY
    =
NO NODE POOL CHANGE
```

If the context is unexpected, stop the investigation until the target is
confirmed.

------------------------------------------------------------------------

# 4. Confirm Namespace

List namespaces if needed:

``` bash
kubectl get namespaces
```

Verify the intended namespace:

``` bash
kubectl get namespace %NS%
```

Record:

``` text
Namespace:
```

Do not infer the namespace from a workload name.

------------------------------------------------------------------------

# 5. Confirm Pod Identity

Run:

``` bash
kubectl get pod %POD% -n %NS% -o wide
```

Then:

``` bash
kubectl get pod %POD% -n %NS% -o jsonpath="{.metadata.name}{'\n'}{.metadata.uid}{'\n'}{.metadata.creationTimestamp}{'\n'}"
```

Record:

``` text
Pod:
Pod UID:
Creation Timestamp:
Current Phase:
Current Node:
```

Production rule:

``` text
POD NAME ≠ COMPLETE POD IDENTITY
```

A recreated Pod can have the same name pattern but a different UID.

------------------------------------------------------------------------

# 6. Identify the Workload Owner

Run:

``` bash
kubectl get pod %POD% -n %NS% -o jsonpath="{range .metadata.ownerReferences[*]}{.kind}{'\t'}{.name}{'\t'}{.uid}{'\n'}{end}"
```

Record:

``` text
Immediate Owner Kind:
Immediate Owner Name:
Immediate Owner UID:
```

If the immediate owner is a ReplicaSet, inspect it read-only:

``` bash
kubectl get rs <replicaset-name> -n %NS% -o yaml
```

If necessary, follow its owner reference to the Deployment.

The purpose is to identify the authoritative workload chain, not to
modify it.

------------------------------------------------------------------------

# 7. Capture Pod Scheduler Events First

Run:

``` bash
kubectl describe pod %POD% -n %NS%
```

Focus on the Events section.

Record exact scheduler evidence:

``` text
Scheduler Event:
Reason:
Message:
First Seen:
Last Seen:
Count:
```

Look for evidence involving:

``` text
node affinity / selector mismatch
untolerated taints
insufficient CPU
insufficient memory
volume / storage constraints
topology constraints
scheduling gates
other scheduling restrictions
```

Do not translate every Pending Pod into a capacity problem.

``` text
POD PENDING
    ≠
NODE POOL CAPACITY FAILURE PROVEN
```

------------------------------------------------------------------------

# 8. Check Scheduler Name

Run:

``` bash
kubectl get pod %POD% -n %NS% -o jsonpath="{.spec.schedulerName}{'\n'}"
```

Record:

``` text
Scheduler Name:
```

If the value is not the expected scheduler, record that before
interpreting scheduler behavior.

------------------------------------------------------------------------

# 9. Check Scheduling Gates

Run:

``` bash
kubectl get pod %POD% -n %NS% -o jsonpath="{range .spec.schedulingGates[*]}{.name}{'\n'}{end}"
```

Record:

``` text
Scheduling Gates:
```

If no values are returned, record:

``` text
Scheduling Gates: none observed
```

Rule:

``` text
POD PENDING
    ≠
SCHEDULER IS CURRENTLY TRYING TO PLACE IT
```

A scheduling gate can intentionally delay normal scheduling.

------------------------------------------------------------------------

# 10. Capture Rendered nodeSelector

Run:

``` bash
kubectl get pod %POD% -n %NS% -o jsonpath="{.spec.nodeSelector}{'\n'}"
```

Record:

``` text
Rendered nodeSelector:
```

Do not use the intended deployment configuration as a substitute for the
rendered Pod.

``` text
DESIRED ≠ RENDERED ≠ SCHEDULED
```

------------------------------------------------------------------------

# 11. Capture Rendered Node Affinity

Run:

``` bash
kubectl get pod %POD% -n %NS% -o jsonpath="{.spec.affinity.nodeAffinity}{'\n'}"
```

For complete structure when needed:

``` bash
kubectl get pod %POD% -n %NS% -o yaml
```

Record:

``` text
Required Node Affinity:
Preferred Node Affinity:
```

Determine whether the intended dedicated pool is actually required or
merely preferred.

------------------------------------------------------------------------

# 12. Capture Rendered Tolerations

Run:

``` bash
kubectl get pod %POD% -n %NS% -o jsonpath="{range .spec.tolerations[*]}{.key}{'\t'}{.operator}{'\t'}{.value}{'\t'}{.effect}{'\t'}{.tolerationSeconds}{'\n'}{end}"
```

Record:

``` text
Rendered Tolerations:
```

Remember:

``` text
TOLERATION ≠ NODE POOL TARGETING
```

------------------------------------------------------------------------

# 13. Capture Resource Requests and Limits

Run:

``` bash
kubectl get pod %POD% -n %NS% -o jsonpath="{range .spec.containers[*]}{.name}{'\n'}{.resources.requests}{'\n'}{.resources.limits}{'\n'}{end}"
```

Record per container:

``` text
Container:
CPU Request:
Memory Request:
Other Requests:
CPU Limit:
Memory Limit:
Other Limits:
```

For scheduling analysis, requests are critical.

``` text
REQUEST ≠ USAGE
```

------------------------------------------------------------------------

# 14. Check Direct Node Assignment

Run:

``` bash
kubectl get pod %POD% -n %NS% -o jsonpath="{.spec.nodeName}{'\n'}"
```

Record:

``` text
nodeName:
```

If a value exists before normal scheduler placement analysis, note that
direct node assignment can change how the Pod reached the Node.

Do not assume every running Pod was selected through the normal
scheduler path.

------------------------------------------------------------------------

# 15. Define the Intended Node Pool

Based on approved architecture or platform configuration, record:

``` text
Intended Node Pool:
Expected Pool Label Key:
Expected Pool Label Value:
Expected Pool Taints:
Expected Instance Type:
Expected Zones:
Expected Minimum Size:
Expected Maximum Size:
Autoscaling Expected:
Expected Provisioner / Autoscaler:
```

Do not invent missing values.

Use:

``` text
UNKNOWN
```

when authoritative desired-state information is unavailable.

------------------------------------------------------------------------

# 16. Inventory Kubernetes Nodes

Run:

``` bash
kubectl get nodes -o wide
```

Record:

``` text
Total Registered Nodes:
Ready Nodes:
NotReady Nodes:
```

This establishes the cluster-level node inventory.

------------------------------------------------------------------------

# 17. Display Node Labels

Run:

``` bash
kubectl get nodes --show-labels
```

If you know the expected pool label, use a read-only selector:

``` bash
kubectl get nodes -l <label-key>=<label-value> -o wide
```

Example only:

``` bash
kubectl get nodes -l node-pool=ai -o wide
```

Record:

``` text
Nodes Matching Expected Pool Label:
```

Do not assume `node-pool` is a universal Kubernetes-standard label. Use
the actual label defined by your environment.

------------------------------------------------------------------------

# 18. Prove Pool Membership

For each candidate Node, capture the evidence that identifies it as
belonging to the intended infrastructure pool.

Possible evidence can include:

``` text
provider-specific node-group label
custom pool label
instance type
zone
node name
provider ID
infrastructure metadata
```

Read ProviderID:

``` bash
kubectl get node %NODE% -o jsonpath="{.spec.providerID}{'\n'}"
```

Record:

``` text
Pool Membership Evidence:
Confidence:
```

Use:

``` text
PROVEN
SUPPORTED
UNKNOWN
ASSUMED
```

------------------------------------------------------------------------

# 19. Inspect Node Labels Precisely

Run:

``` bash
kubectl get node %NODE% -o jsonpath="{.metadata.labels}{'\n'}"
```

Or:

``` bash
kubectl describe node %NODE%
```

Record relevant labels:

``` text
Pool Label:
Instance Type:
Architecture:
Operating System:
Zone:
Region:
Other Placement Labels:
```

Compare these values against the workload's required selector and
affinity.

------------------------------------------------------------------------

# 20. Inspect Node Taints

Run:

``` bash
kubectl get node %NODE% -o jsonpath="{range .spec.taints[*]}{.key}{'='}{.value}{':' }{.effect}{'\n'}{end}"
```

If your client does not render the JSONPath expression as expected, use:

``` bash
kubectl describe node %NODE%
```

Record:

``` text
Node Taints:
```

Compare every relevant taint against the Pod's tolerations.

``` text
ONE TAINT TOLERATED
    ≠
ALL RELEVANT TAINTS TOLERATED
```

------------------------------------------------------------------------

# 21. Inspect Node Readiness

Run:

``` bash
kubectl get node %NODE%
```

Then:

``` bash
kubectl get node %NODE% -o jsonpath="{range .status.conditions[*]}{.type}{'\t'}{.status}{'\t'}{.reason}{'\t'}{.lastTransitionTime}{'\n'}{end}"
```

Record:

``` text
Ready:
MemoryPressure:
DiskPressure:
PIDPressure:
NetworkUnavailable:
```

Rule:

``` text
NODE REGISTERED ≠ NODE READY
```

------------------------------------------------------------------------

# 22. Check Unschedulable State

Run:

``` bash
kubectl get node %NODE% -o jsonpath="{.spec.unschedulable}{'\n'}"
```

Record:

``` text
spec.unschedulable:
```

If the field is empty or false, do not automatically conclude that the
Node is feasible. All other constraints still apply.

------------------------------------------------------------------------

# 23. Inspect Node Capacity and Allocatable

Run:

``` bash
kubectl get node %NODE% -o jsonpath="{.status.capacity}{'\n'}{.status.allocatable}{'\n'}"
```

Or:

``` bash
kubectl describe node %NODE%
```

Record:

``` text
CPU Capacity:
CPU Allocatable:
Memory Capacity:
Memory Allocatable:
Other Capacity:
Other Allocatable:
```

Remember:

``` text
NODE CAPACITY ≠ NODE ALLOCATABLE
```

------------------------------------------------------------------------

# 24. Inspect Requested Resource Accounting

Run:

``` bash
kubectl describe node %NODE%
```

Review the `Allocated resources` section.

Record:

``` text
CPU Requests:
CPU Limits:
Memory Requests:
Memory Limits:
Other Requested Resources:
```

Compare scheduler accounting against the target Pod's requests.

Do not substitute runtime utilization for this evidence.

------------------------------------------------------------------------

# 25. Optional Runtime Utilization

If Metrics Server or another supported metrics API is available:

``` bash
kubectl top node %NODE%
```

Optionally:

``` bash
kubectl top nodes
```

Use this only as operational context.

``` text
kubectl top ≠ SCHEDULER CAPACITY MODEL
```

A Node can have low current CPU usage while scheduler-request accounting
leaves insufficient room for the Pod.

------------------------------------------------------------------------

# 26. Build the Pool Node Inventory

For each Node expected to belong to the pool, record:

  ----------------------------------------------------------------------------------------------------
  Node         Registered   Ready   Unschedulable   Pool    Taint        Resource   Zone    Feasible
                                                    Label   Compatible   Fit                
                                                    Match                                   
  ------------ ------------ ------- --------------- ------- ------------ ---------- ------- ----------
  `<node-1>`                                                                                

  `<node-2>`                                                                                

  `<node-3>`                                                                                
  ----------------------------------------------------------------------------------------------------

Do not fill unknown fields from assumption.

------------------------------------------------------------------------

# 27. Compare Pool Nodes for Drift

Compare expected-equivalent Nodes across:

``` text
Labels
Taints
Readiness
Conditions
Unschedulable state
Instance type
CPU
Memory
Accelerators
Allocatable resources
Zone
Kubernetes version
Container runtime
```

Useful read-only overview:

``` bash
kubectl get nodes -o wide
```

Then inspect individual Nodes as needed.

Record:

``` text
Observed Drift:
Affected Nodes:
Evidence:
```

Rule:

``` text
SAME NODE POOL
    ≠
IDENTICAL EFFECTIVE NODE STATE
```

------------------------------------------------------------------------

# 28. Label Drift Analysis

If a Node has an unexpected label:

``` text
Expected:
node-pool=ai

Observed:
node-pool=general
```

Record:

``` text
Label Drift: PROVEN / SUPPORTED / UNKNOWN
```

Do not run `kubectl label`.

``` text
LABEL MISMATCH PROVEN
    ≠
MANUAL LABEL PATCH AUTHORIZED
```

Identify the authoritative label owner instead.

------------------------------------------------------------------------

# 29. Taint Drift Analysis

If Nodes expected to be equivalent have different taints, record the
exact difference.

Example:

``` text
Node A → dedicated=ai:NoSchedule
Node B → dedicated=ai:NoSchedule
Node C → no dedicated taint
```

Ask:

``` text
Can unintended workloads become eligible for Node C?
Can intended workloads use all expected Nodes?
```

Do not run `kubectl taint`.

``` text
TAINT DIFFERENCE ≠ MANUAL FIX AUTHORIZED
```

------------------------------------------------------------------------

# 30. Validate Dedicated-Pool Protection

Determine whether intended workloads are attracted/required and
unintended workloads are repelled.

Record:

``` text
Pool Label:
Workload Required Placement:
Pool Taint:
Workload Toleration:
```

Evaluate:

``` text
Intended workload can enter: PROVEN / SUPPORTED / UNKNOWN
Unintended workloads excluded: PROVEN / SUPPORTED / UNKNOWN
```

Rule:

``` text
DEDICATED POOL HEALTH
    ≠
ONLY "CAN MY POD SCHEDULE?"
```

------------------------------------------------------------------------

# 31. Validate Toleration Is Not Being Misread as Targeting

If the Pod has the dedicated-pool toleration but no required selector or
affinity, record:

``` text
Pod tolerates dedicated capacity: YES
Pod requires dedicated capacity: NO / UNKNOWN
```

Rule:

``` text
TOLERATION ≠ NODE POOL TARGETING
```

------------------------------------------------------------------------

# 32. Validate Targeting Is Not Being Misread as Exclusivity

If the workload requires the pool but the Nodes have no repelling taint
or equivalent policy, record:

``` text
Intended workload targets pool: YES
Pool protected from unintended workloads: NOT PROVEN
```

Rule:

``` text
WORKLOAD TARGETS DEDICATED POOL
    ≠
OTHER WORKLOADS ARE EXCLUDED
```

------------------------------------------------------------------------

# 33. Build the Candidate-Node Funnel

Count Nodes through each stage.

Record:

``` text
All Registered Nodes:
Ready / Schedulable Nodes:
Pool-Compatible Nodes:
nodeSelector-Compatible Nodes:
Required-Affinity-Compatible Nodes:
Taint-Compatible Nodes:
Topology / Storage-Compatible Nodes:
Resource-Fit Nodes:
Final Feasible Nodes:
```

The central diagnostic question is:

``` text
WHERE DID THE CANDIDATE SET BECOME ZERO?
```

------------------------------------------------------------------------

# 34. Placement-Compatible Node Count

Using the rendered selector and required affinity, determine which Nodes
can satisfy the workload's hard placement rules.

For a simple known label selector, a read-only command may be:

``` bash
kubectl get nodes -l <label-key>=<label-value> -o wide
```

For complex affinity expressions, inspect labels and evaluate the
expressions carefully.

Record:

``` text
Placement-Compatible Nodes:
```

Do not treat preferred affinity as a hard exclusion.

------------------------------------------------------------------------

# 35. Taint-Compatible Node Count

For every placement-compatible Node:

1.  list all taints;
2.  list all Pod tolerations;
3.  evaluate each relevant taint;
4.  determine whether a remaining taint excludes scheduling.

Record:

``` text
Placement-Compatible Nodes:
Taint-Compatible Nodes:
Excluded by Untolerated Taint:
```

Rule:

``` text
TOLERATION MATCH ≠ COMPLETE NODE ELIGIBILITY
```

------------------------------------------------------------------------

# 36. Topology Constraints

Inspect the Pod YAML for topology-related scheduling constraints:

``` bash
kubectl get pod %POD% -n %NS% -o yaml
```

Review relevant fields such as:

``` text
topologySpreadConstraints
podAffinity
podAntiAffinity
nodeAffinity
```

Record:

``` text
Topology Restrictions:
Effective Zones / Domains:
Nodes Excluded by Topology:
```

Rule:

``` text
POOL HAS N READY NODES
    ≠
ALL N NODES ARE VALID FOR THIS POD
```

------------------------------------------------------------------------

# 37. Storage Constraints

If the Pod uses persistent storage, list claims:

``` bash
kubectl get pod %POD% -n %NS% -o jsonpath="{range .spec.volumes[*]}{.name}{'\t'}{.persistentVolumeClaim.claimName}{'\n'}{end}"
```

For a referenced PVC:

``` bash
kubectl get pvc <pvc-name> -n %NS% -o wide
```

Inspect if necessary:

``` bash
kubectl describe pvc <pvc-name> -n %NS%
```

If a PV is bound:

``` bash
kubectl get pv <pv-name> -o yaml
```

Record:

``` text
PVC:
PV:
StorageClass:
Relevant Node / Zone Affinity:
Storage Constraint Proven:
```

Do not modify the PVC, PV, or StorageClass.

------------------------------------------------------------------------

# 38. Resource-Fit Analysis

For each Node that survives placement, taint, topology, and storage
checks:

1.  record Pod resource requests;
2.  record Node allocatable resources;
3.  inspect current requested-resource accounting;
4.  determine whether the Pod fits on that individual Node.

Record:

  --------------------------------------------------------------------------
  Node           CPU Fit        Memory Fit     Other Resource Overall
                                               Fit            Resource Fit
  -------------- -------------- -------------- -------------- --------------
  `<node-1>`                                                  

  `<node-2>`                                                  
  --------------------------------------------------------------------------

Rule:

``` text
AGGREGATE POOL CAPACITY ≠ POD FIT
```

------------------------------------------------------------------------

# 39. Detect Capacity Fragmentation

If the pool has resources in aggregate but no individual Node satisfies
all requirements, record:

``` text
Aggregate Pool Capacity Appears Available: YES
Single-Node Fit Exists: NO
Capacity Fragmentation: PROVEN / SUPPORTED
```

Rule:

``` text
LOW POOL UTILIZATION
    ≠
CAPACITY EXISTS FOR THIS POD
```

------------------------------------------------------------------------

# 40. Separate Pool Size From Feasible Capacity

Complete:

``` text
Expected Pool Size:
Registered Nodes:
Ready Nodes:
Placement-Compatible Nodes:
Taint-Compatible Nodes:
Resource-Fit Nodes:
Final Feasible Nodes:
```

Then state:

``` text
First count transition that became unexpectedly smaller:
```

This is more useful than reporting only the configured node-pool size.

------------------------------------------------------------------------

# 41. Inspect Node Events

For a suspicious Node:

``` bash
kubectl describe node %NODE%
```

Review Events and conditions.

Record:

``` text
Node Event:
Reason:
Message:
Timestamp:
```

Look for evidence around readiness, pressure, networking, registration,
runtime, or lifecycle transitions.

Do not infer root cause solely from a condition name.

------------------------------------------------------------------------

# 42. Condition-Related Taints

Record any well-known condition-related taints observed on the Node.

Examples can include:

``` text
node.kubernetes.io/not-ready
node.kubernetes.io/unreachable
node.kubernetes.io/memory-pressure
node.kubernetes.io/disk-pressure
node.kubernetes.io/pid-pressure
node.kubernetes.io/network-unavailable
```

Then distinguish:

``` text
Observed Taint:
Underlying Node Condition:
Underlying Cause Proven:
```

Rule:

``` text
TAINT MAY BE A MECHANISM / CONSEQUENCE
    ≠
UNDERLYING ROOT CAUSE
```

Do not remove a health-related taint simply to make the Pod schedulable.

------------------------------------------------------------------------

# 43. Check Cluster Events for Timeline Evidence

Read recent namespace Events:

``` bash
kubectl get events -n %NS% --sort-by=.metadata.creationTimestamp
```

For cluster-scoped node-related Events where access permits:

``` bash
kubectl get events -A --sort-by=.metadata.creationTimestamp
```

Use this as timeline evidence.

Record:

``` text
Pod Pending Time:
Relevant Scheduler Event Time:
Relevant Node Event Time:
Relevant Provisioning Event Time:
```

Event retention is finite. Absence of an old Event does not prove that
an event never occurred.

------------------------------------------------------------------------

# 44. Autoscaling Evidence Available Through Kubernetes

First determine whether autoscaling/provisioning components are visible
in the cluster.

Read-only examples:

``` bash
kubectl get pods -A
```

Look for components relevant to your environment, such as a cluster
autoscaler or node provisioning controller.

Do not assume a specific implementation.

Record:

``` text
Autoscaling Expected:
Autoscaling Component Observed:
Provisioner Observed:
```

------------------------------------------------------------------------

# 45. Inspect Autoscaler / Provisioner Logs Only If Authorized

If the relevant component and namespace are known and log access is
permitted, use read-only logs:

``` bash
kubectl logs <pod> -n <namespace>
```

For a multi-container Pod:

``` bash
kubectl logs <pod> -n <namespace> -c <container>
```

Search manually for evidence tied to:

``` text
Pod name
Pod UID
node pool
provisioning request
unschedulable reason
instance type
quota
capacity
constraint
```

Record exact evidence and timestamps.

Do not restart the component.

------------------------------------------------------------------------

# 46. Autoscaler Evidence Questions

Answer:

``` text
Did the autoscaler observe the unschedulable Pod?
Did it consider a compatible pool / node configuration?
Did it decide to scale?
Was scaling blocked?
Was provisioning requested?
Was a Node created?
```

Use:

``` text
PROVEN
SUPPORTED
UNKNOWN
ASSUMED
```

Do not convert missing log access into a conclusion.

------------------------------------------------------------------------

# 47. Autoscaler Enabled Is Not Enough

Record:

``` text
Autoscaling Enabled:
Minimum Size:
Maximum Size:
Current Size:
Provisioning Limits:
```

If those values are not available through your authorized read-only
sources, mark them `UNKNOWN`.

Rule:

``` text
AUTOSCALER ENABLED
    ≠
THIS POD CAN TRIGGER USEFUL CAPACITY
```

------------------------------------------------------------------------

# 48. Would Another Correctly Configured Node Help?

Before recommending scaling, answer:

``` text
Would a new Node with the intended:
  labels,
  taints,
  topology,
  resources,
  storage compatibility

make the Pod schedulable?
```

Record:

``` text
YES / NO / UNKNOWN
Evidence:
```

If the answer is `NO`, scaling the same incompatible capacity is not a
supported remediation.

``` text
MORE NODES ≠ MORE FEASIBLE NODES
```

------------------------------------------------------------------------

# 49. Scale-to-Zero Investigation

If the intended pool currently has zero registered Nodes, record:

``` text
Zero Nodes Expected:
Scale-to-Zero Supported:
Pending Compatible Workload Exists:
Demand Detected:
Provisioning Attempted:
Node Created:
Node Registered:
Node Ready:
```

Unknown values must remain `UNKNOWN`.

Rule:

``` text
ZERO NODES ≠ NODE POOL FAILURE PROVEN
```

------------------------------------------------------------------------

# 50. Provisioned but Not Registered

If external evidence indicates infrastructure was created but no
Kubernetes Node appeared, record:

``` text
Infrastructure Created:
Kubernetes Node Registered:
```

Potential failure boundary:

``` text
PROVISIONED INFRASTRUCTURE
        ↓
KUBERNETES REGISTRATION
```

Do not call this a scheduler failure.

------------------------------------------------------------------------

# 51. Registered but Not Ready

If the Node exists but is NotReady:

``` bash
kubectl describe node %NODE%
```

Capture:

``` text
Ready Condition:
Condition Reason:
Condition Message:
Taints:
Events:
```

Potential failure boundary:

``` text
NODE REGISTERED
        ↓
NODE READY
```

Possible investigation domains include kubelet, runtime, networking,
CNI, disk, memory, certificates, bootstrap, or cloud integration.

Do not select one without evidence.

------------------------------------------------------------------------

# 52. Ready but Placement-Incompatible

If the Node is Ready but the Pod cannot use it, capture the first proven
mismatch:

``` text
Missing required label
Required affinity mismatch
Untolerated taint
Topology mismatch
Storage constraint
```

Potential failure boundary:

``` text
READY NODE
    ↓
PLACEMENT COMPATIBILITY
```

------------------------------------------------------------------------

# 53. Placement-Compatible but Resource-Incompatible

If placement and taint compatibility pass but the Pod does not fit:

``` text
PLACEMENT COMPATIBLE
        ↓
RESOURCE FIT
```

Record the exact resource that fails.

Example:

``` text
CPU Fit: PASS
Memory Fit: FAIL
Other Resource Fit: PASS
```

Do not report generic "capacity issue" when the failing resource is
known.

------------------------------------------------------------------------

# 54. Build a Timeline

Record, where evidence exists:

``` text
Pod Created:
Pod Became Pending:
Scheduler Failure Event:
Autoscaler Detected Demand:
Provisioning Requested:
Infrastructure Created:
Node Registered:
Node Ready:
Pod Scheduled:
```

Then classify delay:

``` text
Detection:
Provisioning:
Bootstrap:
Registration:
Readiness:
Scheduling:
```

Unknown timestamps remain `UNKNOWN`.

------------------------------------------------------------------------

# 55. Preserve Autoscaler Evidence Before Manual Scaling

If an incident responder proposes manual scale-up, first ensure that
current evidence has been captured.

Record:

``` text
Scheduler Events Captured:
Current Pool State Captured:
Autoscaler Evidence Captured:
Provisioning Evidence Captured:
Timeline Captured:
```

Rule:

``` text
MANUAL SCALE-UP
    CAN DESTROY AUTOSCALER FAILURE EVIDENCE
```

This lab does not perform the scale-up.

------------------------------------------------------------------------

# 56. Determine Minimum Supported Blast Radius

Classify only what the evidence supports:

``` text
One Pod
One workload
One Node
One node pool
One availability zone
One instance type
Multiple node pools
Cluster-wide
```

Record:

``` text
Minimum Supported Blast Radius:
Evidence:
```

Rule:

``` text
ONE DEDICATED POOL IMPACTED
    ≠
CLUSTER-WIDE SCHEDULING FAILURE
```

------------------------------------------------------------------------

# 57. Determine Lowest Proven Healthy Layer

Use the investigation chain:

``` text
Desired workload
        ↓
Rendered Pod
        ↓
Scheduling readiness
        ↓
Scheduler evaluation
        ↓
Placement compatibility
        ↓
Taint compatibility
        ↓
Topology / storage compatibility
        ↓
Resource fit
        ↓
Autoscaler decision
        ↓
Provisioning
        ↓
Registration
        ↓
Readiness
```

Record the lowest layer proven healthy.

Example:

``` text
Lowest Proven Healthy Layer:
Dedicated-pool placement and taint compatibility.
```

------------------------------------------------------------------------

# 58. Determine First Failed Transition

Record the first transition where evidence shows failure.

Examples:

``` text
Ready Nodes → placement compatibility

Placement compatibility → taint compatibility

Taint compatibility → resource fit

Autoscaler decision → provisioning request

Provisioning request → infrastructure creation

Infrastructure creation → Kubernetes registration

Node registration → Node Ready
```

Record:

``` text
First Failed Transition:
Evidence:
```

------------------------------------------------------------------------

# 59. Evidence Confidence

For every major conclusion use:

``` text
PROVEN
SUPPORTED
UNKNOWN
ASSUMED
```

Example:

``` text
PROVEN
Pod requires node-pool=ai.

PROVEN
No Ready Node currently matches that requirement.

SUPPORTED
Node provisioning failed during the incident window.

UNKNOWN
Why provisioning failed.

ASSUMED
TrueFoundry caused the failure.
```

Rule:

``` text
ASSUMED ≠ ROOT CAUSE
```

------------------------------------------------------------------------

# 60. Determine Desired-State Ownership

Record ownership independently:

``` text
Pool Existence Owner:
Minimum Size Owner:
Maximum Size Owner:
Desired Size Owner:
Instance Type Owner:
Availability Zone Owner:
Node Label Owner:
Node Taint Owner:
Autoscaling Policy Owner:
Bootstrap Configuration Owner:
Workload nodeSelector Owner:
Workload Affinity Owner:
Workload Toleration Owner:
Resource Request Owner:
```

Potential systems or teams can include:

``` text
TrueFoundry
Terraform
GitOps
Managed Kubernetes
Cloud infrastructure
Kubernetes platform
Autoscaler / provisioner
Application configuration
```

Do not assume one owner controls every field.

------------------------------------------------------------------------

# 61. Determine Current Actionable Owner

Record:

``` text
Symptom:
Failure Domain:
First Failed Transition:
Current Actionable Owner:
Requested Action:
```

Example:

``` text
Symptom:
Pod Pending

First Failed Transition:
Provisioning request → infrastructure creation

Failure Domain:
Cloud infrastructure capacity

Current Actionable Owner:
Cloud infrastructure owner

Requested Action:
Validate why the requested node capacity could not be provisioned.
```

Rule:

``` text
SYMPTOM ≠ FAILURE DOMAIN ≠ OWNER
```

------------------------------------------------------------------------

# 62. TrueFoundry Context

For a TrueFoundry-managed workload, distinguish:

``` text
TrueFoundry Deployment Intent
        ↓
Rendered Kubernetes Pod
        ↓
Kubernetes Scheduler
        ↓
Dedicated Capacity
        ↓
Autoscaling / Provisioning
        ↓
Kubernetes Nodes
```

If TrueFoundry displays the Pending or deployment failure, record that
as an observation.

Do not automatically classify TrueFoundry as the failure domain.

``` text
ERROR VISIBLE THROUGH TRUEFOUNDRY
    ≠
TRUEFOUNDRY ROOT CAUSE
```

------------------------------------------------------------------------

# 63. Final Evidence Handoff

Complete:

``` text
Environment:
Cluster:
Namespace:

Workload:
Workload Owner:
Pod:
Pod UID:
Timestamp:

Symptom:
Blast Radius:

Intended Node Pool:

Rendered nodeSelector:
Rendered Required Affinity:
Rendered Tolerations:
Rendered Resource Requests:
Scheduling Gates:
Scheduler Name:

Scheduler Events:

Expected Pool Labels:
Expected Pool Taints:
Expected Instance Type:
Expected Zones:

Expected Pool Size:
Provisioned Infrastructure:
Registered Pool Nodes:
Ready Pool Nodes:
Placement-Compatible Nodes:
Taint-Compatible Nodes:
Topology / Storage-Compatible Nodes:
Resource-Fit Nodes:
Final Feasible Nodes:

Observed Node Labels:
Observed Node Taints:
Observed Node Conditions:

Autoscaling Enabled:
Minimum Size:
Maximum Size:
Current Size:

Autoscaler Evidence:
Provisioning Evidence:

Pod Pending Time:
Scale-Up Detection Time:
Provisioning Request Time:
Infrastructure Creation Time:
Node Registration Time:
Node Ready Time:
Pod Scheduling Time:

Lowest Proven Healthy Layer:
First Failed Transition:

Evidence Confidence:

Authoritative Desired-State Owner:
Current Actionable Owner:

Requested Action:
```

------------------------------------------------------------------------

# 64. Lab Decision Model

Use this final flow:

``` text
IDENTITY
    ↓
INTENDED NODE POOL
    ↓
SCHEDULING READY?
    ↓
SCHEDULER EVENTS
    ↓
RENDERED PLACEMENT POLICY
    ↓
REGISTERED NODES
    ↓
READY NODES
    ↓
POOL / LABEL MATCH
    ↓
TAINT / TOLERATION MATCH
    ↓
TOPOLOGY / STORAGE
    ↓
RESOURCE FIT
    ↓
FEASIBLE NODE EXISTS?
    │
    ├── YES
    │     ↓
    │  Continue scheduler-specific investigation
    │
    └── NO
          ↓
    WOULD CORRECT NEW CAPACITY HELP?
          ↓
    AUTOSCALER / PROVISIONER EVIDENCE
          ↓
    PROVISIONING
          ↓
    REGISTRATION
          ↓
    READINESS
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
```

------------------------------------------------------------------------

# 65. Lab Acceptance Criteria

The lab is complete when you can answer all of the following without
mutating the environment:

-   [ ] Correct cluster/context confirmed.
-   [ ] Namespace confirmed.
-   [ ] Pod name and UID confirmed.
-   [ ] Workload owner identified.
-   [ ] Intended node pool identified or explicitly marked `UNKNOWN`.
-   [ ] Scheduler Events captured.
-   [ ] Scheduler name captured.
-   [ ] Scheduling gates checked.
-   [ ] Rendered `nodeSelector` captured.
-   [ ] Required node affinity captured.
-   [ ] Tolerations captured.
-   [ ] Resource requests captured.
-   [ ] Pool Nodes inventoried.
-   [ ] Pool membership evidence recorded.
-   [ ] Node labels compared.
-   [ ] Node taints compared.
-   [ ] Node readiness and conditions checked.
-   [ ] Unschedulable state checked.
-   [ ] Allocatable resources captured.
-   [ ] Requested-resource accounting reviewed.
-   [ ] Pool drift evaluated.
-   [ ] Candidate-node funnel completed.
-   [ ] Placement-compatible Node count established.
-   [ ] Taint-compatible Node count established.
-   [ ] Topology constraints evaluated.
-   [ ] Storage constraints evaluated when relevant.
-   [ ] Resource-fit Node count established.
-   [ ] Final feasible Node count established.
-   [ ] Autoscaler/provisioner evidence reviewed where available.
-   [ ] Scale-to-zero considered where relevant.
-   [ ] Timeline established as far as evidence permits.
-   [ ] Minimum Supported Blast Radius recorded.
-   [ ] Lowest Proven Healthy Layer recorded.
-   [ ] First Failed Transition recorded.
-   [ ] Evidence confidence recorded.
-   [ ] Authoritative desired-state owner identified or marked
    `UNKNOWN`.
-   [ ] Current Actionable Owner identified or marked `UNKNOWN`.
-   [ ] Requested Action stated.
-   [ ] No production mutation performed.

------------------------------------------------------------------------

# 66. Final Production Rules

``` text
UNKNOWN WORKLOAD IDENTITY = NO NODE POOL CHANGE

NODE POOL EXISTS ≠ WORKLOAD CAN SCHEDULE THERE

NODE HAS POOL LABEL ≠ WORKLOAD TARGETS THAT LABEL

TOLERATION ≠ NODE POOL TARGETING

WORKLOAD TARGETS DEDICATED POOL
    ≠
OTHER WORKLOADS ARE EXCLUDED

DEDICATED NODE POOL
    ≠
COMPLETE SECURITY BOUNDARY

POOL SIZE
    ≠
READY NODE COUNT
    ≠
FEASIBLE NODE COUNT

NODE REGISTERED ≠ NODE READY

NODE READY ≠ NODE FEASIBLE FOR THIS POD

AGGREGATE POOL CAPACITY ≠ POD FIT

LOW POOL UTILIZATION
    ≠
CAPACITY EXISTS FOR THIS POD

REQUEST ≠ USAGE

kubectl top ≠ SCHEDULER CAPACITY MODEL

POD PENDING
    ≠
NODE POOL CAPACITY FAILURE PROVEN

POD PENDING
    ≠
SCHEDULER IS CURRENTLY TRYING TO PLACE IT

AUTOSCALER ENABLED
    ≠
THIS POD CAN TRIGGER USEFUL CAPACITY

UNSCHEDULABLE POD
    ≠
AUTOSCALER WILL NECESSARILY ADD A NODE

MORE NODES ≠ MORE FEASIBLE NODES

ZERO NODES ≠ NODE POOL FAILURE PROVEN

SCHEDULING SYMPTOM ≠ SCHEDULER FAILURE

LABEL MISMATCH PROVEN
    ≠
MANUAL LABEL PATCH AUTHORIZED

TAINT DIFFERENCE
    ≠
MANUAL FIX AUTHORIZED

OBSERVED DRIFT ≠ PERMISSION TO PATCH

NODE POOL CHANGE ≠ LOCAL POD FIX

MANUAL SCALE-UP
    CAN DESTROY AUTOSCALER FAILURE EVIDENCE

REMEDIATION CAN DESTROY INCIDENT EVIDENCE

ASSUMED ≠ ROOT CAUSE

SYMPTOM ≠ FAILURE DOMAIN ≠ OWNER
```

------------------------------------------------------------------------

## Result

This lab provides a read-only method for proving whether a
dedicated-node-pool incident originates from workload placement, taint
compatibility, topology, storage, resource fit, node health,
autoscaling, infrastructure provisioning, registration, or readiness.

No mutation is required to establish the first supported failure
boundary.
