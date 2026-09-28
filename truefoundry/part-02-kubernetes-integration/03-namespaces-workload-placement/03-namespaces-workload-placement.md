# Part 2.3 — Namespaces & Workload Placement

## 1. Purpose

A TrueFoundry workload ultimately runs as Kubernetes resources in a Compute Plane.

For production operations, knowing the application or Workspace name is not enough. An SRE must establish the complete placement identity:

```text
TrueFoundry Workspace
        ↓
Associated Kubernetes cluster
        ↓
Target Kubernetes namespace
        ↓
Namespace authorization / policy
        ↓
Workload controller
        ↓
Pod creation request
        ↓
Admission
        ↓
Pod object
        ↓
Scheduler
        ↓
Node
        ↓
Container startup
        ↓
Readiness
```

Each transition can fail independently.

The objective is to determine:

```text
WHAT workload is expected?
WHERE should it exist?
WHERE does it actually exist?
WHAT is the lowest proven healthy layer?
WHAT is the first failed transition?
WHAT is the minimum supported blast radius?
WHO currently owns the next actionable step?
```

---

## 2. Fundamental Placement Rule

TrueFoundry Workspaces provide a deployment scope associated with Kubernetes infrastructure.

Production operators must not determine the actual Kubernetes target solely from naming similarity.

```text
WORKSPACE NAME ≠ PROOF OF ACTUAL NAMESPACE
```

Use:

```text
Workspace
   ↓
Verify associated cluster
   ↓
Verify actual Kubernetes namespace
   ↓
Verify workload controller
   ↓
Verify Pod
```

A familiar-looking name is evidence at most. It is not sufficient proof of topology.

---

## 3. Production Placement Identity

Before changing anything, establish:

```text
Environment:
TrueFoundry Workspace:

Cluster:
Kubernetes context:
Namespace:
Namespace phase:

Controller kind:
Controller name:

Pod:
Node:

Image/version:
ServiceAccount:

Observation timestamp:
```

This is the workload's **Placement Identity**.

Production safety rule:

```text
UNKNOWN PLACEMENT = NO MUTATION
```

If the environment, cluster, namespace, or authoritative workload cannot be proven, continue discovery. Do not remediate based on assumptions.

---

## 4. Placement Has Multiple Dimensions

```text
PLATFORM PLACEMENT
TrueFoundry Workspace
        ↓
CLUSTER PLACEMENT
Kubernetes cluster
        ↓
LOGICAL PLACEMENT
Namespace
        ↓
WORKLOAD MATERIALIZATION
Controller / Pod
        ↓
SCHEDULER PLACEMENT
Node / node pool
        ↓
RUNTIME
Container
```

Key distinctions:

```text
Namespace Placement ≠ Node Placement
Correct Namespace ≠ Correct Node Placement
```

---

## 5. Four Production Placement Gates

### Gate 1 — Target

```text
Environment
    ↓
Workspace
    ↓
Cluster
    ↓
Namespace
```

### Gate 2 — Materialization

```text
Controller
    ↓
Pod creation request
    ↓
Admission
    ↓
Pod object
```

### Gate 3 — Placement

```text
Pod
    ↓
Scheduler
    ↓
Node
```

### Gate 4 — Execution

```text
Container startup
    ↓
Application startup
    ↓
Readiness
```

The first failed gate significantly narrows the investigation.

---

## 6. Verify the Kubernetes Target First

Before searching for workloads:

```bash
kubectl config current-context
kubectl cluster-info
kubectl get nodes
```

Confirm:

```text
Expected environment
        ↓
Expected cluster
        ↓
Current Kubernetes context
```

A query against the wrong cluster can return valid but irrelevant information.

```text
VALID KUBERNETES OUTPUT ≠ CORRECT INVESTIGATION TARGET
```

---

## 7. Discover and Verify the Namespace

Start read-only:

```bash
kubectl get namespaces
```

or:

```bash
kubectl get ns
```

For a candidate namespace:

```bash
kubectl get namespace <namespace>
```

Verify whether it is `Active`, `Terminating`, or otherwise abnormal.

```text
Namespace Exists ≠ Namespace Operational
```

---

## 8. Namespace Existence Does Not Prove Usability

A namespace may exist while deployment is blocked by controls such as:

```text
Kubernetes RBAC
tfy-agent namespace scope
ResourceQuota
LimitRange
admission policy
security policy
NetworkPolicy
organizational policy
```

Therefore:

```text
Namespace Exists ≠ Namespace Usable for Deployment
Correct Namespace ≠ Agent Authorized for Namespace
```

Detailed ServiceAccount and RBAC investigation is covered in Part 2.4.

---

## 9. Discover the Workload

If the workload name is known:

```bash
kubectl get pods -A | grep -i <workload>
```

On Windows CMD:

```bat
kubectl get pods -A | findstr /I "<workload>"
```

Also inspect relevant controllers:

```bash
kubectl get deployments -A
kubectl get statefulsets -A
kubectl get daemonsets -A
kubectl get jobs -A
kubectl get cronjobs -A
```

Establish:

```text
Workload
   ↓
Namespace
   ↓
Controller
   ↓
Pod
```

A missing Pod does not automatically mean scheduler failure. The Pod may never have been created.

---

## 10. Kubernetes Resource Identity

The same resource name can exist in multiple namespaces.

```text
staging / model-api
production / model-api
```

Therefore:

```text
Resource Name ≠ Complete Resource Identity
```

At minimum use:

```text
Cluster + Namespace + Kind + Name
```

For incident evidence, also record the observation timestamp.

---

## 11. Controller Before Pod

Determine Pod ownership:

```bash
kubectl get pod <pod> \
  -n <namespace> \
  -o jsonpath='{.metadata.ownerReferences[*].kind}{" "}{.metadata.ownerReferences[*].name}{"\n"}'
```

Common chains include:

```text
Deployment → ReplicaSet → Pod
StatefulSet → Pod
CronJob → Job → Pod
```

Production rule:

```text
POD ≠ WORKLOAD DEFINITION
```

Changing a controller-owned Pod directly usually does not modify authoritative desired state.

---

## 12. Determine the Desired-State Owner

A resource may ultimately be controlled through:

```text
TrueFoundry
Helm
GitOps
Argo CD
Kubernetes Operator
another reconciliation system
```

Trace:

```text
Observed Pod
    ↑
Kubernetes controller
    ↑
Platform/controller
    ↑
Authoritative desired state
```

Production rule:

```text
DO NOT PATCH THE SYMPTOM
WHEN ANOTHER SYSTEM OWNS DESIRED STATE
```

---

## 13. Desired State vs Observed State

For replicated workloads compare:

```text
Desired replicas
Created Pods
Scheduled Pods
Ready Pods
```

Example:

```text
Desired:      4
Created:      4
Scheduled:    3
Ready:        3
```

This differs materially from:

```text
Desired:      4
Created:      2
Scheduled:    2
Ready:        2
```

Preserve these distinctions:

```text
Desired Replicas ≠ Created Pods
Created Pods ≠ Scheduled Pods
Scheduled Pods ≠ Ready Pods
```

---

## 14. Pod Creation and Scheduling Are Different

The lifecycle is:

```text
Controller
    ↓
Pod creation request
    ↓
Admission
    ↓
Pod object created
    ↓
Scheduling
    ↓
Node assignment
```

Critical rule:

```text
POD NOT CREATED ≠ POD PENDING
```

A Pod that does not exist cannot have a scheduler-placement failure.

---

## 15. Admission Failures

Admission happens before normal scheduler placement.

Namespace-level policy can influence whether a Pod is admitted:

```text
ResourceQuota
LimitRange
admission controls
security policy
```

Example:

```text
Deployment exists
        ↓
ReplicaSet exists
        ↓
Pod creation attempted
        ↓
Admission rejected
        X
```

Therefore:

```text
Deployment Exists ≠ Pod Creation Succeeded
```

Do not classify admission failure as scheduler failure.

---

## 16. ResourceQuota Awareness

Inspect:

```bash
kubectl get resourcequota -n <namespace>
kubectl describe resourcequota -n <namespace>
```

A namespace can reach quota even when the cluster has capacity.

```text
Namespace Quota Available ≠ Cluster Capacity Available
Cluster Capacity Available ≠ Namespace Quota Available
```

Quota and physical cluster capacity are different control layers.

---

## 17. LimitRange Awareness

Inspect:

```bash
kubectl get limitrange -n <namespace>
```

and, when required:

```bash
kubectl describe limitrange <name> -n <namespace>
```

A LimitRange can influence resource defaults or constraints during admission.

```text
Same submitted workload
       +
Different namespace policy
       =
Potentially different admission
or resource behavior
```

Do not assume changing a LimitRange automatically modifies already-created Pods.

---

## 18. Namespace Policy Inventory

Useful read-only discovery includes:

```bash
kubectl get resourcequota -n <namespace>
kubectl get limitrange -n <namespace>
kubectl get networkpolicy -n <namespace>
```

These commands do not represent every possible policy mechanism. Admission systems, security controls, cloud integration, and organizational governance may introduce additional policy.

---

## 19. Pod Scheduling

If the Pod exists, determine whether it has been scheduled:

```bash
kubectl describe pod <pod> -n <namespace>
```

Pay attention to:

```text
PodScheduled
```

Distinguish:

```text
Pod exists
    ↓
PodScheduled=False
```

from:

```text
Pod exists
    ↓
PodScheduled=True
    ↓
Container startup failure
```

These represent different failure domains.

---

## 20. Scheduling Evidence

A Pod may remain Pending because of factors such as:

```text
Insufficient CPU
Insufficient memory
Insufficient GPU
node selector mismatch
node affinity
taints/tolerations
topology constraints
PVC binding
scheduler policy
```

Inspect timestamped events:

```bash
kubectl get events \
  -n <namespace> \
  --sort-by=.metadata.creationTimestamp
```

Detailed scheduling mechanics are covered in Part 3.

Part 2.3 establishes the boundary:

```text
Pod Exists ≠ Pod Scheduled
```

---

## 21. Determine Node Placement

Once scheduling succeeds:

```bash
kubectl get pods -n <namespace> -o wide
```

Record the assigned node.

When required:

```bash
kubectl describe node <node>
```

Distinguish:

```text
Logical placement:
Namespace

Physical placement:
Node / node pool
```

Therefore:

```text
Namespace Placement ≠ Node Placement
```

---

## 22. Multi-Replica Placement

Do not inspect only one Pod for a replicated workload.

Example:

```text
model-api-a    node-01    Ready
model-api-b    node-02    Ready
model-api-c    <none>     Pending
```

Capture:

```text
Desired replicas:
Created replicas:
Scheduled replicas:
Ready replicas:
```

Avoid describing a partially degraded workload simply as healthy or failed.

---

## 23. Container Startup Is Another Boundary

If:

```text
Pod exists
PodScheduled=True
Node assigned
Container fails
```

placement succeeded.

Move the investigation toward:

```text
image
container configuration
volume mounting
runtime
startup command
application initialization
```

Therefore:

```text
Pod Scheduled ≠ Container Started
Container Started ≠ Ready
```

---

## 24. Namespace Resource Discovery

After verifying the namespace:

```bash
kubectl get all -n <namespace>
```

is useful initial discovery, but:

```text
kubectl get all ≠ literally all Kubernetes resources
```

Explicitly query relevant resource types when required:

```bash
kubectl get deployments -n <namespace>
kubectl get statefulsets -n <namespace>
kubectl get jobs -n <namespace>
kubectl get cronjobs -n <namespace>
kubectl get services -n <namespace>
kubectl get configmaps -n <namespace>
kubectl get secrets -n <namespace>
kubectl get resourcequota -n <namespace>
kubectl get limitrange -n <namespace>
kubectl get networkpolicy -n <namespace>
```

---

## 25. Secret Safety

Secret metadata may be useful:

```bash
kubectl get secrets -n <namespace>
```

But placement investigation normally does not require Secret values.

```text
SECRET EXISTS ≠ SECRET SHOULD BE DECODED
```

Do not expose credentials merely to prove that a Secret exists.

---

## 26. ServiceAccount Placement

Determine the Pod's Kubernetes identity:

```bash
kubectl get pod <pod> \
  -n <namespace> \
  -o jsonpath='{.spec.serviceAccountName}{"\n"}'
```

Conceptually:

```text
Namespace
    ↓
Workload
    ↓
Pod
    ↓
ServiceAccount
```

But:

```text
ServiceAccount Exists ≠ Authorization Correct
```

Detailed ServiceAccount and RBAC investigation belongs in Part 2.4.

---

## 27. Namespace Is Not a Complete Security Boundary

Do not teach:

```text
Different Namespace = Secure Isolation
```

Actual isolation can depend on:

```text
Namespace
+
RBAC
+
NetworkPolicy
+
ServiceAccounts
+
admission/security controls
+
cloud identity
+
node/runtime controls
```

Therefore:

```text
Namespace ≠ Complete Security Boundary
```

This is particularly important in shared clusters.

---

## 28. Cross-Namespace Dependencies

A workload may run in one namespace while depending on resources elsewhere:

```text
namespace-a
     │
     └── model-api
             │
             ├── service in namespace-b
             ├── shared ingress
             ├── external database
             └── object/model storage
```

Therefore:

```text
Workload Namespace ≠ Dependency Boundary
```

Blast-radius analysis must account for dependencies.

---

## 29. Shared Cluster Infrastructure

A namespace-local symptom may originate from cluster-wide infrastructure:

```text
DNS
CNI
CSI
Ingress controller
admission webhook
scheduler
node pool
Kubernetes API
```

Therefore:

```text
Symptom in Namespace ≠ Failure Originates in Namespace
```

Avoid assigning ownership based solely on where the symptom appears.

---

## 30. Labels and Annotations

Labels can help discovery:

```bash
kubectl get pod <pod> \
  -n <namespace> \
  --show-labels
```

Annotations can provide useful evidence:

```bash
kubectl get pod <pod> \
  -n <namespace> \
  -o jsonpath='{.metadata.annotations}{"\n"}'
```

They may reveal:

```text
application identity
release
controller
deployment tooling
policy
platform integration
observability
```

But:

```text
LABEL ≠ AUTOMATIC AUTHORITATIVE PROOF
ANNOTATION ≠ AUTOMATIC AUTHORITATIVE PROOF
```

Use metadata as evidence within the wider topology.

---

## 31. Placement Drift

Compare expected and observed configuration.

Potential drift includes:

```text
Workspace/namespace relationship
Controller configuration
ServiceAccount
Resource requests
Node selectors
Affinity
Tolerations
Runtime class
PVC references
Image/version
```

Production rule:

```text
Observed Drift ≠ Permission to Patch
```

Determine the authoritative configuration source before remediation.

---

## 32. Namespace Lifecycle Risk

Namespace operations can have broad consequences:

```text
namespace deletion
namespace recreation
quota changes
RBAC changes
NetworkPolicy changes
security-policy changes
```

Production rule:

```text
NAMESPACE CHANGE = BLAST-RADIUS EVENT
```

Namespace lifecycle remediation must use the appropriate production change process.

---

## 33. Failure Scenario — Wrong Cluster

Symptom:

```text
Expected workload not found
```

Before declaring deployment failure, verify:

```text
Environment
Workspace
Associated cluster
Kubernetes context
```

Example:

```text
Actual workload cluster:
cluster-a

Current kubectl context:
cluster-b
```

Classification:

```text
Workload failure:
NOT PROVEN

Investigation target:
INCORRECT
```

---

## 34. Failure Scenario — Wrong Namespace Assumption

Suppose:

```text
Workspace:
payments-prod

Assumed namespace:
payments-prod
```

but discovery proves the workload exists elsewhere according to deployed configuration.

Then:

```text
Workload placement:
HEALTHY

Investigation assumption:
INCORRECT
```

This demonstrates why:

```text
WORKSPACE NAME ≠ PROOF OF ACTUAL NAMESPACE
```

is an operational safety rule.

---

## 35. Failure Scenario — Quota Rejection

Evidence:

```text
Cluster                  PROVEN
Namespace                PROVEN
Controller               PROVEN
Pod creation attempted   PROVEN
Quota rejection          PROVEN
Expected Pod             NOT CREATED
```

Failure chain:

```text
Controller
    ↓
Pod creation request
    ↓
Admission
    X
```

Record:

```text
First Failed Transition:
Pod creation request → admission
```

Do not classify this as scheduler failure.

---

## 36. Failure Scenario — Pod Pending

Evidence:

```text
Cluster             PROVEN
Namespace           PROVEN
Controller          PROVEN
Pod                 PROVEN
Admission           PROVEN
PodScheduled        FALSE
```

If events show no matching schedulable capacity:

```text
Lowest Proven Healthy Layer:
Pod admission / creation

First Failed Transition:
Pod creation/admission → scheduler placement
```

Continue into Part 3 for detailed scheduling analysis.

---

## 37. Failure Scenario — Scheduled but Not Running

Evidence:

```text
Pod created         PROVEN
PodScheduled        TRUE
Node assigned       PROVEN
Container startup   FAILED
```

Placement succeeded.

Continue investigation into the container/runtime layer rather than treating this as a namespace-placement failure.

---

## 38. Lowest Proven Healthy Layer

Example:

```text
Cluster reachable             PROVEN
Namespace                     PROVEN
Controller                    PROVEN
Pod creation                  PROVEN
Admission                     PROVEN
Scheduling                    FAILED
Container startup             NOT TESTED
```

Record:

```text
Lowest Proven Healthy Layer:
Pod admission / creation
```

This establishes the boundary for the next investigation.

---

## 39. First Failed Transition

For the same case:

```text
Controller
    ↓
Pod creation
    ↓
Admission
    ↓
Scheduler
    X
```

Record:

```text
First Failed Transition:
Pod creation/admission → scheduler placement
```

This is more actionable than simply stating that deployment failed.

---

## 40. Minimum Supported Blast Radius

Possible scopes include:

```text
One container
One Pod
One replica
One controller
One workload
One namespace
One node
One node pool
One cluster
Multiple namespaces
Multiple clusters
```

Only claim what evidence supports.

```text
One Pod Pending ≠ Namespace Scheduling Failure
One Pod Pending ≠ Cluster Scheduling Failure
```

---

## 41. Evidence Confidence

Classify conclusions as:

```text
PROVEN
SUPPORTED
UNKNOWN
ASSUMED
```

Example:

```text
Correct cluster             PROVEN
Correct namespace           PROVEN
Pod admitted                PROVEN
GPU capacity unavailable    SUPPORTED
Cluster capacity exhausted  UNKNOWN
TrueFoundry platform issue  ASSUMED
```

Production rule:

```text
ASSUMED ≠ ROOT CAUSE
```

---

## 42. Current Actionable Owner

Ownership should follow the first proven failure, not merely the visible symptom.

Example:

```text
Symptom:
TrueFoundry workload not becoming Ready

Lowest Proven Healthy Layer:
Pod creation

First Failed Transition:
Scheduler placement

Evidence:
Insufficient matching capacity

Failure domain:
Kubernetes scheduling / capacity

Current Actionable Owner:
Team responsible for the relevant cluster/node capacity
```

Therefore:

```text
SYMPTOM ≠ FAILURE DOMAIN ≠ OWNER
```

---

## 43. Evidence Handoff Contract

Capture:

```text
Environment:
TrueFoundry Workspace:

Cluster:
Kubernetes context:
Namespace:
Namespace phase:

Controller kind:
Controller name:

Desired replicas:
Created Pods:
Scheduled Pods:
Ready Pods:

Affected Pod:
Pod phase:
PodScheduled:
Node:

ServiceAccount:

ResourceQuota:
LimitRange:

Relevant events:

Observation timestamp:

Lowest Proven Healthy Layer:

First Failed Transition:

Minimum Supported Blast Radius:

Evidence Confidence:

Authoritative Configuration Owner:

Current Actionable Owner:

Requested Action:

Sensitive information removed:
YES
```

The receiving team should know both what was proven and what action is requested.

---

## 44. Production Investigation Flow

```text
VERIFY ENVIRONMENT
        ↓
VERIFY TRUEFOUNDRY WORKSPACE
        ↓
VERIFY ASSOCIATED CLUSTER
        ↓
VERIFY KUBERNETES CONTEXT
        ↓
VERIFY ACTUAL NAMESPACE
        ↓
CHECK NAMESPACE STATE
        ↓
CHECK NAMESPACE ACCESS / POLICY
        ↓
DISCOVER CONTROLLER
        ↓
IDENTIFY DESIRED-STATE OWNER
        ↓
COMPARE DESIRED vs CREATED
        ↓
CHECK POD CREATION
        ↓
CHECK ADMISSION
        ↓
POD EXISTS?
        ↓
CHECK PodScheduled
        ↓
CHECK NODE PLACEMENT
        ↓
COMPARE CREATED vs SCHEDULED
        ↓
CHECK CONTAINER STARTUP
        ↓
COMPARE SCHEDULED vs READY
        ↓
CORRELATE TIMESTAMPED EVENTS
        ↓
DETERMINE LOWEST PROVEN HEALTHY LAYER
        ↓
DETERMINE FIRST FAILED TRANSITION
        ↓
DETERMINE MINIMUM SUPPORTED BLAST RADIUS
        ↓
IDENTIFY CURRENT ACTIONABLE OWNER
        ↓
EVIDENCE HANDOFF
        ↓
APPROVED REMEDIATION
```

---

## 45. Core Production Rules

```text
WORKSPACE NAME ≠ PROOF OF ACTUAL NAMESPACE

UNKNOWN PLACEMENT = NO MUTATION

Namespace Exists ≠ Namespace Operational

Namespace Exists ≠ Namespace Usable for Deployment

Correct Namespace ≠ Agent Authorized for Namespace

Resource Name ≠ Complete Resource Identity

POD ≠ WORKLOAD DEFINITION

Deployment Exists ≠ Pod Creation Succeeded

POD NOT CREATED ≠ POD PENDING

Pod Exists ≠ Pod Scheduled

Pod Scheduled ≠ Container Started

Container Started ≠ Ready

Desired Pods ≠ Created Pods

Created Pods ≠ Scheduled Pods

Scheduled Pods ≠ Ready Pods

Namespace Quota Available ≠ Cluster Capacity Available

Cluster Capacity Available ≠ Namespace Quota Available

Namespace Placement ≠ Node Placement

Namespace ≠ Complete Security Boundary

Workload Namespace ≠ Dependency Boundary

Symptom in Namespace ≠ Failure Originates in Namespace

kubectl get all ≠ all namespace resources

SECRET EXISTS ≠ SECRET SHOULD BE DECODED

Observed Drift ≠ Permission to Patch

NAMESPACE CHANGE = BLAST-RADIUS EVENT

DO NOT PATCH THE SYMPTOM
WHEN ANOTHER SYSTEM OWNS DESIRED STATE

ASSUMED ≠ ROOT CAUSE

SYMPTOM ≠ FAILURE DOMAIN ≠ OWNER
```

Operational objective:

```text
PROVE THE TARGET
        ↓
PROVE THE PLACEMENT CHAIN
        ↓
FIND THE FIRST FAILED TRANSITION
        ↓
PROVE THE BLAST RADIUS
        ↓
IDENTIFY THE AUTHORITATIVE OWNER
        ↓
TAKE THE NEXT APPROVED ACTION
```
