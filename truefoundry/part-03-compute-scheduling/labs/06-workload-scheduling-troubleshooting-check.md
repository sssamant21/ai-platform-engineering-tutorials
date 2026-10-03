# Part 3.6 Hands-On Lab --- Workload Scheduling Troubleshooting

> **Safety Classification:** `[SAFE-READ]`
>
> This lab is designed for production-safe evidence collection. It uses
> read-only Kubernetes inspection commands and does **not** instruct you
> to modify Pods, Nodes, labels, taints, resource requests, affinity,
> replicas, node pools, or autoscaling configuration.

## 1. Lab Objective

Use a repeatable SRE methodology to determine why a Kubernetes Pod
cannot obtain valid placement.

The lab follows five phases:

``` text
IDENTIFY
   ↓
PROVE SCHEDULING STATE
   ↓
REDUCE CANDIDATE NODES
   ↓
INVESTIGATE CAPACITY PATH
   ↓
CONCLUDE, HAND OFF & VALIDATE
```

The central question is:

``` text
WHY DOES THIS POD CURRENTLY HAVE NO VALID PLACEMENT?
```

Do not begin by guessing a remediation.

``` text
POD PENDING = SYMPTOM
NOT ROOT CAUSE
```

------------------------------------------------------------------------

## 2. Safety Rules

Allowed examples in this lab include:

``` bash
kubectl config current-context
kubectl get
kubectl describe
kubectl logs
kubectl top
```

`kubectl top` is optional and only provides runtime-utilization context.

Do **not** use this lab to perform unapproved production mutations such
as:

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

Do not change:

``` text
requests / limits
nodeSelector
affinity
tolerations
replicas
labels
taints
node-pool configuration
autoscaling configuration
```

Production rule:

``` text
UNKNOWN WORKLOAD IDENTITY = NO MUTATION
```

------------------------------------------------------------------------

# Phase 1 --- Identify

## 3. Confirm Kubernetes Context

Run:

``` bash
kubectl config current-context
```

Record:

``` text
Expected environment:
Expected cluster:
Observed context:
Match confirmed: YES / NO
```

If the context is unexpected, stop the investigation until identity is
resolved.

------------------------------------------------------------------------

## 4. Confirm Namespace

``` bash
kubectl get namespace <namespace>
```

Record:

``` text
Namespace:
Environment:
Namespace exists: YES / NO
```

------------------------------------------------------------------------

## 5. Capture Pod Identity

``` bash
kubectl get pod <pod> -n <namespace> -o wide
```

Then:

``` bash
kubectl get pod <pod> -n <namespace> \
  -o jsonpath="{.metadata.name}{'\n'}{.metadata.uid}{'\n'}{.metadata.creationTimestamp}{'\n'}"
```

Record:

``` text
Pod:
Pod UID:
Creation timestamp:
Current phase:
Assigned node:
Pod IP:
```

The UID matters because a recreated Pod with the same name is a
different Kubernetes object.

------------------------------------------------------------------------

## 6. Identify Workload Owner

``` bash
kubectl get pod <pod> -n <namespace> \
  -o jsonpath="{range .metadata.ownerReferences[*]}{.kind}{'\t'}{.name}{'\n'}{end}"
```

Record:

``` text
Owner kind:
Owner name:
```

Where appropriate, continue read-only ownership tracing:

``` bash
kubectl get replicaset <replicaset> -n <namespace> -o yaml
```

Determine the authoritative workload identity before investigating
configuration.

------------------------------------------------------------------------

## 7. Establish Minimum Supported Blast Radius

Inspect comparable Pods:

``` bash
kubectl get pods -n <namespace> -o wide
```

If you know the workload labels:

``` bash
kubectl get pods -n <namespace> -l <label-selector> -o wide
```

Record only what the evidence supports:

``` text
[ ] One Pod
[ ] One workload
[ ] One namespace
[ ] One node
[ ] One node pool
[ ] One zone
[ ] One workload class
[ ] Multiple pools
[ ] Cluster-wide
[ ] Unknown
```

Rule:

``` text
OBSERVED IMPACT ≠ ASSUMED BLAST RADIUS
```

------------------------------------------------------------------------

# Phase 2 --- Prove Scheduling State

## 8. Confirm Pod Phase

``` bash
kubectl get pod <pod> -n <namespace> \
  -o jsonpath="{.status.phase}{'\n'}"
```

Record:

``` text
Phase:
```

If the Pod is Pending, do not immediately classify the incident as
scheduler failure.

``` text
POD PENDING ≠ SCHEDULER FAILURE
```

------------------------------------------------------------------------

## 9. Capture Scheduling Identity

``` bash
kubectl get pod <pod> -n <namespace> \
  -o jsonpath="{.spec.schedulerName}{'\n'}{.spec.nodeName}{'\n'}"
```

Record:

``` text
schedulerName:
nodeName:
```

Interpretation:

``` text
nodeName empty
    → normal scheduler placement may apply

nodeName populated
    → verify whether direct assignment changed the normal path
```

Rule:

``` text
POD ASSIGNED TO NODE
    ≠
KUBE-SCHEDULER SELECTED THAT NODE
```

------------------------------------------------------------------------

## 10. Check Scheduling Gates

Inspect the rendered Pod:

``` bash
kubectl get pod <pod> -n <namespace> -o yaml
```

Look for:

``` yaml
spec:
  schedulingGates:
```

Also inspect conditions and Events for scheduling-gated evidence.

Record:

``` text
Scheduling gates present: YES / NO
Gate names:
SchedulingGated evidence:
```

Rule:

``` text
POD PENDING
    ≠
SCHEDULER IS CURRENTLY TRYING TO PLACE IT
```

And:

``` text
SCHEDULING READINESS ≠ SCHEDULING FEASIBILITY
```

------------------------------------------------------------------------

## 11. Capture Scheduler Events

Run early:

``` bash
kubectl describe pod <pod> -n <namespace>
```

Record the scheduling-related Event exactly:

``` text
Reason:
Message:
First timestamp:
Last timestamp:
Count:
```

Do not simplify a multi-reason scheduler message into one generic label.

Example evidence:

``` text
0/12 nodes are available:
4 Insufficient memory,
3 node(s) had untolerated taint,
5 node(s) didn't match Pod's node affinity/selector.
```

Rule:

``` text
SCHEDULER SUMMARY ≠ SINGLE ROOT CAUSE
```

And:

``` text
SCHEDULER REASON ≠ UNDERLYING ROOT CAUSE
```

------------------------------------------------------------------------

## 12. Capture Namespace Events

``` bash
kubectl get events -n <namespace> \
  --sort-by=.metadata.creationTimestamp
```

Focus on the incident window.

Record:

``` text
Relevant timestamp:
Object:
Reason:
Message:
```

Do not assume current state fully represents failure-time state.

------------------------------------------------------------------------

## 13. Preserve the Rendered Pod

``` bash
kubectl get pod <pod> -n <namespace> -o yaml
```

Capture or record the relevant sections before any approved remediation
occurs:

``` text
metadata.uid
metadata.creationTimestamp

spec.schedulerName
spec.schedulingGates
spec.nodeName

spec.nodeSelector
spec.affinity
spec.tolerations
spec.topologySpreadConstraints

spec.containers[].resources
spec.initContainers[].resources

spec.volumes
```

Rule:

``` text
DESIRED ≠ RENDERED ≠ SCHEDULED
```

And:

``` text
SOURCE MANIFEST ≠ RENDERED POD RESOURCE CONTRACT
```

------------------------------------------------------------------------

## 14. Record Resource Requests and Limits

Inspect:

``` bash
kubectl get pod <pod> -n <namespace> -o yaml
```

Create a table:

  Container     CPU Request   CPU Limit   Memory Request   Memory Limit
  ----------- ------------- ----------- ---------------- --------------
                                                         
                                                         

Also inspect init containers when present.

Record:

``` text
Effective scheduling request observations:
```

Do not substitute runtime usage for requests.

------------------------------------------------------------------------

## 15. Record QoS Class

``` bash
kubectl get pod <pod> -n <namespace> \
  -o jsonpath="{.status.qosClass}{'\n'}"
```

Record:

``` text
QoS class:
```

Use this as context only.

``` text
QoS CLASS ≠ SCHEDULING ROOT CAUSE
```

------------------------------------------------------------------------

## 16. Record nodeSelector

``` bash
kubectl get pod <pod> -n <namespace> \
  -o jsonpath="{.spec.nodeSelector}{'\n'}"
```

Record:

``` text
nodeSelector:
```

------------------------------------------------------------------------

## 17. Record Node Affinity

Inspect:

``` bash
kubectl get pod <pod> -n <namespace> -o yaml
```

Record separately:

``` text
Required node affinity:
Preferred node affinity:
```

Rules:

``` text
REQUIRED AFFINITY → ELIGIBILITY

PREFERRED AFFINITY → SCORING

PREFERRED AFFINITY ≠ HARD PLACEMENT REQUIREMENT
```

------------------------------------------------------------------------

## 18. Record Tolerations

``` bash
kubectl get pod <pod> -n <namespace> \
  -o jsonpath="{range .spec.tolerations[*]}{.key}{'\t'}{.operator}{'\t'}{.value}{'\t'}{.effect}{'\t'}{.tolerationSeconds}{'\n'}{end}"
```

Record all tolerations relevant to placement.

Rule:

``` text
TOLERATION ≠ PLACEMENT REQUIREMENT
```

------------------------------------------------------------------------

## 19. Record Topology Constraints

Inspect:

``` bash
kubectl get pod <pod> -n <namespace> -o yaml
```

Record:

``` text
Pod affinity:
Pod anti-affinity:
Topology spread constraints:
Zone/region requirements:
Other topology restrictions:
```

Do not collapse all topology behavior into node affinity.

------------------------------------------------------------------------

## 20. Record Storage Requirements

List PVCs:

``` bash
kubectl get pvc -n <namespace>
```

Inspect relevant PVCs:

``` bash
kubectl describe pvc <pvc> -n <namespace>
```

Record:

``` text
PVC:
Status:
StorageClass:
Requested capacity:
Volume:
Relevant Events:
```

------------------------------------------------------------------------

## 21. Inspect StorageClass

``` bash
kubectl get storageclass <storage-class> -o yaml
```

Record:

``` text
StorageClass:
Provisioner:
volumeBindingMode:
allowedTopologies:
```

Rule:

``` text
PVC PENDING ≠ STORAGE FAILURE PROVEN
```

If `WaitForFirstConsumer` is configured, a Pending PVC can be part of
the expected scheduler-aware binding workflow.

------------------------------------------------------------------------

## 22. Inspect Bound PV Where Applicable

``` bash
kubectl get pv <pv> -o yaml
```

Record:

``` text
PV:
StorageClass:
Node affinity:
Zone/topology:
```

Compare storage topology with Pod placement constraints.

------------------------------------------------------------------------

## 23. Advanced Check --- nodeName + WaitForFirstConsumer

If both are true:

``` text
Pod .spec.nodeName is populated
StorageClass volumeBindingMode = WaitForFirstConsumer
```

record:

``` text
Direct node assignment present: YES
WaitForFirstConsumer present: YES
Potential scheduler-assisted volume-binding conflict requires investigation: YES
```

Do not mutate either configuration during this lab.

------------------------------------------------------------------------

## 24. Advanced Check --- Custom Scheduler

If:

``` text
schedulerName != default-scheduler
```

record:

``` text
Custom scheduler:
Scheduler configuration available to responder: YES / NO
```

If authorized read-only access exists, investigate scheduler
configuration for additional effective constraints such as profile-level
affinity.

Rule:

``` text
VISIBLE POD AFFINITY
    ≠
COMPLETE EFFECTIVE AFFINITY
WHEN CUSTOM SCHEDULER PROFILES ARE USED
```

------------------------------------------------------------------------

# Phase 3 --- Reduce Candidate Nodes

## 25. Inventory Nodes

``` bash
kubectl get nodes -o wide
```

Record:

``` text
Registered node count:
```

------------------------------------------------------------------------

## 26. Inspect Node Readiness

``` bash
kubectl get nodes
```

Count:

``` text
Registered:
Ready:
NotReady:
SchedulingDisabled:
```

Rule:

``` text
NODE EXISTS ≠ NODE READY
```

------------------------------------------------------------------------

## 27. Inspect Node Conditions

For candidate Nodes:

``` bash
kubectl describe node <node>
```

Record:

``` text
Ready:
MemoryPressure:
DiskPressure:
PIDPressure:
NetworkUnavailable:
Unschedulable:
```

Also record relevant Node Events.

------------------------------------------------------------------------

## 28. Inspect Node Labels

``` bash
kubectl get nodes --show-labels
```

For a focused Node:

``` bash
kubectl get node <node> --show-labels
```

Compare labels against:

``` text
Pod nodeSelector
Required node affinity
Expected node-pool labels
Zone/topology requirements
```

Record:

``` text
Ready nodes:
nodeSelector-compatible nodes:
required-affinity-compatible nodes:
```

------------------------------------------------------------------------

## 29. Inspect Node Taints

``` bash
kubectl describe node <node>
```

Record all relevant taints:

``` text
Node:
Taint:
Effect:
Pod tolerates it: YES / NO
```

Rule:

``` text
ONE TAINT TOLERATED
    ≠
ALL RELEVANT TAINTS TOLERATED
```

Count:

``` text
Placement-compatible nodes:
Taint-compatible nodes:
```

------------------------------------------------------------------------

## 30. Validate Topology Compatibility

Compare Pod topology constraints with candidate Node labels.

Useful inventory:

``` bash
kubectl get nodes \
  -L topology.kubernetes.io/region,topology.kubernetes.io/zone
```

Record:

``` text
Taint-compatible nodes:
Topology-compatible nodes:
```

Rule:

``` text
TOTAL CLUSTER CAPACITY
    ≠
CAPACITY IN THE REQUIRED TOPOLOGY
```

------------------------------------------------------------------------

## 31. Validate Storage Compatibility

Compare:

``` text
Pod topology
PVC / PV topology
StorageClass behavior
Node topology
```

Record:

``` text
Topology-compatible nodes:
Storage-compatible nodes:
```

If storage does not apply:

``` text
Storage constraint applicable: NO
Storage-compatible nodes: same as topology-compatible nodes
```

------------------------------------------------------------------------

## 32. Inspect Node Allocatable Resources

For candidate Nodes:

``` bash
kubectl describe node <node>
```

Record:

``` text
Capacity:
Allocatable:
Allocated resources:
```

Pay particular attention to:

``` text
cpu requests
memory requests
special resources
```

------------------------------------------------------------------------

## 33. Compare Pod Request With Node Accounting

For every remaining candidate Node, compare:

``` text
Pod request
+
existing requested resources
<=
node allocatable
```

Create:

  --------------------------------------------------------------------------
  Node           CPU Fit        Memory Fit     Other Resource Overall
                                               Fit            Resource Fit
  -------------- -------------- -------------- -------------- --------------
                                                              

                                                              
  --------------------------------------------------------------------------

Count:

``` text
Storage-compatible nodes:
Resource-fit nodes:
```

Rules:

``` text
REQUEST ≠ USAGE

NODE ALLOCATABLE ≠ NODE CAPACITY
```

------------------------------------------------------------------------

## 34. Optional Runtime Context

If Metrics Server or another compatible metrics source exists:

``` bash
kubectl top nodes
```

and, if useful:

``` bash
kubectl top pod <pod> -n <namespace> --containers
```

This is contextual evidence only.

``` text
kubectl top ≠ SCHEDULER CAPACITY MODEL
```

And:

``` text
LOW RUNTIME UTILIZATION ≠ SCHEDULABLE CAPACITY
```

------------------------------------------------------------------------

## 35. Check for Resource Fragmentation

Create a candidate matrix:

  -----------------------------------------------------------------------------------------
  Node     Placement   Taints      Topology    Storage     CPU         Memory      Final
  -------- ----------- ----------- ----------- ----------- ----------- ----------- --------
           PASS/FAIL   PASS/FAIL   PASS/FAIL   PASS/FAIL   PASS/FAIL   PASS/FAIL   

                                                                                   
  -----------------------------------------------------------------------------------------

Look for patterns where resources exist across the pool but not together
on one eligible Node.

Rule:

``` text
AGGREGATE CLUSTER CAPACITY ≠ POD FIT
```

And:

``` text
LOW CLUSTER UTILIZATION
    ≠
SCHEDULABLE CAPACITY EXISTS FOR THIS POD
```

------------------------------------------------------------------------

## 36. Build Candidate Counts

Complete:

``` text
Registered Nodes:                 _____
Ready/Schedulable Nodes:          _____
nodeSelector-Compatible:          _____
Required-Affinity-Compatible:     _____
Taint-Compatible:                 _____
Topology-Compatible:              _____
Storage-Compatible:               _____
Resource-Fit:                     _____
Final Feasible Nodes:             _____
```

The diagnostic funnel is an SRE troubleshooting model. It is not
intended to assert the exact internal execution order of every
kube-scheduler plugin.

Central question:

``` text
WHERE DID THE CANDIDATE SET BECOME ZERO?
```

------------------------------------------------------------------------

## 37. Build an Exclusion Ledger

Create one row per investigated Node.

  Node   First Diagnostic Exclusion   Supporting Evidence
  ------ ---------------------------- ---------------------
                                      
                                      
                                      

Possible exclusions:

``` text
Not Ready
Unschedulable
nodeSelector
required affinity
untolerated taint
topology
storage
CPU fit
memory fit
other resource fit
feasible
```

A Node can violate multiple constraints. The ledger uses a consistent
diagnostic exclusion so candidate counts are not accidentally
double-counted.

Rule:

``` text
ONE NODE'S STATE
    ≠
CLUSTER SCHEDULABILITY FOR THE POD
```

------------------------------------------------------------------------

## 38. Identify Lowest Proven Healthy Layer

Complete:

``` text
Rendered Pod:                  HEALTHY / FAILED / UNKNOWN
Scheduling readiness:          HEALTHY / FAILED / UNKNOWN
Node readiness:                HEALTHY / FAILED / UNKNOWN
nodeSelector compatibility:    HEALTHY / FAILED / UNKNOWN
Required affinity:             HEALTHY / FAILED / UNKNOWN
Taint compatibility:           HEALTHY / FAILED / UNKNOWN
Topology compatibility:        HEALTHY / FAILED / UNKNOWN
Storage compatibility:         HEALTHY / FAILED / UNKNOWN
Resource fit:                  HEALTHY / FAILED / UNKNOWN
```

Record:

``` text
Lowest Proven Healthy Layer:
```

------------------------------------------------------------------------

## 39. Identify First Failed Transition

Example:

``` text
Lowest Proven Healthy Layer:
Storage compatibility

First Failed Transition:
Storage-compatible nodes → resource-fit nodes
```

Your result:

``` text
First Failed Transition:
```

Remember:

``` text
FIRST FAILED TRANSITION
    ≠
FINAL ROOT CAUSE AUTOMATICALLY
```

------------------------------------------------------------------------

# Phase 4 --- Investigate Capacity Path

## 40. Identify Intended Node Pool

Use workload configuration, node labels, platform metadata, or
documented infrastructure conventions.

Record:

``` text
Intended node pool:
Evidence:
```

Do not infer a pool solely from workload type.

``` text
GPU WORKLOAD ≠ GPU NODE POOL TARGETING PROVEN
```

------------------------------------------------------------------------

## 41. Inventory Pool Nodes

Use the actual label convention for your environment.

Example:

``` bash
kubectl get nodes -l node-pool=<pool> -o wide
```

Do not assume `node-pool` is universally used; it is an
environment-specific example.

Record:

``` text
Expected pool:
Registered pool nodes:
Ready pool nodes:
```

------------------------------------------------------------------------

## 42. Compare Pool Node Configuration

For pool Nodes, compare:

``` text
labels
taints
instance type
zone
capacity
allocatable
special resources
readiness
conditions
Kubernetes version
container runtime
```

Create:

  ----------------------------------------------------------------------------
  Node       Ready      Instance   Zone       Labels     Taints     Resource
                                              Expected   Expected   Shape
                                                                    Expected
  ---------- ---------- ---------- ---------- ---------- ---------- ----------
                                                                    

                                                                    
  ----------------------------------------------------------------------------

Rules:

``` text
SAME NODE POOL ≠ IDENTICAL EFFECTIVE NODE STATE

NODE BELONGS TO EXPECTED POOL
    ≠
NODE HAS EXPECTED EFFECTIVE CONFIGURATION
```

------------------------------------------------------------------------

## 43. Look for Label Drift

Compare equivalent Nodes.

Record:

``` text
Expected label:
Nodes matching:
Nodes missing:
Unexpected values:
```

Do not manually patch drift during this lab.

``` text
LABEL MISMATCH PROVEN
    ≠
MANUAL LABEL PATCH AUTHORIZED
```

------------------------------------------------------------------------

## 44. Look for Taint Drift

Record:

``` text
Expected taints:
Nodes matching:
Nodes missing expected taints:
Nodes with unexpected taints:
```

Do not manually change taints.

``` text
OBSERVED DRIFT ≠ PERMISSION TO PATCH
```

------------------------------------------------------------------------

## 45. Determine Whether Additional Capacity Would Help

Answer:

``` text
WOULD ADDING A CORRECTLY CONFIGURED NODE
MAKE THIS POD SCHEDULABLE?
```

Evaluate:

``` text
Scheduling gate resolved?               YES / NO / N/A
Selector satisfiable?                   YES / NO
Required affinity satisfiable?          YES / NO
Tolerations compatible?                 YES / NO
Topology satisfiable?                   YES / NO
Storage satisfiable?                    YES / NO
Correct node resource shape possible?   YES / NO
```

Conclusion:

``` text
Correct new capacity would help:
YES / NO / UNKNOWN
```

Rule:

``` text
MORE NODES ≠ MORE FEASIBLE NODES
```

------------------------------------------------------------------------

## 46. Determine Whether Autoscaling Applies

Record:

``` text
Node autoscaling expected for this pool:
YES / NO / UNKNOWN

Autoscaler / provisioner implementation:
```

Examples might include environment-specific cluster autoscaling or node
provisioning systems. Use the implementation actually deployed in your
environment.

------------------------------------------------------------------------

## 47. Discover Autoscaling Components Read-Only

If authorized and known:

``` bash
kubectl get pods -A
```

Use environment-specific labels/names to locate the relevant
autoscaling/provisioning component.

Do not assume one specific autoscaler implementation.

Record:

``` text
Autoscaling component:
Namespace:
Pod:
Status:
```

------------------------------------------------------------------------

## 48. Inspect Autoscaler / Provisioner Logs When Authorized

Read-only example:

``` bash
kubectl logs <autoscaler-pod> -n <namespace> --since=30m
```

Adjust the incident window appropriately.

Search for evidence related to:

``` text
unschedulable Pod detection
candidate node group / pool
scale-up decision
provisioning request
constraint mismatch
maximum size
cloud capacity
instance availability
quota
bootstrap
```

Record exact evidence and timestamp.

Do not interpret absence of one log line as proof that an action did not
occur without considering retention and logging configuration.

------------------------------------------------------------------------

## 49. Classify Scale-Up Outcome

Choose only when evidence supports it:

``` text
[ ] A — Scale-up not required
[ ] B — Scale-up required but not triggered
[ ] C — Scale-up triggered but provisioning failed
[ ] D — Capacity created but Pod still cannot schedule
[ ] UNKNOWN
```

For outcome D, investigate:

``` text
labels
taints
instance type
resource shape
topology
storage
workload constraints
node readiness
```

Rule:

``` text
NODE CREATED ≠ USEFUL CAPACITY CREATED
```

------------------------------------------------------------------------

## 50. Separate Capacity States

Record counts where evidence is available:

``` text
Desired capacity:
Provisioned machines:
Registered nodes:
Ready nodes:
Correctly configured nodes:
Feasible nodes:
```

Rules:

``` text
PROVISIONED ≠ REGISTERED

REGISTERED ≠ READY

READY ≠ CORRECTLY CONFIGURED

CORRECTLY CONFIGURED ≠ FEASIBLE
```

------------------------------------------------------------------------

## 51. Investigate Provisioned but Not Registered

If infrastructure evidence shows a machine exists but Kubernetes does
not show the Node:

``` text
Provisioned machine evidence:
Kubernetes Node present: NO
```

Potential investigation domain:

``` text
bootstrap
kubelet
networking
authentication
cluster registration
infrastructure initialization
```

Do not classify this as scheduler failure.

------------------------------------------------------------------------

## 52. Investigate Registered but Not Ready

``` bash
kubectl describe node <node>
```

Inspect:

``` text
Conditions
Events
kubelet-related state
network readiness
pressure conditions
runtime information
```

Record:

``` text
Node registered:
Node Ready:
First unhealthy condition:
Relevant timestamp:
```

------------------------------------------------------------------------

## 53. Investigate Ready but Placement-Incompatible

If a new Node is Ready but excluded:

``` text
Expected labels present:
Expected taints present:
Selector match:
Required affinity match:
Topology match:
Storage compatibility:
```

Record the first proven exclusion.

------------------------------------------------------------------------

## 54. Investigate Ready but Resource-Incompatible

Record:

``` text
Node allocatable CPU:
Node allocatable memory:
Other allocatable resources:

Pod CPU request:
Pod memory request:
Other requested resources:

Fit:
```

A newly created Node can still be too small for the Pod.

------------------------------------------------------------------------

## 55. Scale-to-Zero / Zero-Node Pool Check

If the intended pool currently has zero registered Nodes:

``` text
Expected pool can scale from zero:
YES / NO / UNKNOWN

How workload requirements map to prospective capacity:
KNOWN / UNKNOWN
```

Do not automatically classify zero Nodes as failure.

``` text
ZERO NODES ≠ NODE POOL FAILURE PROVEN
```

------------------------------------------------------------------------

## 56. Build Time-to-Capacity Timeline

Complete with the best available evidence:

``` text
Pod created:
Pod first Pending:
First scheduling failure:
Autoscaler detected demand:
Scale decision:
Provisioning requested:
Machine created:
Node registered:
Node Ready:
Pod scheduled:
```

Then classify delay:

``` text
Detection delay:
Decision delay:
Provisioning delay:
Bootstrap delay:
Registration delay:
Readiness delay:
Scheduling delay:
```

------------------------------------------------------------------------

## 57. Preserve Evidence Before Manual Scale-Up

Before any separately approved remediation, ensure you have captured:

``` text
Pod UID
scheduler Events
rendered Pod
candidate counts
exclusion ledger
pool state
autoscaler evidence
provisioning evidence
timeline
```

Rule:

``` text
MANUAL SCALE-UP
    CAN DESTROY AUTOSCALER FAILURE EVIDENCE
```

This lab does not perform the scale-up.

------------------------------------------------------------------------

# Phase 5 --- Conclude, Hand Off & Validate

## 58. Build Incident Timeline

Create:

  Time   Layer          Event   Evidence
  ------ -------------- ------- ----------
         Pod                    
         Scheduler              
         Autoscaler             
         Provisioning           
         Node                   
         Remediation            

Rule:

``` text
CURRENT STATE ≠ HISTORICAL INCIDENT STATE
```

------------------------------------------------------------------------

## 59. Reconfirm Minimum Supported Blast Radius

After investigation, update:

``` text
Minimum Supported Blast Radius:
```

Evidence:

``` text
Affected Pods:
Affected workloads:
Affected namespaces:
Affected nodes:
Affected pools:
Affected zones:
```

Do not expand the scope beyond the evidence.

------------------------------------------------------------------------

## 60. Assign Evidence Confidence

Use:

``` text
PROVEN
SUPPORTED
UNKNOWN
ASSUMED
```

Create:

  Finding   Confidence   Evidence
  --------- ------------ ----------
                         
                         
                         

Rule:

``` text
ASSUMED ≠ ROOT CAUSE
```

------------------------------------------------------------------------

## 61. Determine Failure Domain

Choose the narrowest evidence-supported domain:

``` text
Rendered workload configuration
Scheduling readiness
Scheduler configuration
Node health
Placement constraints
Taints / tolerations
Topology
Storage
Resource fit / fragmentation
Dedicated node pool
Autoscaling
Infrastructure provisioning
Node bootstrap
Node registration
Node readiness
Unknown
```

Record:

``` text
Failure Domain:
```

------------------------------------------------------------------------

## 62. Determine Authoritative Desired-State Owner

For the failing configuration/state, identify who owns desired state.

Possible examples:

``` text
Application configuration
TrueFoundry
GitOps
Terraform / IaC
Kubernetes platform
Cloud infrastructure
Autoscaler / provisioner
Storage platform
```

Record:

``` text
Observed field/state:
Authoritative desired-state owner:
Evidence:
```

Rule:

``` text
OBSERVED DRIFT ≠ PERMISSION TO PATCH
```

------------------------------------------------------------------------

## 63. Determine Current Actionable Owner

Record:

``` text
Symptom:
First Failed Transition:
Failure Domain:
Evidence Confidence:
Current Actionable Owner:
Requested Action:
```

Rule:

``` text
SYMPTOM ≠ FAILURE DOMAIN ≠ OWNER
```

------------------------------------------------------------------------

## 64. TrueFoundry Context

When the workload is deployed through TrueFoundry, record:

``` text
TrueFoundry workload/service:
Workspace/environment:
Rendered Kubernetes workload:
Namespace:
Pod:
```

Then locate the first proven failure in:

``` text
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

Rule:

``` text
ERROR VISIBLE THROUGH TRUEFOUNDRY
    ≠
TRUEFOUNDRY ROOT CAUSE
```

------------------------------------------------------------------------

## 65. Evidence Handoff Contract

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
nodeSelector-Compatible Nodes:
Required-Affinity-Compatible Nodes:
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
Correctly Configured Pool Nodes:

Autoscaling Expected:
Autoscaler Evidence:
Provisioning Evidence:
Scale-Up Classification:

Pod Created:
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

------------------------------------------------------------------------

## 66. Post-Remediation Validation Checklist

This section is executed only after an approved remediation has occurred
through the appropriate operational process.

Use read-only validation:

``` bash
kubectl get pod <pod> -n <namespace> -o wide
kubectl describe pod <pod> -n <namespace>
```

Where relevant:

``` bash
kubectl get pods -n <namespace> -o wide
kubectl get nodes
kubectl get pvc -n <namespace>
```

Validate:

``` text
[ ] Pod scheduled
[ ] Pod placed on expected Node / pool
[ ] Pod Ready
[ ] Containers healthy
[ ] PVCs bound / attached as expected
[ ] No new scheduling failure Events
[ ] Workload replacement stable
[ ] Expected capacity healthy
[ ] Blast radius cleared
[ ] Service behavior validated by the responsible team
[ ] Root-cause remediation status recorded
```

Rules:

``` text
POD SCHEDULED ≠ INCIDENT FULLY RESOLVED
```

and:

``` text
SERVICE RESTORED ≠ ROOT CAUSE REMEDIATED
```

------------------------------------------------------------------------

# Final Decision Worksheet

## 67. Scheduling Investigation Summary

Complete:

``` text
1. Correct workload identity proven:
   YES / NO

2. Scheduling readiness proven:
   YES / NO

3. Scheduler evidence captured:
   YES / NO

4. Rendered scheduling contract captured:
   YES / NO

5. Candidate-node funnel completed:
   YES / NO

6. Candidate set reached zero:
   YES / NO / NOT APPLICABLE

7. First Failed Transition:
   ______________________________

8. Correctly configured new capacity would help:
   YES / NO / UNKNOWN

9. Autoscaling investigation required:
   YES / NO

10. Scale-up classification:
    A / B / C / D / UNKNOWN

11. Lowest Proven Healthy Layer:
    ______________________________

12. Failure Domain:
    ______________________________

13. Evidence Confidence:
    PROVEN / SUPPORTED / UNKNOWN / ASSUMED

14. Authoritative Desired-State Owner:
    ______________________________

15. Current Actionable Owner:
    ______________________________

16. Requested Action:
    ______________________________
```

------------------------------------------------------------------------

# Acceptance Checklist

## 68. Lab Completion Criteria

The lab is complete when you can answer all applicable items without
guessing:

``` text
[ ] Correct cluster/context verified
[ ] Namespace verified
[ ] Pod name and UID captured
[ ] Workload owner identified
[ ] Minimum Supported Blast Radius established
[ ] Pod state captured
[ ] schedulerName captured
[ ] schedulingGates checked
[ ] nodeName checked
[ ] Scheduler Events preserved
[ ] Rendered Pod contract inspected
[ ] Requests/limits recorded
[ ] QoS recorded as context
[ ] nodeSelector evaluated
[ ] Required/preferred affinity evaluated
[ ] Taints/tolerations evaluated
[ ] Topology constraints evaluated
[ ] Storage constraints evaluated
[ ] Node readiness evaluated
[ ] Candidate counts completed
[ ] Resource fit evaluated using requests/allocatable
[ ] Exclusion ledger completed
[ ] Intended node pool identified where applicable
[ ] Pool consistency checked
[ ] Label/taint drift checked
[ ] Whether new capacity would help determined
[ ] Autoscaler evidence collected where applicable
[ ] Provisioning evidence collected where applicable
[ ] Scale-up outcome classified
[ ] Time-to-capacity timeline built
[ ] Lowest Proven Healthy Layer identified
[ ] First Failed Transition identified
[ ] Evidence confidence assigned
[ ] Failure Domain identified
[ ] Desired-state owner identified
[ ] Current Actionable Owner identified
[ ] Evidence Handoff Contract completed
[ ] Post-remediation validation criteria documented
```

------------------------------------------------------------------------

# Production Rules Recap

``` text
UNKNOWN WORKLOAD IDENTITY = NO MUTATION

POD PENDING = SYMPTOM, NOT ROOT CAUSE

POD PENDING ≠ SCHEDULER FAILURE

SCHEDULING READINESS ≠ SCHEDULING FEASIBILITY

SCHEDULER SUMMARY ≠ SINGLE ROOT CAUSE

SCHEDULER REASON ≠ UNDERLYING ROOT CAUSE

DESIRED ≠ RENDERED ≠ SCHEDULED

SOURCE MANIFEST ≠ RENDERED POD RESOURCE CONTRACT

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

LOW CLUSTER UTILIZATION
    ≠
SCHEDULABLE CAPACITY EXISTS FOR THIS POD

ONE NODE'S STATE
    ≠
CLUSTER SCHEDULABILITY FOR THE POD

POOL SIZE ≠ READY NODE COUNT ≠ FEASIBLE NODE COUNT

SAME NODE POOL ≠ IDENTICAL EFFECTIVE NODE STATE

MORE NODES ≠ MORE FEASIBLE NODES

NODE CREATED ≠ USEFUL CAPACITY CREATED

PROVISIONED ≠ REGISTERED ≠ READY ≠ FEASIBLE

CURRENT STATE ≠ HISTORICAL INCIDENT STATE

OBSERVED DRIFT ≠ PERMISSION TO PATCH

REMEDIATION CAN DESTROY INCIDENT EVIDENCE

FIRST FAILED TRANSITION
    ≠
FINAL ROOT CAUSE AUTOMATICALLY

ASSUMED ≠ ROOT CAUSE

ERROR VISIBLE THROUGH TRUEFOUNDRY
    ≠
TRUEFOUNDRY ROOT CAUSE

SYMPTOM ≠ FAILURE DOMAIN ≠ OWNER

POD SCHEDULED ≠ INCIDENT FULLY RESOLVED

SERVICE RESTORED ≠ ROOT CAUSE REMEDIATED
```

------------------------------------------------------------------------

## Lab Outcome

A successful investigation should produce an evidence-based statement
similar to:

``` text
The Pod is Pending because the rendered workload requires the dedicated
compute pool and all placement, taint, topology, and storage constraints
are satisfied, but zero eligible Nodes can satisfy the Pod's memory
request.

The Lowest Proven Healthy Layer is storage compatibility.

The First Failed Transition is storage-compatible nodes → resource-fit
nodes.

Additional correctly configured capacity would make the Pod schedulable.

Autoscaling was expected but no successful provisioning transition was
observed during the incident window.

The scheduling symptom is therefore separated from the current
capacity/provisioning failure domain, and the evidence package is ready
for handoff to the authoritative infrastructure owner.
```

The objective is not to guess the fix.

``` text
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
REMEDIATE THROUGH APPROVED DESIRED STATE
        ↓
VALIDATE SERVICE RESTORATION
        ↓
VALIDATE ROOT-CAUSE REMEDIATION
```
