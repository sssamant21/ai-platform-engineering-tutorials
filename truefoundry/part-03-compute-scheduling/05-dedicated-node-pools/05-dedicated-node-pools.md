# Part 3.5 --- Dedicated Node Pools

## 1. Purpose

A dedicated node pool is an infrastructure grouping of Kubernetes worker
nodes intended for a particular workload class.

Common examples include:

-   GPU / AI workloads
-   High-memory workloads
-   Compute-intensive workloads
-   Batch workloads
-   Platform / infrastructure services
-   Production-critical workloads
-   Specialized hardware

A dedicated pool commonly combines:

``` text
NODE POOL
   ↓
Nodes
   ↓
Labels
+
Taints
+
Placement policy
+
Resource capacity
+
Provisioning / autoscaling
```

The workload then supplies compatible scheduling requirements:

``` text
WORKLOAD
   ↓
nodeSelector / nodeAffinity
+
Tolerations
+
Resource requests
+
Other scheduling constraints
```

**Production rule**

``` text
NODE POOL EXISTS ≠ WORKLOAD CAN SCHEDULE THERE
```

------------------------------------------------------------------------

## 2. Kubernetes Schedules Nodes, Not Infrastructure Pools

Cloud and Kubernetes platforms commonly organize worker infrastructure
into concepts such as node pools, node groups, managed node groups,
machine pools, or provisioner-managed capacity.

These abstractions create and manage Kubernetes Nodes. The Kubernetes
scheduler ultimately evaluates Nodes.

``` text
INFRASTRUCTURE NODE POOL
        ↓
Creates / manages
        ↓
KUBERNETES NODES
        ↓
Labels
Taints
Topology
Resources
Readiness
Other scheduling properties
        ↓
kube-scheduler
```

``` text
CLOUD NODE POOL EXISTS
    ≠
KUBERNETES SCHEDULER HAS A "NODE POOL" OBJECT
```

A label such as `node-pool=gpu` is an example of a pool-identification
label. It is not automatically a universal Kubernetes-standard label.

------------------------------------------------------------------------

## 3. Why Dedicated Node Pools?

Dedicated pools can separate infrastructure with different requirements.

``` text
General Applications
        ↓
General-Purpose Nodes

Large JVM Workloads
        ↓
High-Memory Nodes

AI / LLM Workloads
        ↓
Accelerator-Capable Nodes
```

Potential benefits include specialized hardware access, capacity
isolation, workload placement control, performance predictability,
failure-domain management, operational separation, independent scaling,
and cost management.

``` text
DEDICATED NODE POOL ≠ AUTOMATIC WORKLOAD ISOLATION
```

The scheduling policy must implement the intended isolation.

------------------------------------------------------------------------

## 4. Node Pool Identification

Nodes commonly carry labels identifying infrastructure characteristics
or operational purpose.

Example:

``` text
node-pool=ai
```

Inspect nodes:

``` bash
kubectl get nodes --show-labels
```

Inspect one node:

``` bash
kubectl get node <node> -o yaml
```

Labels may participate in `nodeSelector`, node affinity, topology
decisions, operational inventory, and automation.

``` text
NODE HAS POOL LABEL ≠ WORKLOAD TARGETS THAT LABEL
```

------------------------------------------------------------------------

## 5. Targeting the Pool

A workload can use `nodeSelector`:

``` yaml
nodeSelector:
  node-pool: ai
```

Or required node affinity:

``` yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: node-pool
              operator: In
              values:
                - ai
```

Conceptually:

``` text
WORKLOAD
   ↓
Placement Requirement
   ↓
node-pool=ai
   ↓
Matching Nodes
```

``` text
POOL LABEL MATCH ≠ POD CAN SCHEDULE
```

Other scheduling constraints still apply.

------------------------------------------------------------------------

## 6. Protecting Dedicated Capacity

Suppose the dedicated nodes have:

``` text
dedicated=ai:NoSchedule
```

An intended workload might contain:

``` yaml
tolerations:
  - key: dedicated
    operator: Equal
    value: ai
    effect: NoSchedule
```

The model becomes:

``` text
NODE SELECTOR / AFFINITY
        ↓
TARGET INTENDED CAPACITY

TAINT
        ↓
REPEL INCOMPATIBLE PODS

TOLERATION
        ↓
ALLOW COMPATIBLE POD THROUGH TAINT
```

A common dedicated-pool pattern is:

``` text
ATTRACT / REQUIRE
        +
REPEL
        +
TOLERATE
```

------------------------------------------------------------------------

## 7. Toleration Does Not Target a Pool

A toleration means that a matching taint does not automatically exclude
the Pod. It does not mean "schedule this Pod on that node."

``` text
TOLERATION ≠ NODE POOL TARGETING
```

A workload that merely tolerates `dedicated=ai:NoSchedule` may still be
eligible for other Nodes unless additional placement requirements
restrict it.

------------------------------------------------------------------------

## 8. Targeting Does Not Automatically Protect the Pool

Suppose a workload requires `node-pool=ai`, but the AI Nodes are not
protected by an appropriate scheduling policy. Other workloads may still
be able to consume that capacity.

``` text
WORKLOAD TARGETS DEDICATED POOL
    ≠
OTHER WORKLOADS ARE EXCLUDED
```

A complete design must evaluate both:

1.  Can intended workloads enter?
2.  Can unintended workloads enter?

------------------------------------------------------------------------

## 9. Dedicated Does Not Automatically Mean Security Isolation

Dedicated placement can support operational isolation. It should not
automatically be treated as a complete security boundary.

``` text
DEDICATED NODE POOL ≠ COMPLETE SECURITY BOUNDARY
```

When node labels participate in security-sensitive isolation, the trust
and ownership of those labels must also be considered. Detailed security
controls are covered later in the Security track.

------------------------------------------------------------------------

## 10. Scheduler Candidate Funnel

A useful mental model is:

``` text
ALL NODES
    ↓
Ready / schedulable candidates
    ↓
Pool / label compatibility
    ↓
nodeSelector
    ↓
required nodeAffinity
    ↓
Node taints / Pod tolerations
    ↓
Topology / storage / other hard constraints
    ↓
Resource fit
    ↓
FEASIBLE NODES
    ↓
Scheduler scoring
    ↓
SELECTED NODE
```

The most useful incident question is often:

``` text
WHERE DID THE CANDIDATE SET BECOME ZERO?
```

------------------------------------------------------------------------

## 11. Feasible Nodes

A pool may contain 10 Nodes while the workload has 0 feasible Nodes.

``` text
POOL SIZE ≠ FEASIBLE NODE COUNT
```

A Node is useful to a workload only when the relevant scheduling
requirements can be satisfied.

------------------------------------------------------------------------

## 12. Resource Fit Still Applies

Suppose a workload correctly targets the pool and tolerates the pool
taint, but requests 8 CPU and 32 GiB memory. If no matching Node has
sufficient allocatable capacity:

``` text
PLACEMENT POLICY       PASS
TAINT COMPATIBILITY    PASS
RESOURCE FIT           FAIL
```

The Pod can remain Pending.

``` text
CORRECT NODE POOL ≠ SUFFICIENT NODE POOL CAPACITY
```

------------------------------------------------------------------------

## 13. Aggregate Capacity Is Not Pod Fit

Suppose:

``` text
Node A → 4 CPU available
Node B → 4 CPU available
```

A Pod requests 8 CPU. The pool may have eight CPU in aggregate, but
Kubernetes needs a single feasible Node for that Pod.

``` text
AGGREGATE POOL CAPACITY ≠ POD FIT
```

------------------------------------------------------------------------

## 14. Capacity Fragmentation

Dedicated capacity may be fragmented.

``` text
Node A
CPU available
Memory insufficient

Node B
Memory available
Required device unavailable

Node C
Required device available
CPU insufficient
```

The pool may appear underutilized while still having zero feasible
Nodes.

``` text
LOW POOL UTILIZATION ≠ CAPACITY EXISTS FOR THIS POD
```

------------------------------------------------------------------------

## 15. Requests Are Not Runtime Usage

From Parts 3.1 and 3.2:

``` text
REQUEST ≠ USAGE
kubectl top ≠ SCHEDULER CAPACITY MODEL
```

Runtime metrics remain useful operational evidence, but a Node showing
low current CPU utilization can still have insufficient unallocated
requested capacity for another Pod.

------------------------------------------------------------------------

## 16. Topology Can Reduce Effective Pool Capacity

Suppose a dedicated pool contains six Ready Nodes:

``` text
zone-a → 3
zone-b → 3
```

If a workload effectively requires `zone-a`, its relevant candidate
population may be only three Nodes.

``` text
POOL HAS N READY NODES
    ≠
ALL N NODES ARE VALID FOR THIS POD
```

------------------------------------------------------------------------

## 17. Storage Can Also Reduce the Candidate Set

A stateful workload may have enough CPU and memory, correct labels, and
compatible taints but still be unable to use a Node because storage
cannot be attached or satisfied from that topology.

``` text
NODE HAS CPU / MEMORY
    ≠
NODE SATISFIES ALL WORKLOAD CONSTRAINTS
```

------------------------------------------------------------------------

## 18. Desired Pool Is Not the Same as Existing Capacity

Production operations must separate:

``` text
DESIRED NODE POOL
        ↓
PROVISIONED INFRASTRUCTURE
        ↓
REGISTERED KUBERNETES NODES
        ↓
READY / SCHEDULABLE NODES
        ↓
PLACEMENT-COMPATIBLE NODES
        ↓
TAINT-COMPATIBLE NODES
        ↓
RESOURCE-FIT NODES
        ↓
FEASIBLE NODES
```

``` text
DESIRED ≠ PROVISIONED ≠ REGISTERED ≠ READY ≠ FEASIBLE
```

------------------------------------------------------------------------

## 19. Count Nodes at Every Transition

During an incident, record:

``` text
Expected pool nodes:                 ?
Provisioned infrastructure nodes:    ?
Registered Kubernetes nodes:         ?
Ready nodes:                         ?
Placement-compatible nodes:          ?
Taint-compatible nodes:              ?
Resource-fit nodes:                  ?
Final feasible nodes:                ?
```

Example:

``` text
Expected                  5
   ↓
Registered                5
   ↓
Ready                     4
   ↓
Placement-compatible      4
   ↓
Taint-compatible          3
   ↓
Resource-fit              0
```

``` text
POOL SIZE ≠ READY NODE COUNT ≠ FEASIBLE NODE COUNT
```

------------------------------------------------------------------------

## 20. Node Pool Exists but Kubernetes Nodes May Not

Infrastructure configuration might define a dedicated pool with desired
size 2 while Kubernetes currently has zero registered Nodes.

Possible reasons include provisioning failure, cloud capacity, quota,
bootstrap failure, registration failure, scale-to-zero, or other
infrastructure constraints.

``` text
NODE POOL CONFIGURATION EXISTS
    ≠
USABLE KUBERNETES NODES EXIST
```

------------------------------------------------------------------------

## 21. Registered Does Not Mean Ready

A provisioned machine can successfully register with Kubernetes but
still be unusable because it is NotReady, unschedulable,
condition-tainted, incorrectly labeled, unexpectedly tainted, or lacks
allocatable resources.

``` text
NODE PROVISIONED ≠ NODE REGISTERED
NODE REGISTERED ≠ NODE READY
NODE READY ≠ NODE FEASIBLE FOR THIS POD
```

------------------------------------------------------------------------

## 22. Node Conditions Matter

Dedicated pool Nodes can develop conditions related to readiness, memory
pressure, disk pressure, PID pressure, networking, or reachability.
Kubernetes can represent relevant conditions through well-known node
taints.

``` text
EXPECTED NODE COUNT ≠ HEALTHY USABLE CAPACITY
```

Do not remove condition-related taints merely to make scheduling
succeed.

``` text
REMOVE SCHEDULING SYMPTOM ≠ REMEDIATE NODE FAILURE
```

------------------------------------------------------------------------

## 23. Pool Consistency

Nodes expected to serve the same workload class should be compared
across:

-   Labels
-   Taints
-   Instance type
-   CPU
-   Memory
-   Accelerators
-   Allocatable resources
-   Availability zone
-   Readiness
-   Node conditions
-   Kubernetes version
-   Runtime
-   Relevant system components

``` text
SAME NODE POOL ≠ IDENTICAL EFFECTIVE NODE STATE
```

Infrastructure membership alone does not prove operational equivalence.

------------------------------------------------------------------------

## 24. Label Drift

Suppose the pool should produce `node-pool=ai`, but one Node reports
`node-pool=general`. A required affinity rule may exclude that Node.

``` text
LABEL MISMATCH PROVEN
    ≠
MANUAL LABEL PATCH AUTHORIZED
```

First determine who owns the desired label. Potential owners include
Terraform, managed Kubernetes configuration, a node provisioner,
autoscaler, bootstrap automation, GitOps, or platform automation.

------------------------------------------------------------------------

## 25. Taint Drift

Suppose:

``` text
Node A → dedicated=ai:NoSchedule
Node B → dedicated=ai:NoSchedule
Node C → no dedicated taint
```

Node C may become available to workloads that were not intended to
consume specialized capacity.

``` text
MISSING TAINT CAN BE A CAPACITY-ISOLATION PROBLEM
TAINT DIFFERENCE ≠ MANUAL FIX AUTHORIZED
```

------------------------------------------------------------------------

## 26. Observed Drift Does Not Authorize Mutation

Whether the difference involves labels, taints, pool size, instance
type, zones, or bootstrap configuration:

``` text
OBSERVED DRIFT ≠ PERMISSION TO PATCH
```

Determine authoritative desired-state ownership first.

------------------------------------------------------------------------

## 27. Scheduling Evidence Comes Before Scaling

For a Pending Pod, start with scheduler evidence:

``` bash
kubectl describe pod <pod> -n <namespace>
```

Events may identify affinity mismatch, untolerated taints, insufficient
CPU or memory, storage constraints, topology constraints, or other
scheduling restrictions.

``` text
POD PENDING ≠ NODE POOL CAPACITY FAILURE PROVEN
```

Do not begin with "increase the node pool."

------------------------------------------------------------------------

## 28. Verify the Pod Is Actually Ready for Scheduling

A Pending Pod may intentionally have scheduling gates or other
conditions preventing normal scheduling.

``` text
POD PENDING
    ≠
SCHEDULER IS CURRENTLY TRYING TO PLACE IT
```

This is an advanced diagnostic consideration explored further in Part
3.6.

------------------------------------------------------------------------

## 29. Rendered Pod State Is Critical Evidence

Capture:

``` bash
kubectl get pod <pod> -n <namespace> -o yaml
```

Inspect:

-   `nodeSelector`
-   `nodeAffinity`
-   `tolerations`
-   resource requests
-   `schedulerName`
-   `schedulingGates`
-   `nodeName`
-   topology-related constraints

Separate:

``` text
DESIRED WORKLOAD CONFIGURATION
        ↓
PLATFORM / ADMISSION PROCESSING
        ↓
RENDERED POD
        ↓
SCHEDULER EVALUATION
```

``` text
DESIRED ≠ RENDERED ≠ SCHEDULED
```

------------------------------------------------------------------------

## 30. Workload Type Does Not Prove Placement Policy

``` text
AI WORKLOAD ≠ AI NODE POOL TARGETING PROVEN
GPU WORKLOAD ≠ GPU NODE POOL TARGETING PROVEN
```

Inspect the actual rendered placement requirements. Detailed GPU
scheduling remains in Part 4.

------------------------------------------------------------------------

## 31. Scaling Is Not Automatically the Fix

A Pending Pod may result from:

-   Wrong `nodeSelector`
-   Wrong required affinity
-   Missing toleration
-   Unexpected taint
-   Topology constraint
-   Storage constraint
-   Oversized resource request
-   No Ready Nodes
-   Capacity exhaustion
-   Provisioning failure
-   Scheduling gate

``` text
POD PENDING ≠ NODE POOL MUST SCALE
```

The better question is:

``` text
WOULD ADDING A CORRECTLY CONFIGURED NODE
MAKE THIS POD SCHEDULABLE?
```

------------------------------------------------------------------------

## 32. Node Autoscaling Path

Conceptually:

``` text
Pod unschedulable
        ↓
Autoscaler / provisioner evaluates demand
        ↓
Can compatible capacity be created?
        ↓
Provisioning request
        ↓
Cloud infrastructure
        ↓
Machine created
        ↓
Node bootstrap
        ↓
Kubernetes registration
        ↓
Node Ready
        ↓
Expected labels / taints / resources
        ↓
Scheduler reevaluation
        ↓
Pod placement
```

There are many possible failure transitions.

------------------------------------------------------------------------

## 33. Autoscaler Enabled Does Not Guarantee Scale-Up

Capture:

``` text
Autoscaling enabled:
Minimum size:
Maximum size:
Current size:
Provisioning limits:
Relevant pool / provisioner:
```

Determine whether the Pod can match capacity that the provisioning
system is allowed to create.

``` text
AUTOSCALER ENABLED
    ≠
THIS POD CAN TRIGGER USEFUL CAPACITY

UNSCHEDULABLE POD
    ≠
AUTOSCALER WILL NECESSARILY ADD A NODE
```

------------------------------------------------------------------------

## 34. More Nodes Do Not Necessarily Help

If new Nodes are created without the required pool label or with an
incompatible taint, the workload may gain zero additional feasible
Nodes.

``` text
MORE NODES ≠ MORE FEASIBLE NODES
```

------------------------------------------------------------------------

## 35. Scale-to-Zero

Some provisioning systems and managed Kubernetes configurations permit a
node group or pool to reach zero active Nodes.

A current node count of zero does not by itself prove failure.

Investigate:

-   Is zero expected?
-   Should this workload trigger provisioning?
-   Does the workload match the pool's provisioning constraints?
-   Was demand detected?
-   Was provisioning attempted?
-   Did infrastructure creation succeed?
-   Did the Node register?
-   Did it become Ready?

``` text
ZERO NODES ≠ NODE POOL FAILURE PROVEN
```

------------------------------------------------------------------------

## 36. Separate Autoscaler and Infrastructure Failures

Use the full chain:

``` text
Scheduler
    ↓
Unschedulable Pod
    ↓
Autoscaler Decision
    ↓
Provisioning Request
    ↓
Cloud Infrastructure
    ↓
Machine Created
    ↓
Bootstrap
    ↓
Kubernetes Registration
    ↓
Node Ready
```

Examples:

``` text
No scale-up decision
→ autoscaler / constraint investigation

Scale-up requested, machine absent
→ infrastructure provisioning investigation

Machine exists, Kubernetes Node absent
→ bootstrap / registration investigation

Node exists, NotReady
→ node / runtime / networking investigation

Node Ready, wrong labels / taints
→ desired-state / bootstrap investigation
```

------------------------------------------------------------------------

## 37. Scheduling Symptom Does Not Prove Scheduler Failure

Example:

``` text
Scheduler
    ↓
Correctly reports no feasible nodes

Autoscaler
    ↓
Correctly requests capacity

Cloud provider
    ↓
Cannot create requested capacity
```

The Pod remains Pending, but:

``` text
SCHEDULING SYMPTOM ≠ SCHEDULER FAILURE
```

------------------------------------------------------------------------

## 38. Time-to-Capacity

For dynamically provisioned capacity, capture:

``` text
Pod became Pending:
Autoscaler detected demand:
Provisioning requested:
Machine created:
Node registered:
Node Ready:
Pod scheduled:
```

This separates detection delay, provisioning delay, bootstrap delay,
registration delay, readiness delay, and scheduling delay.

Without a timeline, normal infrastructure startup time can be mistaken
for a scheduler incident.

------------------------------------------------------------------------

## 39. Preserve Evidence Before Manual Scaling

Before manually scaling, capture:

-   Pod YAML
-   Pod UID
-   Scheduler Events
-   Current pool size
-   Placement constraints
-   Tolerations
-   Resource requests
-   Candidate-node counts
-   Autoscaler evidence
-   Provisioning evidence
-   Timeline

``` text
MANUAL SCALE-UP CAN DESTROY AUTOSCALER FAILURE EVIDENCE
REMEDIATION CAN DESTROY INCIDENT EVIDENCE
```

------------------------------------------------------------------------

## 40. Preserve Evidence Before Node Replacement

Deleting or recycling a problematic Node can remove node conditions,
events, runtime state, bootstrap symptoms, labels, taints, and local
system evidence.

Capture the relevant evidence first.

------------------------------------------------------------------------

## 41. Production Identity Model

Before changing anything:

``` text
Environment
↓
Cluster / Context
↓
Namespace
↓
Workload
↓
Pod
↓
Pod UID
↓
Intended Node Pool
↓
Timestamp
```

``` text
UNKNOWN WORKLOAD IDENTITY = NO NODE POOL CHANGE
```

Also establish node-pool identity:

``` text
Cloud / Provider:
Cluster:
Node Pool:
Expected Labels:
Expected Taints:
Expected Instance Type:
Expected Zones:
Minimum Size:
Maximum Size:
Current Size:
Autoscaling:
Provisioner:
```

------------------------------------------------------------------------

## 42. Minimum Supported Blast Radius

Determine whether evidence supports impact to:

-   One Pod
-   One workload
-   One Node
-   One node pool
-   One availability zone
-   One instance type
-   Multiple pools
-   Cluster-wide

``` text
ONE DEDICATED POOL IMPACTED
    ≠
CLUSTER-WIDE SCHEDULING FAILURE
```

Do not promote one workload's inability to schedule into a Kubernetes
scheduling outage without evidence.

------------------------------------------------------------------------

## 43. Lowest Proven Healthy Layer

Example:

``` text
Rendered Pod                    HEALTHY
      ↓
Pool affinity                  HEALTHY
      ↓
Taint compatibility            HEALTHY
      ↓
Ready candidate Nodes          HEALTHY
      ↓
Resource fit                   FAILED
```

Record:

``` text
Lowest Proven Healthy Layer:
Dedicated-pool placement compatibility.

First Failed Transition:
Placement-compatible Nodes → resource fit.
```

------------------------------------------------------------------------

## 44. Provisioning Failure Example

``` text
Pod rendered correctly         HEALTHY
       ↓
Scheduler evaluates Pod        HEALTHY
       ↓
No existing feasible Node      CONFIRMED
       ↓
Autoscaler detects demand      HEALTHY
       ↓
Provisioning requested         HEALTHY
       ↓
Infrastructure creation        FAILED
```

``` text
First Failed Transition:
Provisioning request → infrastructure creation
```

The scheduler is not the proven failure domain.

------------------------------------------------------------------------

## 45. Registration Failure Example

``` text
Provisioning requested         HEALTHY
       ↓
Machine created                HEALTHY
       ↓
Bootstrap initiated            HEALTHY
       ↓
Kubernetes registration        FAILED
```

The workload's Pending state is downstream evidence.

------------------------------------------------------------------------

## 46. Node Readiness Failure Example

``` text
Machine exists                 HEALTHY
       ↓
Node registered                HEALTHY
       ↓
Node Ready                     FAILED
```

Possible investigation areas include:

-   Kubelet
-   Container runtime
-   CNI / networking
-   Disk
-   Memory
-   Certificates
-   Bootstrap
-   Cloud integration

Do not remove scheduling protections merely to bypass an unhealthy Node
state.

------------------------------------------------------------------------

## 47. Evidence Confidence

Classify major conclusions as:

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
No Ready Node currently matches the requirement.

PROVEN
Autoscaler observed the unschedulable Pod.

SUPPORTED
Cloud capacity prevented provisioning.

UNKNOWN
Whether an alternative instance type could have been provisioned.

ASSUMED
TrueFoundry caused the capacity failure.
```

``` text
ASSUMED ≠ ROOT CAUSE
```

------------------------------------------------------------------------

## 48. Desired-State Ownership

Identify ownership independently for:

``` text
Pool existence
Minimum size
Maximum size
Desired size
Instance type
Availability zones
Node labels
Node taints
Autoscaling policy
Bootstrap configuration
Workload nodeSelector
Workload affinity
Workload tolerations
Resource requests
```

Potential owners can include TrueFoundry, Terraform, GitOps, managed
Kubernetes, cloud infrastructure, the Kubernetes platform team, an
autoscaler/provisioner, or application configuration.

Do not assume one team owns every layer.

------------------------------------------------------------------------

## 49. Current Actionable Owner

Separate symptom from failure domain and Current Actionable Owner.

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
```

``` text
SYMPTOM ≠ FAILURE DOMAIN ≠ OWNER
```

------------------------------------------------------------------------

## 50. TrueFoundry Operational Context

For a TrueFoundry-managed workload:

``` text
TrueFoundry Deployment Intent
        ↓
Desired Workload Configuration
        ↓
Rendered Kubernetes Pod
        ↓
Placement Requirements
        ↓
Tolerations
        ↓
Resource Requests
        ↓
Kubernetes Scheduler
        ↓
Existing Dedicated Capacity
        ↓
Optional Provisioning / Autoscaling
        ↓
Kubernetes Nodes
        ↓
Placement
```

``` text
ERROR VISIBLE THROUGH TRUEFOUNDRY
    ≠
TRUEFOUNDRY ROOT CAUSE
```

A failure displayed through TrueFoundry can originate deeper in
Kubernetes or infrastructure.

------------------------------------------------------------------------

## 51. Node Pool Changes Have Broad Blast Radius

Changes to pool labels, pool taints, instance type, minimum size,
maximum size, desired size, zones, autoscaling, or bootstrap
configuration may affect multiple workloads.

``` text
NODE POOL CHANGE ≠ LOCAL POD FIX
```

Determine affected workload populations before mutation.

------------------------------------------------------------------------

## 52. Evidence Handoff Contract

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

## 53. Production Investigation Flow

``` text
IDENTITY
    ↓
INTENDED NODE POOL
    ↓
POD SCHEDULING READINESS
    ↓
SCHEDULER EVIDENCE
    ↓
RENDERED PLACEMENT POLICY
    ↓
EXPECTED POOL CONFIGURATION
    ↓
PROVISIONED INFRASTRUCTURE
    ↓
REGISTERED NODES
    ↓
READY / SCHEDULABLE NODES
    ↓
POOL / LABEL COMPATIBILITY
    ↓
TAINT / TOLERATION COMPATIBILITY
    ↓
TOPOLOGY / STORAGE / OTHER CONSTRAINTS
    ↓
RESOURCE FIT
    ↓
FEASIBLE NODES
    ↓
IF ZERO:
CAN AUTOSCALING HELP?
    ↓
AUTOSCALER DECISION
    ↓
PROVISIONING
    ↓
BOOTSTRAP
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
    ↓
SAFE REMEDIATION
```

------------------------------------------------------------------------

## 54. Production Rules

``` text
UNKNOWN WORKLOAD IDENTITY = NO NODE POOL CHANGE

NODE POOL EXISTS ≠ WORKLOAD CAN SCHEDULE THERE

CLOUD NODE POOL EXISTS
    ≠
KUBERNETES SCHEDULER HAS A "NODE POOL" OBJECT

NODE HAS POOL LABEL ≠ WORKLOAD TARGETS THAT LABEL

POOL LABEL MATCH ≠ POD CAN SCHEDULE

WORKLOAD TARGETS DEDICATED POOL
    ≠
OTHER WORKLOADS ARE EXCLUDED

TOLERATION ≠ NODE POOL TARGETING

DEDICATED NODE POOL ≠ COMPLETE SECURITY BOUNDARY

CORRECT NODE POOL ≠ SUFFICIENT NODE POOL CAPACITY

AGGREGATE POOL CAPACITY ≠ POD FIT

LOW POOL UTILIZATION ≠ CAPACITY EXISTS FOR THIS POD

REQUEST ≠ USAGE

kubectl top ≠ SCHEDULER CAPACITY MODEL

POOL HAS N READY NODES
    ≠
ALL N NODES ARE VALID FOR THIS POD

DESIRED ≠ PROVISIONED ≠ REGISTERED ≠ READY ≠ FEASIBLE

POOL SIZE ≠ READY NODE COUNT ≠ FEASIBLE NODE COUNT

NODE POOL CONFIGURATION EXISTS
    ≠
USABLE KUBERNETES NODES EXIST

NODE PROVISIONED ≠ NODE REGISTERED

NODE REGISTERED ≠ NODE READY

NODE READY ≠ NODE FEASIBLE FOR THIS POD

SAME NODE POOL ≠ IDENTICAL EFFECTIVE NODE STATE

LABEL MISMATCH PROVEN
    ≠
MANUAL LABEL PATCH AUTHORIZED

TAINT DIFFERENCE ≠ MANUAL FIX AUTHORIZED

OBSERVED DRIFT ≠ PERMISSION TO PATCH

POD PENDING ≠ NODE POOL CAPACITY FAILURE PROVEN

POD PENDING
    ≠
SCHEDULER IS CURRENTLY TRYING TO PLACE IT

AI WORKLOAD ≠ AI NODE POOL TARGETING PROVEN

GPU WORKLOAD ≠ GPU NODE POOL TARGETING PROVEN

POD PENDING ≠ NODE POOL MUST SCALE

AUTOSCALER ENABLED
    ≠
THIS POD CAN TRIGGER USEFUL CAPACITY

UNSCHEDULABLE POD
    ≠
AUTOSCALER WILL NECESSARILY ADD A NODE

MORE NODES ≠ MORE FEASIBLE NODES

ZERO NODES ≠ NODE POOL FAILURE PROVEN

SCHEDULING SYMPTOM ≠ SCHEDULER FAILURE

MANUAL SCALE-UP
    CAN DESTROY AUTOSCALER FAILURE EVIDENCE

REMEDIATION CAN DESTROY INCIDENT EVIDENCE

ONE DEDICATED POOL IMPACTED
    ≠
CLUSTER-WIDE SCHEDULING FAILURE

ERROR VISIBLE THROUGH TRUEFOUNDRY
    ≠
TRUEFOUNDRY ROOT CAUSE

NODE POOL CHANGE ≠ LOCAL POD FIX

ASSUMED ≠ ROOT CAUSE

SYMPTOM ≠ FAILURE DOMAIN ≠ OWNER
```

------------------------------------------------------------------------

## 55. Scope Boundary

Part 3.5 establishes the production model for dedicated Kubernetes
capacity.

Detailed GPU infrastructure remains in Part 4, including:

-   GPU architecture
-   NVIDIA drivers
-   NVIDIA Device Plugin
-   GPU Operator
-   CUDA
-   GPU node pools
-   GPU resource requests
-   GPU validation
-   GPU troubleshooting

Detailed workload scheduling incident investigation follows immediately
in **Part 3.6 --- Workload Scheduling Troubleshooting**.

Part 3.6 combines the scheduling primitives from Parts 3.1--3.5 into a
single production troubleshooting methodology.
