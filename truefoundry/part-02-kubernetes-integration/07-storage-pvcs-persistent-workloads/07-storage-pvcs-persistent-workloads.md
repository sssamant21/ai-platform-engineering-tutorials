# Part 2.7 --- Storage, PVCs & Persistent Workloads

## Purpose

Persistent storage incidents are different from ordinary stateless
workload failures because an incorrect remediation can affect both
service availability and application data.

This tutorial establishes a production/SRE method for understanding and
troubleshooting persistent storage used by TrueFoundry workloads running
on Kubernetes.

The central production question is:

> **What storage did the workload expect, what storage did it actually
> receive, where is the first proven failure in the storage path, what
> data is at risk, what is the smallest supported blast radius, and who
> owns the authoritative configuration?**

The core operating principle is:

``` text
DO NOT TROUBLESHOOT "STORAGE"
AS A SINGLE COMPONENT.

PROVE:

STORAGE IDENTITY
→ PROVISIONING
→ TOPOLOGY
→ ATTACHMENT WHEN REQUIRED
→ NODE PRESENTATION
→ APPLICATION ACCESS
→ CAPACITY
→ PERFORMANCE
→ DATA INTEGRITY

ONE TRANSITION AT A TIME.
```

------------------------------------------------------------------------

# 1. Production Storage Architecture

A useful conceptual path is:

``` text
TrueFoundry / Authoritative Configuration
        ↓
Rendered Kubernetes Workload
        ↓
PVC
        ↓
StorageClass / Static PV Selection
        ↓
CSI Provisioning
        ↓
PersistentVolume
        ↓
Infrastructure Storage
        ↓
[Volume Attachment — when required]
        ↓
Filesystem Mount OR Raw Block Device
        ↓
Container
        ↓
Application Storage Usage
        ↓
Application Data
```

Not every implementation uses every transition.

For example:

-   some PVs are statically provisioned;
-   some CSI drivers do not require a separate attach operation;
-   some volumes use `Filesystem`;
-   some use raw `Block`;
-   some workloads use external/object storage instead of PVCs.

Therefore, first discover the actual storage architecture.

------------------------------------------------------------------------

# 2. Five Storage States

For production analysis, separate storage into five states:

``` text
DESIRED STORAGE
        ↓
RENDERED STORAGE
        ↓
PROVISIONED STORAGE
        ↓
ATTACHED / MOUNTED STORAGE
        ↓
APPLICATION-OBSERVED STORAGE
```

These are not equivalent.

``` text
DESIRED STORAGE
≠
RENDERED STORAGE
≠
PROVISIONED STORAGE
≠
ATTACHED/MOUNTED STORAGE
≠
APPLICATION-OBSERVED STORAGE
```

A workload can therefore have:

-   correct desired configuration but incorrect rendered configuration;
-   correct PVC configuration but failed provisioning;
-   a bound PV but failed attachment;
-   an attached device but failed mount;
-   a mounted filesystem but application permission failures;
-   usable storage that is nevertheless too slow;
-   apparently healthy storage objects while the application sees the
    wrong data.

------------------------------------------------------------------------

# 3. Ephemeral and Persistent Storage

A Kubernetes workload may use:

``` text
Container writable layer
emptyDir
PersistentVolumeClaim
CSI ephemeral volume
Object storage
External filesystem
Application-managed remote storage
```

Do not assume that every workload with persistent data necessarily has a
PVC.

``` text
NO PVC
≠
NO PERSISTENT DATA
```

For data that must survive Pod replacement, the architecture must
explicitly provide appropriate persistence semantics.

``` text
POD RECREATED
≠
DATA SHOULD DISAPPEAR
```

But persistence is not backup:

``` text
PERSISTENCE
≠
BACKUP
```

------------------------------------------------------------------------

# 4. PersistentVolumeClaims

A PVC represents a request for storage.

Important fields commonly include:

``` yaml
spec:
  accessModes:
  resources:
    requests:
      storage:
  storageClassName:
  volumeMode:
```

The SRE should establish:

``` text
Which workload references the PVC?

Which namespace contains it?

What capacity was requested?

What access mode was requested?

What volume mode is used?

Which StorageClass was selected?

Which PV is bound?

What infrastructure storage ultimately serves it?
```

Rules:

``` text
PVC EXISTS
≠
PVC BOUND

PVC BOUND
≠
STORAGE USABLE
```

------------------------------------------------------------------------

# 5. Storage Identity

Storage identity must be proven before remediation.

Capture:

``` text
Environment
Cluster
Namespace
Workload
Pod
Pod UID
Node
Mount path / device
PVC
PVC UID
PV
StorageClass
CSI driver
Infrastructure volume identifier
Volume mode
Access mode
```

Important rules:

``` text
RESOURCE NAME
≠
RESOURCE IDENTITY

SAME PVC NAME
≠
SAME STORAGE IDENTITY
```

A PVC can be deleted and recreated with the same name while representing
a different Kubernetes object and potentially different underlying
storage.

Never infer identity from names alone.

------------------------------------------------------------------------

# 6. Prove the Workload-to-Storage Chain

For a persistent workload, prove:

``` text
Workload
→ Pod
→ Volume reference
→ PVC
→ PV
→ CSI volumeHandle / infrastructure volume
```

If application data appears missing:

``` text
DATA NOT VISIBLE
≠
DATA LOST
```

Possible explanations include:

``` text
Wrong environment
Wrong workload
Wrong Pod
Wrong PVC
Recreated PVC
Wrong PV
Wrong mount path
Wrong application path
Empty replacement volume
Wrong replica
Actual deletion
Filesystem corruption
Application-level data loss
```

Storage identity must be proven before declaring data loss.

------------------------------------------------------------------------

# 7. PVC Lifecycle

A simplified dynamically provisioned path is:

``` text
PVC Created
   ↓
StorageClass Selected
   ↓
Provisioning Triggered
   ↓
Infrastructure Storage Created
   ↓
PV Created
   ↓
PVC Bound
   ↓
Consumer Pod Scheduled
   ↓
[Volume Attached]
   ↓
Filesystem Mounted / Block Device Presented
   ↓
Container Started
   ↓
Application Uses Storage
```

Each arrow is a troubleshooting boundary.

``` text
PVC BOUND
≠
ATTACHMENT HEALTHY

ATTACHMENT HEALTHY
≠
MOUNT HEALTHY

MOUNT HEALTHY
≠
APPLICATION STORAGE HEALTHY
```

------------------------------------------------------------------------

# 8. Four Diagnostic Gates

## Gate 1 --- Provisioning

``` text
PVC
→ StorageClass
→ Provisioner / CSI
→ PV
→ Infrastructure Storage
```

Questions:

``` text
Was the correct StorageClass selected?

Was provisioning expected to happen immediately?

Did the provisioner accept the request?

Was a PV created?

Was the infrastructure volume created?

Did PVC/PV binding complete?
```

## Gate 2 --- Placement and Attachment

``` text
Consumer Pod
→ Scheduler
→ Node / Topology
→ [VolumeAttachment when required]
```

Questions:

``` text
Where was the Pod scheduled?

Is the selected node compatible with storage topology?

Does the driver require attachment?

Did attachment complete?

Is another node still using the volume?
```

## Gate 3 --- Node Presentation

``` text
Infrastructure Storage
→ CSI Node Path
→ Filesystem OR Raw Block Device
→ Container
```

Questions:

``` text
Did the node receive the volume?

Did mount/publish succeed?

Is the expected filesystem or block device presented?

Did Pod startup fail waiting for storage?
```

## Gate 4 --- Application

``` text
Container-visible Storage
→ Permissions
→ Expected Path
→ Application I/O
→ Application Data
```

Questions:

``` text
Can the application use the expected path?

Does its runtime identity have required permissions?

Is capacity available?

Are inodes available?

Is I/O performance acceptable?

Is the expected data present?
```

------------------------------------------------------------------------

# 9. StorageClasses

A StorageClass describes how storage should be provisioned.

Important properties can include:

``` text
Provisioner
Parameters
Reclaim policy
Volume binding mode
Allow volume expansion
Mount options
Topology restrictions
```

Inspect:

``` bash
kubectl get storageclass
```

and:

``` bash
kubectl describe storageclass <storage-class>
```

Rule:

``` text
STORAGECLASS EXISTS
≠
PROVISIONER FUNCTIONAL
```

------------------------------------------------------------------------

# 10. TrueFoundry and StorageClasses

TrueFoundry volume creation can expose StorageClass choices to users.

The StorageClasses available through the platform can depend on the
cluster and platform configuration.

Therefore:

``` text
STORAGECLASS EXISTS IN KUBERNETES
≠
STORAGECLASS AVAILABLE FOR TRUEFOUNDRY VOLUME CREATION
```

Likewise:

``` text
TRUEFOUNDRY VOLUME CONFIGURATION
≠
SUCCESSFUL KUBERNETES PROVISIONING
```

This creates an important troubleshooting boundary:

``` text
TrueFoundry Desired Configuration
        ↓
Rendered Kubernetes Configuration
        ↓
Kubernetes Storage Control Plane
        ↓
Runtime Storage
```

------------------------------------------------------------------------

# 11. Default StorageClass Behavior

A PVC can receive a default StorageClass depending on cluster
configuration even when `storageClassName` is not explicitly set by the
workload.

Therefore:

``` text
NO EXPLICIT storageClassName
≠
NO STORAGECLASS
```

Always inspect the actual PVC and resulting PV.

Do not infer the effective StorageClass solely from the application
configuration.

------------------------------------------------------------------------

# 12. Static Provisioning

Not every PV is dynamically provisioned.

A static model can be:

``` text
Infrastructure Volume
        ↓
Administrator-Created PV
        ↓
PVC
        ↓
Workload
```

Therefore:

``` text
PV EXISTS
≠
PV WAS DYNAMICALLY PROVISIONED
```

Discover the provisioning model before diagnosing the provisioner.

------------------------------------------------------------------------

# 13. Volume Binding Modes

A StorageClass can use binding behavior such as:

``` text
Immediate
```

or:

``` text
WaitForFirstConsumer
```

With `WaitForFirstConsumer`, provisioning or binding can intentionally
wait until a consumer Pod provides scheduling information.

Therefore:

``` text
PVC PENDING
≠
PVC BROKEN
```

A Pending PVC must be classified.

------------------------------------------------------------------------

# 14. Expected vs Abnormal Pending

## Expected Pending

Examples:

``` text
WaitForFirstConsumer
No consumer Pod yet
Consumer has not reached scheduling decision
```

## Abnormal Pending

Potential causes:

``` text
StorageClass missing
Provisioner failure
Cloud authorization failure
Topology conflict
Storage capacity unavailable
Invalid parameters
CSI controller failure
Backend storage failure
```

Do not delete and recreate a PVC merely because it is Pending.

``` text
PVC PENDING
≠
DELETE AND RECREATE PVC
```

------------------------------------------------------------------------

# 15. Access Modes

Common access modes include:

``` text
ReadWriteOnce
ReadOnlyMany
ReadWriteMany
ReadWriteOncePod
```

Do not infer more than the mode actually guarantees.

Especially:

``` text
ReadWriteOnce
≠
Exactly One Pod
```

`ReadWriteOnce` describes read/write mounting in relation to node access
semantics; it should not be interpreted as a universal one-Pod
guarantee.

Also:

``` text
ReadWriteOncePod
≠
ReadWriteOnce
```

Access mode alone does not describe the complete storage topology or
application architecture.

``` text
ACCESS MODE
≠
COMPLETE STORAGE TOPOLOGY
```

------------------------------------------------------------------------

# 16. Volume Modes

Persistent volumes may use:

``` text
volumeMode: Filesystem
```

or:

``` text
volumeMode: Block
```

With Filesystem mode:

``` text
Volume
→ Filesystem
→ Mount
→ Container path
```

With Block mode:

``` text
Volume
→ Raw block device
→ Container device
```

Therefore:

``` text
PERSISTENT VOLUME
≠
ALWAYS A FILESYSTEM
```

Troubleshooting must follow the actual volume mode.

------------------------------------------------------------------------

# 17. PersistentVolumes

Inspect:

``` bash
kubectl get pv
```

and the relevant PV:

``` bash
kubectl describe pv <pv>
```

Correlate:

``` text
PVC
→ PV
→ StorageClass
→ CSI Driver
→ Infrastructure Volume
```

Important properties can include:

``` text
Capacity
Access modes
Volume mode
Reclaim policy
StorageClass
Claim reference
CSI driver
volumeHandle
Node affinity
Status
```

Do not treat the PV as an isolated object.

------------------------------------------------------------------------

# 18. CSI

Kubernetes commonly integrates storage systems through the Container
Storage Interface.

Conceptually:

``` text
Kubernetes
   ↓
CSI Components
   ↓
Storage / Cloud API
   ↓
Infrastructure Storage
```

Potential failure domains include:

``` text
PVC
StorageClass
CSI controller
CSI node component
Cloud IAM
Storage API
Infrastructure capacity
Topology
Attachment
Mount
Filesystem
Application
```

Rule:

``` text
STORAGE FAILURE
≠
AUTOMATICALLY KUBERNETES FAILURE
```

------------------------------------------------------------------------

# 19. CSI Controller Health Is Not Enough

A CSI controller Pod can be Running while an operation fails because of:

``` text
Cloud IAM
Storage service authorization
Invalid parameters
Quota
Capacity
Topology
Backend API errors
```

Therefore:

``` text
CSI POD RUNNING
≠
CSI CLOUD AUTHORIZATION HEALTHY
```

And:

``` text
KUBERNETES RBAC SUCCESS
≠
CLOUD STORAGE AUTHORIZATION SUCCESS
```

The broader authorization model remains:

``` text
TrueFoundry Authorization
≠
Kubernetes RBAC
≠
Cloud IAM
```

------------------------------------------------------------------------

# 20. Attachment Is Conditional

Do not assume every CSI driver requires a separate attachment phase.

A more accurate model is:

``` text
PVC
→ PV
→ [Attachment when required]
→ Node Publish / Mount
→ Application
```

Therefore:

``` text
NO VolumeAttachment OBJECT
≠
STORAGE FAILURE
```

First establish whether attachment is expected for the driver and volume
architecture.

------------------------------------------------------------------------

# 21. VolumeAttachment

When attachment is used, Kubernetes can expose a `VolumeAttachment`
object representing an attach/detach operation involving a volume, CSI
attacher, and node.

Safe inspection can include:

``` bash
kubectl get volumeattachments
```

and:

``` bash
kubectl describe volumeattachment <name>
```

Relevant evidence can include:

``` text
PV
Node
Attacher
Attached state
Attach error
Detach error
```

Remember:

``` text
VOLUMEATTACHMENT EXISTS
≠
ATTACHMENT SUCCEEDED
```

------------------------------------------------------------------------

# 22. Multi-Attach Incidents

A block volume can encounter a multi-attach conflict when its access
semantics and current attachment state do not permit the requested new
attachment.

Investigate:

``` text
PVC
PV
Access mode
Current Pod
Old node
New Pod
New node
VolumeAttachment
Infrastructure attachment state
Workload controller
```

Critical rule:

``` text
MULTI-ATTACH ERROR
≠
PERMISSION TO FORCE DETACH
```

Before any approved force-detach, establish:

``` text
Is the old workload still writing?

Is the old node reachable?

Is the filesystem/application quiesced?

What access mode applies?

Could concurrent access corrupt data?

Who owns the application?

What is the recovery plan?

What is the rollback plan?
```

Force-detach is remediation, not diagnosis.

------------------------------------------------------------------------

# 23. Storage Topology

Storage can be constrained by:

``` text
Region
Availability Zone
Node
Storage pool
Backend topology
```

A production investigation should correlate:

``` text
Pod scheduling constraints
+
Selected node
+
Node topology
+
StorageClass topology
+
PV topology
+
Infrastructure volume topology
```

This is a conceptual troubleshooting model.

Do not assume that a healthy PV and a healthy node are compatible with
each other.

------------------------------------------------------------------------

# 24. `WaitForFirstConsumer` and Topology

`WaitForFirstConsumer` helps Kubernetes defer provisioning or binding
until scheduling constraints for the consuming Pod are available.

This can prevent premature volume placement that conflicts with workload
topology.

Therefore:

``` text
PVC PENDING
+
WaitForFirstConsumer
+
Unsatisfied Consumer Scheduling
```

can be expected behavior rather than storage failure.

Always evaluate the consumer Pod.

------------------------------------------------------------------------

# 25. CSI Storage Capacity

Some CSI environments expose capacity information to Kubernetes for
topology-aware scheduling.

When relevant, advanced diagnostics can include `CSIStorageCapacity`.

This can help distinguish:

``` text
PVC Pending
+
Valid StorageClass
+
Valid Topology
+
Insufficient Backend Capacity
```

from other provisioning failures.

Do not assume every CSI driver publishes this information.

------------------------------------------------------------------------

# 26. Workload-to-PVC Relationship

For a Pod:

``` bash
kubectl describe pod <pod> -n <namespace>
```

Correlate:

``` text
Pod
→ volume
→ PVC
→ PV
→ infrastructure storage
```

Rule:

``` text
PVC HEALTHY
≠
POD MOUNT HEALTHY
```

------------------------------------------------------------------------

# 27. Mount Failures

A Pod may be scheduled successfully but fail before application startup
because storage cannot be presented.

Possible causes include:

``` text
Attachment failure
CSI node failure
Filesystem problem
Topology issue
Backend unavailable
Permission/configuration issue
Stale attachment
Missing dependency
```

Pod events are often important evidence.

Rule:

``` text
POD PENDING
≠
SCHEDULER FAILURE
```

A Pod that already has a node but is waiting for storage is a different
failure boundary from a Pod that cannot be scheduled.

------------------------------------------------------------------------

# 28. Node Failure vs Storage Failure

When a workload cannot recover after a node failure:

``` text
NODE FAILURE
≠
STORAGE FAILURE
```

Investigate:

``` text
Old node
Old attachment
Detach state
New node
New attachment
CSI controller
CSI node component
Topology
Infrastructure volume
```

A storage system may be waiting for safe detach behavior before allowing
attachment elsewhere.

Do not convert a node failure into a data-integrity incident through
unsafe recovery actions.

------------------------------------------------------------------------

# 29. StatefulSets

Stateful workloads commonly use stable Pod identities and per-replica
PVCs.

Conceptually:

``` text
StatefulSet
   ↓
Pod-0 → PVC-0
Pod-1 → PVC-1
Pod-2 → PVC-2
```

Production invariant:

``` text
Pod ordinal
↔
PVC identity
↔
Application identity
```

Therefore:

``` text
POD-0 STORAGE
≠
POD-1 STORAGE
```

unless the architecture explicitly defines shared storage.

Never attach another replica's persistent data as an incident experiment
without an application-approved recovery procedure.

------------------------------------------------------------------------

# 30. Shared Storage

Some architectures use storage shared by multiple Pods or workloads.

Potential relationship:

``` text
Shared Storage Backend
        ↓
Multiple Pods
        ↓
Multiple Workloads
        ↓
Possibly Multiple Namespaces
```

Therefore:

``` text
ONE POD STORAGE SYMPTOM
≠
ONE-POD BLAST RADIUS
```

But the inverse is equally important:

``` text
ONE POD FAILURE
≠
SHARED STORAGE FAILURE
```

Prove the smallest supported scope.

------------------------------------------------------------------------

# 31. Reclaim Policy

Common reclaim policies include:

``` text
Delete
Retain
```

The reclaim policy can determine what happens to the backing storage
when the Kubernetes claim/PV lifecycle changes.

This makes deletion a data-lifecycle concern.

``` text
PVC DELETE
=
DATA-LIFECYCLE EVENT
```

Before any approved deletion, establish:

``` text
PVC
→ PV
→ Reclaim Policy
→ Infrastructure Asset
→ Data Criticality
→ Backup State
→ Restore Capability
→ Rollback
→ Approved Change
```

Rules:

``` text
PVC DELETE
≠
SAFE RESET

UNKNOWN RECLAIM BEHAVIOR
=
NO PVC DELETION
```

------------------------------------------------------------------------

# 32. Data Criticality Gate

Before potentially destructive storage remediation, classify the data.

Use:

``` text
Disposable
Reconstructable
Stateful
Unique
UNKNOWN
```

Examples:

``` text
Disposable cache
Re-downloadable model artifact
Temporary training workspace
Checkpoint
Application state
Unique production data
```

Critical rule:

``` text
UNKNOWN DATA CRITICALITY
=
NO DESTRUCTIVE STORAGE ACTION
```

------------------------------------------------------------------------

# 33. Persistence Is Not Backup

A persistent volume protects data across some Pod lifecycle events.

It does not automatically protect against:

``` text
Application deletion
Operator error
Filesystem corruption
Storage corruption
Incorrect PVC deletion
Ransomware
Credential misuse
Region failure
Logical data corruption
```

Therefore:

``` text
PERSISTENCE
≠
BACKUP

SNAPSHOT EXISTS
≠
RECOVERY VALIDATED

BACKUP COMPLETED
≠
RESTORE TESTED
```

Backup and disaster recovery require separate controls.

------------------------------------------------------------------------

# 34. Volume Expansion

Some StorageClasses and CSI drivers support expanding volumes.

A production expansion path can be:

``` text
Requested Size
        ↓
PVC Updated
        ↓
Infrastructure Volume Expanded
        ↓
Filesystem Expanded
        ↓
Application Sees Capacity
```

Rules:

``` text
ALLOW VOLUME EXPANSION
≠
EXPANSION COMPLETED

VOLUME EXPANSION
≠
VOLUME SHRINK

PVC SIZE UPDATED
≠
BACKEND EXPANSION COMPLETE

BACKEND EXPANSION COMPLETE
≠
FILESYSTEM EXPANSION COMPLETE

FILESYSTEM EXPANSION COMPLETE
≠
APPLICATION VALIDATION COMPLETE
```

Expansion is a change operation and should follow normal change
controls.

------------------------------------------------------------------------

# 35. StorageClass Change Is an Architecture Change

A different StorageClass can imply different:

``` text
Storage technology
Performance characteristics
Availability behavior
Topology
Provisioner
Encryption
Reclaim behavior
Cost
```

Therefore:

``` text
STORAGECLASS CHANGE
=
ARCHITECTURE + DATA + AVAILABILITY + COST EVENT
```

Do not switch StorageClasses as incident experimentation.

------------------------------------------------------------------------

# 36. Capacity Has Multiple Layers

Do not use the phrase "disk size" without identifying which layer is
being discussed.

Model capacity as:

``` text
PVC requested capacity
        ↓
PV capacity
        ↓
Infrastructure volume capacity
        ↓
Filesystem capacity
        ↓
Filesystem free bytes
        ↓
Filesystem free inodes
        ↓
Application quota
        ↓
Application-consumable capacity
```

Rules:

``` text
PVC SIZE
≠
FILESYSTEM FREE SPACE

VOLUME SIZE
≠
APPLICATION-USABLE CAPACITY
```

------------------------------------------------------------------------

# 37. Filesystem Full

A volume can remain:

``` text
Bound
Attached
Mounted
```

while its filesystem has no useful free capacity.

Therefore:

``` text
PVC BOUND
+
POD RUNNING
≠
STORAGE CAPACITY HEALTHY
```

Possible symptoms include:

``` text
Write failures
Database failures
Application errors
Crash loops
Partial transactions
Log-write failures
Checkpoint failures
```

------------------------------------------------------------------------

# 38. Inode Exhaustion

A filesystem can have free bytes but be unable to create more files
because inode capacity is exhausted.

Therefore:

``` text
FREE DISK SPACE
≠
FILESYSTEM CAN CREATE FILES
```

This is particularly relevant for workloads producing very large numbers
of small files.

Capacity investigations should distinguish:

``` text
Byte capacity
Inode capacity
Application quota
```

------------------------------------------------------------------------

# 39. Storage Performance

Persistent storage can be functionally available but too slow for the
workload.

Relevant dimensions include:

``` text
Latency
IOPS
Throughput
Queue depth
Throttling
Burst limits
Application I/O pattern
```

Conceptually:

``` text
Application
   ↓
Filesystem / Block I/O
   ↓
Node Storage Path
   ↓
CSI / Storage Layer
   ↓
Infrastructure Volume
   ↓
Storage Service Limits
```

Rules:

``` text
MOUNTED
≠
PERFORMANT

LOW CPU
≠
NO STORAGE BOTTLENECK

APPLICATION SLOW
≠
STORAGE ROOT CAUSE
```

------------------------------------------------------------------------

# 40. Performance Correlation

Storage performance becomes stronger incident evidence when correlated
with:

``` text
Application request latency
Application I/O
Storage latency
IOPS
Throughput
Queueing
Throttle events
Incident timestamp
```

Rule:

``` text
TIMING CORRELATION
≠
ROOT CAUSE PROOF
```

Do not attribute application latency to storage merely because storage
metrics changed at a similar time.

------------------------------------------------------------------------

# 41. AI Workload Storage

AI platforms can use persistent or external storage for:

``` text
Model artifacts
Model weights
Datasets
Training data
Caches
Checkpoints
Experiment artifacts
Shared files
Application state
```

These have different durability and performance requirements.

For example:

``` text
Authoritative Model Source
        ↓
Download
        ↓
Local Cache
        ↓
Model Runtime
```

is operationally different from:

``` text
Stateful Application
        ↓
Persistent Database Files
```

The first investigation question should be:

``` text
WHAT DATA DOES THIS STORAGE ACTUALLY CONTAIN?
```

------------------------------------------------------------------------

# 42. Security and Sensitive Data

Persistent volumes can contain sensitive application data.

Troubleshooting must not become data extraction.

Do not:

``` text
Copy production files unnecessarily
Dump database files
Copy model credentials
Extract private keys
Decode Kubernetes Secrets
Copy mounted Secret material
Write test data to production volumes
```

Critical rule:

``` text
READ-ONLY KUBERNETES COMMAND
≠
PERMISSION TO READ APPLICATION DATA
```

Use metadata and control-plane evidence first.

------------------------------------------------------------------------

# 43. Do Not Write Test Files in Production

A command such as:

``` text
touch /data/test
```

modifies application storage.

It is not part of a `[SAFE-READ]` production investigation.

Use:

``` text
Existing application telemetry
Existing health checks
Existing logs
Storage metrics
Filesystem metadata from approved tooling
Non-production validation
```

instead.

------------------------------------------------------------------------

# 44. Desired-State Ownership

A Kubernetes PVC can be rendered from:

``` text
TrueFoundry
GitOps
Helm
Kubernetes manifests
Operator
Terraform / OpenTofu
Application repository
Platform automation
```

Therefore:

``` text
OBSERVED PVC
≠
AUTHORITATIVE CONFIGURATION SOURCE
```

And:

``` text
Observed Drift
≠
Permission to Patch
```

Do not patch the observed Kubernetes object when another system owns
desired state.

``` text
DO NOT PATCH THE SYMPTOM
WHEN ANOTHER SYSTEM OWNS DESIRED STATE
```

------------------------------------------------------------------------

# 45. Production Troubleshooting Sequence

Use:

``` text
Capture Original Symptom
        ↓
Identify Workload
        ↓
Prove Storage Identity
        ↓
Identify Volume Reference
        ↓
Identify PVC
        ↓
Classify PVC State
        ↓
Identify PV
        ↓
Identify StorageClass / Static Provisioning
        ↓
Identify CSI Driver
        ↓
Validate Provisioning
        ↓
Validate Topology
        ↓
Validate Attachment When Required
        ↓
Validate Node Presentation
        ↓
Validate Filesystem / Block Device
        ↓
Validate Application Access
        ↓
Validate Capacity
        ↓
Validate Inodes
        ↓
Validate Performance
        ↓
Validate Expected Data Identity
```

Never begin with:

``` text
Delete PVC and retry.
```

------------------------------------------------------------------------

# 46. Failure Domains

Potential first-failure domains include:

``` text
TrueFoundry workload configuration

Rendered Kubernetes workload

PVC

StorageClass

Static PV selection

CSI controller

Cloud IAM

Infrastructure storage API

PersistentVolume

Scheduling

Topology

Volume attachment

CSI node path

Filesystem mount

Raw block presentation

Filesystem capacity

Inode capacity

Storage performance

Application permissions

Application path

Application storage behavior

External storage dependency
```

Do not classify the root cause more broadly than the evidence supports.

------------------------------------------------------------------------

# 47. Lowest Proven Healthy Layer

Example:

``` text
PVC Bound                         PROVEN
PV exists                         PROVEN
Infrastructure volume exists      PROVEN
Volume attached                   PROVEN
Filesystem mounted                UNKNOWN
Application write succeeds        UNKNOWN
```

Then:

``` text
Lowest Proven Healthy Layer:
Volume attachment
```

The next investigation boundary is:

``` text
Volume Attached
→
Filesystem Mounted
```

------------------------------------------------------------------------

# 48. First Failed Transition

Examples:

``` text
PVC
→
Provisioner
```

``` text
Provisioner
→
Infrastructure Volume
```

``` text
PV
→
Volume Attachment
```

``` text
Volume Attached
→
Filesystem Mounted
```

``` text
Filesystem Mounted
→
Application Access
```

``` text
Application Access
→
Expected Data
```

Record the transition.

Avoid vague conclusions such as:

``` text
Storage problem
Kubernetes problem
TrueFoundry problem
Cloud problem
```

unless that scope is actually proven.

------------------------------------------------------------------------

# 49. Minimum Supported Blast Radius

Possible scopes include:

``` text
One mount
One Pod
One PVC
One PV
One StatefulSet replica
One node
One Availability Zone
One StorageClass
One CSI driver
One shared filesystem
One storage backend
One namespace
One cluster
Multiple clusters
```

Rules:

``` text
ONE PVC FAILURE
≠
CLUSTER STORAGE FAILURE

ONE POD FAILURE
≠
SHARED STORAGE FAILURE

ONE POD STORAGE SYMPTOM
≠
ONE-POD BLAST RADIUS
```

Use the smallest evidence-supported scope.

------------------------------------------------------------------------

# 50. Evidence Confidence

Use:

``` text
PROVEN
SUPPORTED
UNKNOWN
ASSUMED
```

Example:

``` text
PVC is Bound                         PROVEN
PV uses expected StorageClass        PROVEN
Pod reports FailedMount              PROVEN
CSI node path is implicated          SUPPORTED
Underlying disk is corrupted         UNKNOWN
```

Rule:

``` text
ASSUMED
≠
ROOT CAUSE
```

------------------------------------------------------------------------

# 51. Current Actionable Owner

Use:

``` text
First Failed Transition
+
Authoritative Configuration Owner
+
Operational Boundary
=
Current Actionable Owner
```

Possible owners can include:

``` text
Application team
TrueFoundry/platform team
Kubernetes/SRE team
Cloud infrastructure team
Storage team
Security/IAM team
CSI/add-on owner
```

Architecture does not automatically define organizational ownership.

The owner should be determined by the operating model and the first
proven failed transition.

------------------------------------------------------------------------

# 52. Evidence Preservation

Before state-changing remediation, preserve relevant evidence.

Capture as appropriate:

``` text
Original symptom
Timestamp
Workload identity
Pod UID
Node
PVC metadata
PVC UID
PV metadata
StorageClass
CSI driver
VolumeAttachment status
Pod events
Node placement
Infrastructure volume identity
Capacity evidence
Inode evidence
Performance evidence
Relevant safe logs
```

Rule:

``` text
RESTART OR RESCHEDULE
CAN DESTROY INCIDENT EVIDENCE
```

Preserve evidence before remediation whenever operationally practical.

------------------------------------------------------------------------

# 53. Scenario 1 --- PVC Pending with WaitForFirstConsumer

Symptom:

``` text
PVC:
Pending
```

Do not immediately classify this as provisioning failure.

Investigate:

``` text
StorageClass
volumeBindingMode
Consumer Pod
Scheduling state
Node constraints
Topology
```

Possible result:

``` text
PVC Pending                         PROVEN
WaitForFirstConsumer                PROVEN
No schedulable consumer yet         PROVEN
Provisioning failure                NOT PROVEN
```

Rule:

``` text
EXPECTED PENDING
≠
STORAGE FAILURE
```

------------------------------------------------------------------------

# 54. Scenario 2 --- PVC Pending Because Provisioning Failed

Evidence can include:

``` text
PVC Pending
Provisioning events
CSI/provisioner errors
StorageClass
Topology
Cloud/storage authorization
Backend capacity
```

Determine:

``` text
Lowest Proven Healthy Layer:

First Failed Transition:

Minimum Supported Blast Radius:

Evidence Confidence:
```

Do not delete/recreate the PVC as the first response.

------------------------------------------------------------------------

# 55. Scenario 3 --- PVC Bound but Pod Reports FailedMount

Evidence:

``` text
PVC Bound
PV Bound
Pod assigned to node
Pod event reports mount failure
```

Investigate:

``` text
Attachment when applicable
CSI node path
Filesystem
Node
Pod events
Backend availability
```

Rule:

``` text
PVC BOUND
≠
MOUNT HEALTHY
```

------------------------------------------------------------------------

# 56. Scenario 4 --- Multi-Attach Conflict

Evidence:

``` text
Volume already attached
New Pod scheduled elsewhere
Attach operation denied
```

Investigate:

``` text
Old Pod
Old node
New Pod
New node
Access mode
VolumeAttachment
Infrastructure attachment state
Application write state
```

Do not force-detach as diagnostic experimentation.

``` text
MULTI-ATTACH ERROR
≠
PERMISSION TO FORCE DETACH
```

------------------------------------------------------------------------

# 57. Scenario 5 --- Wrong StorageClass or Storage Profile

Compare:

``` text
Desired configuration
Rendered PVC
Effective StorageClass
Bound PV
Infrastructure volume
```

Potential consequences can include:

``` text
Different performance
Different topology
Different cost
Different reclaim behavior
Different availability characteristics
```

Identify authoritative configuration before changing anything.

------------------------------------------------------------------------

# 58. Scenario 6 --- Volume Full

A workload can have:

``` text
PVC Bound
Volume Attached
Filesystem Mounted
Pod Running
```

and still fail because the filesystem is full.

Differentiate:

``` text
PVC requested capacity
PV capacity
Infrastructure capacity
Filesystem capacity
Filesystem free bytes
Application quota
```

Do not assume PVC expansion alone completes remediation.

------------------------------------------------------------------------

# 59. Scenario 7 --- Inode Exhaustion

Symptom:

``` text
Application cannot create files
```

while byte capacity still appears available.

Investigate:

``` text
Filesystem free bytes
Filesystem inode usage
Application file pattern
```

Rule:

``` text
FREE BYTES
≠
FILESYSTEM CAN CREATE FILES
```

------------------------------------------------------------------------

# 60. Scenario 8 --- Mounted Volume but Application Permission Failure

Evidence:

``` text
Volume mounted
Container running
Application reports permission denied
```

Investigate:

``` text
Container identity
Security context
Filesystem ownership
Expected application path
Mount mode
Application permissions
```

Do not automatically classify this as a CSI failure.

------------------------------------------------------------------------

# 61. Scenario 9 --- Pod Rescheduled but Storage Does Not Recover

Investigate:

``` text
Old node
Old attachment
Detach progress
New node
New attachment
CSI
Topology
Infrastructure volume state
```

Rule:

``` text
POD RESCHEDULED
≠
VOLUME IMMEDIATELY AVAILABLE
```

------------------------------------------------------------------------

# 62. Scenario 10 --- Storage Latency Causes Application Slowness

Correlate:

``` text
Application latency
I/O latency
IOPS
Throughput
Queue depth
Throttle events
Backend limits
Incident timestamp
```

Rule:

``` text
MOUNTED
≠
PERFORMANT
```

But also:

``` text
APPLICATION SLOW
≠
STORAGE ROOT CAUSE
```

Prove the correlation.

------------------------------------------------------------------------

# 63. Scenario 11 --- One StatefulSet Replica Has Storage Failure

Correlate:

``` text
StatefulSet
Pod ordinal
PVC
PV
Node
Storage backend
Application replica
```

Do not automatically expand the blast radius to every StatefulSet
replica.

``` text
ONE REPLICA STORAGE FAILURE
≠
STATEFULSET-WIDE STORAGE FAILURE
```

------------------------------------------------------------------------

# 64. Scenario 12 --- PVC Deletion Proposed as Remediation

Stop and establish:

``` text
Data criticality
PVC identity
PV identity
Reclaim policy
Infrastructure volume
Backup state
Restore capability
Recovery procedure
Authoritative owner
Approved change
```

Rule:

``` text
PVC DELETE
≠
SAFE RESET
```

------------------------------------------------------------------------

# 65. Scenario 13 --- Volume Expansion Incomplete

Validate:

``` text
StorageClass allows expansion
PVC requested size
PV/backend size
Filesystem size
Application-visible capacity
```

Classify the failed transition.

Possible example:

``` text
PVC request updated                  PROVEN
Backend volume expanded              PROVEN
Filesystem still old size            PROVEN
Application still capacity constrained PROVEN
```

Then:

``` text
First Failed Transition:
Backend Expansion
→
Filesystem Expansion
```

------------------------------------------------------------------------

# 66. Scenario 14 --- Storage Exists but Expected Data Is Missing

Do not immediately conclude data loss.

Prove:

``` text
Environment
Workload
Pod
PVC UID
PV
Infrastructure volume
Mount path
Application path
Replica identity
```

Possible causes include:

``` text
Wrong environment
Wrong PVC
Recreated PVC
Wrong PV
Wrong mount path
Wrong Pod
Wrong StatefulSet ordinal
Application path change
Empty replacement storage
Actual deletion
Corruption
Restore failure
```

Rule:

``` text
DATA NOT VISIBLE
≠
DATA LOST
```

------------------------------------------------------------------------

# 67. Production Evidence Handoff Contract

Use:

``` text
Environment:

TrueFoundry Workspace:

Cluster:

Namespace:

Workload:

Workload Type:

Pod:

Pod UID:

Node:

Mount Path:

Volume Mode:
Filesystem / Block

PVC:

PVC UID:

PVC Phase:

Requested Capacity:

Access Mode:

StorageClass:

Volume Binding Mode:

PV:

Reclaim Policy:

CSI Driver:

Infrastructure Volume:
If safely identifiable

Attachment Required:
YES / NO / UNKNOWN

VolumeAttachment:
If applicable

Attachment Status:

Mount Status:

Filesystem Status:

Filesystem Capacity:

Filesystem Free Capacity:

Inode Status:
If applicable

Storage Performance:

Data Criticality:
Disposable / Reconstructable / Stateful / Unique / UNKNOWN

Backup Available:
YES / NO / UNKNOWN

Restore Validated:
YES / NO / UNKNOWN

Original Symptom:

Relevant Events:

Relevant Timestamps:

Lowest Proven Healthy Layer:

First Failed Transition:

Minimum Supported Blast Radius:

Evidence Confidence:
PROVEN / SUPPORTED / UNKNOWN / ASSUMED

Authoritative Configuration Source:

Authoritative Configuration Owner:

Current Actionable Owner:

Requested Action:

PVC Modified:
NO

PVC Deleted:
NO

PV Modified:
NO

Volume Force-Detached:
NO

Application Data Modified:
NO

Secret Decoded:
NO

Pod Restarted:
NO
```

------------------------------------------------------------------------

# 68. Production Safety Rules

``` text
PVC EXISTS
≠
PVC BOUND

PVC BOUND
≠
VOLUME ATTACHED

VOLUME ATTACHED
≠
VOLUME MOUNTED

VOLUME MOUNTED
≠
APPLICATION STORAGE HEALTHY

MOUNTED
≠
PERFORMANT

PVC PENDING
≠
PVC BROKEN

ReadWriteOnce
≠
Exactly One Pod

ACCESS MODE
≠
COMPLETE STORAGE TOPOLOGY

PERSISTENT VOLUME
≠
ALWAYS A FILESYSTEM

NO VolumeAttachment OBJECT
≠
STORAGE FAILURE

DATA NOT VISIBLE
≠
DATA LOST

RESOURCE NAME
≠
RESOURCE IDENTITY

PERSISTENCE
≠
BACKUP

SNAPSHOT EXISTS
≠
RECOVERY VALIDATED

BACKUP COMPLETED
≠
RESTORE TESTED

PVC DELETE
≠
SAFE RESET

MULTI-ATTACH ERROR
≠
PERMISSION TO FORCE DETACH

FREE DISK SPACE
≠
FILESYSTEM CAN CREATE FILES

PVC SIZE
≠
FILESYSTEM FREE SPACE

STORAGECLASS EXISTS IN KUBERNETES
≠
STORAGECLASS AVAILABLE FOR TRUEFOUNDRY VOLUME CREATION

TRUEFOUNDRY VOLUME CONFIGURATION
≠
SUCCESSFUL KUBERNETES PROVISIONING

CSI POD RUNNING
≠
CSI CLOUD AUTHORIZATION HEALTHY

TrueFoundry Authorization
≠
Kubernetes RBAC
≠
Cloud IAM

UNKNOWN DATA CRITICALITY
=
NO DESTRUCTIVE STORAGE ACTION

UNKNOWN RECLAIM BEHAVIOR
=
NO PVC DELETION

STORAGECLASS CHANGE
=
ARCHITECTURE + DATA + AVAILABILITY + COST EVENT

OBSERVED PVC
≠
AUTHORITATIVE CONFIGURATION SOURCE

Observed Drift
≠
Permission to Patch

READ-ONLY KUBERNETES COMMAND
≠
PERMISSION TO READ APPLICATION DATA

RESTART RESTORED SERVICE
≠
ROOT CAUSE IDENTIFIED

SERVICE RESTORED
≠
ROOT CAUSE REMEDIATED

TIMING CORRELATION
≠
ROOT CAUSE PROOF

ASSUMED
≠
ROOT CAUSE

SYMPTOM
≠
FAILURE DOMAIN
≠
OWNER
```

------------------------------------------------------------------------

# 69. Production Investigation Checklist

Before closing a persistent-storage incident, answer:

``` text
What was the original symptom?

What is the workload?

What is the exact storage identity?

What PVC UID is involved?

What PV is involved?

What infrastructure volume is involved?

What StorageClass is effective?

Was provisioning expected?

What is the PVC phase?

Is WaitForFirstConsumer relevant?

What access mode applies?

What volume mode applies?

Does attachment apply to this CSI driver?

Did attachment succeed?

Did node presentation/mount succeed?

Is the correct Pod using the correct volume?

Is the correct application path being used?

Is filesystem capacity healthy?

Are inodes healthy?

Is storage performance healthy?

What data does the volume contain?

What is the data criticality?

Is backup available?

Has restore capability been validated?

What is the Lowest Proven Healthy Layer?

What is the First Failed Transition?

What is the Minimum Supported Blast Radius?

What is the Evidence Confidence?

Who owns authoritative configuration?

Who is the Current Actionable Owner?

What action is requested next?
```

------------------------------------------------------------------------

# 70. Final Principle

Persistent storage incidents require more caution than ordinary
stateless workload failures because an unsafe action can convert an
availability incident into a data-integrity incident.

The production method is:

``` text
PROVE STORAGE IDENTITY FIRST.

THEN PROVE:

PROVISIONING
→ TOPOLOGY
→ ATTACHMENT WHEN REQUIRED
→ NODE PRESENTATION
→ APPLICATION ACCESS
→ CAPACITY
→ PERFORMANCE
→ DATA INTEGRITY
```

Never use deletion, force-detach, StorageClass changes, volume
expansion, Pod restart, or data modification merely to test a storage
hypothesis.

``` text
OBSERVE FIRST.
PROVE THE FAILED TRANSITION.
PROTECT THE DATA.
IDENTIFY THE AUTHORITATIVE OWNER.
THEN USE AN APPROVED REMEDIATION.
```
