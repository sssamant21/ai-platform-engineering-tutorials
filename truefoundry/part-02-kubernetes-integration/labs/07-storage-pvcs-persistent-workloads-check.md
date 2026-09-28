# Part 2.7 Lab --- Storage, PVCs & Persistent Workloads Check

> **\[SAFE-READ\] Production Validation Lab**

## Purpose

This lab validates the persistent-storage path used by a TrueFoundry
workload on Kubernetes without modifying production state or application
data.

``` text
TrueFoundry / Authoritative Configuration
→ Rendered Kubernetes Workload
→ PVC
→ StorageClass / Static PV Selection
→ CSI Provisioning
→ PersistentVolume
→ Infrastructure Storage
→ [Volume Attachment — when required]
→ Filesystem Mount OR Raw Block Device
→ Container
→ Application Storage Usage
→ Application Data
```

The goal is to prove storage identity, the lowest healthy layer, the
first failed transition, the smallest supported blast radius, data
criticality, and the authoritative owner.

``` text
DO NOT TROUBLESHOOT "STORAGE" AS A SINGLE COMPONENT.
PROVE THE STORAGE PATH ONE TRANSITION AT A TIME.
```

------------------------------------------------------------------------

# 1. Safety Contract

This is a read-only production validation lab.

Allowed examples:

``` bash
kubectl config current-context
kubectl cluster-info
kubectl get
kubectl describe
kubectl logs
```

Do not run or perform:

``` text
kubectl create/apply/delete/edit/patch/replace/scale
kubectl rollout restart
helm install/upgrade/uninstall
PVC/PV deletion or modification
PVC resize
StorageClass changes
Reclaim-policy changes
Force detach/manual attach
Manual mount/unmount
Filesystem repair
Pod deletion/restart
CSI restart
Production debug-Pod creation
Test writes
Application-data copying
Secret decoding
Snapshot creation/deletion/restore
Cloud IAM changes
Infrastructure-volume changes
```

Rules:

``` text
UNKNOWN TARGET = NO MUTATION
UNKNOWN STORAGE IDENTITY = NO STORAGE CHANGE
UNKNOWN DATA CRITICALITY = NO DESTRUCTIVE STORAGE ACTION
UNKNOWN RECLAIM BEHAVIOR = NO PVC DELETION
Observed Drift ≠ Permission to Patch
```

------------------------------------------------------------------------

# 2. Variables and Original Symptom

Record:

``` text
Environment:
TrueFoundry Workspace:
Kubernetes Context:
Namespace:
Workload:
Workload Type:
Pod:
Container:
PVC:
PV:
StorageClass:
CSI Driver:

Observation Point:
Incident Timestamp:
Original Error:
Expected Behavior:
Observed Behavior:
```

Do not translate the original error into a presumed cause.

------------------------------------------------------------------------

# 3. Verify Context and Namespace

``` bash
kubectl config current-context
kubectl cluster-info
kubectl get namespace <namespace>
```

Record expected versus observed context.

If context is wrong, stop.

``` text
KUBERNETES API REACHABLE ≠ STORAGE PATH HEALTHY
NAMESPACE EXISTS ≠ WORKLOAD STORAGE HEALTHY
```

------------------------------------------------------------------------

# 4. Identify Workload and Pod

Examples:

``` bash
kubectl get deployment -n <namespace>
kubectl get statefulset -n <namespace>
kubectl get pods -n <namespace> -o wide
kubectl describe pod <pod> -n <namespace>
```

Record:

``` text
Workload:
Type:
Pod:
Node:
Phase:
Ready:
Restarts:
Volume references:
Mount paths/devices:
Storage events:
```

Capture Pod UID:

``` bash
kubectl get pod <pod> -n <namespace> -o jsonpath="{.metadata.uid}"
```

Rule:

``` text
RESOURCE NAME ≠ RESOURCE IDENTITY
```

------------------------------------------------------------------------

# 5. Identify PVC and Exact Identity

``` bash
kubectl get pvc -n <namespace>
kubectl describe pvc <pvc> -n <namespace>
kubectl get pvc <pvc> -n <namespace> -o jsonpath="{.metadata.uid}"
```

Record:

``` text
PVC:
PVC UID:
Phase:
Requested Capacity:
Access Mode:
Volume Mode:
StorageClass:
Bound PV:
Events:
```

Rules:

``` text
PVC EXISTS ≠ PVC BOUND
PVC BOUND ≠ STORAGE USABLE
SAME PVC NAME ≠ SAME STORAGE IDENTITY
```

Classify Pending PVCs as:

``` text
Expected Pending / Abnormal Pending / UNKNOWN
```

Do not treat `WaitForFirstConsumer` Pending as automatic failure.

------------------------------------------------------------------------

# 6. Inspect StorageClass

``` bash
kubectl get storageclass
kubectl describe storageclass <storage-class>
```

Record:

``` text
Provisioner:
Reclaim Policy:
Volume Binding Mode:
Allow Volume Expansion:
Relevant Parameters:
Default:
```

Rules:

``` text
STORAGECLASS EXISTS ≠ PROVISIONER FUNCTIONAL
NO EXPLICIT storageClassName ≠ NO STORAGECLASS
```

For TrueFoundry-managed volume workflows also distinguish:

``` text
STORAGECLASS EXISTS IN KUBERNETES
≠
STORAGECLASS AVAILABLE FOR TRUEFOUNDRY VOLUME CREATION

TRUEFOUNDRY VOLUME CONFIGURATION
≠
SUCCESSFUL KUBERNETES PROVISIONING
```

------------------------------------------------------------------------

# 7. WaitForFirstConsumer

If binding mode is `WaitForFirstConsumer`, record:

``` text
Consumer Pod Exists:
Consumer Scheduling State:
Selected Node:
Topology Constraints:
Provisioning Error Present:
```

Rule:

``` text
PVC PENDING + WaitForFirstConsumer
≠
PROVISIONING FAILURE BY ITSELF
```

------------------------------------------------------------------------

# 8. Inspect PersistentVolume

``` bash
kubectl get pv <pv>
kubectl describe pv <pv>
```

Record:

``` text
PV:
Status:
Capacity:
Access Modes:
Volume Mode:
Reclaim Policy:
StorageClass:
Claim Reference:
CSI Driver:
volumeHandle / Infrastructure Identifier:
Node Affinity:
```

Prove:

``` text
Workload → Pod → PVC UID → PV → CSI volumeHandle / infrastructure volume
```

Do not rely on naming conventions.

------------------------------------------------------------------------

# 9. Access and Volume Modes

Record:

``` text
Access Mode:
ReadWriteOnce / ReadOnlyMany / ReadWriteMany / ReadWriteOncePod / Other

Volume Mode:
Filesystem / Block
```

Rules:

``` text
ReadWriteOnce ≠ Exactly One Pod
ReadWriteOncePod ≠ ReadWriteOnce
ACCESS MODE ≠ COMPLETE STORAGE TOPOLOGY
PERSISTENT VOLUME ≠ ALWAYS A FILESYSTEM
```

If Block mode is used, do not apply filesystem assumptions.

------------------------------------------------------------------------

# 10. CSI Boundary

Identify the CSI driver from PV/StorageClass.

Record:

``` text
CSI Driver:
Controller Component:
If known
Node Component:
If known
Namespace:
If known
```

If exact components are known and policy permits, inspect relevant logs
only:

``` bash
kubectl logs <csi-pod> -n <csi-namespace> -c <container>
```

Capture minimum relevant evidence.

Rules:

``` text
CSI POD RUNNING ≠ CSI CLOUD AUTHORIZATION HEALTHY
KUBERNETES RBAC SUCCESS ≠ CLOUD STORAGE AUTHORIZATION SUCCESS

TrueFoundry Authorization
≠
Kubernetes RBAC
≠
Cloud IAM
```

Do not change IAM.

------------------------------------------------------------------------

# 11. Determine Whether Attachment Applies

Record:

``` text
Attachment Required:
YES / NO / UNKNOWN

Evidence:
```

Do not assume every CSI driver requires attachment.

``` text
NO VolumeAttachment OBJECT ≠ STORAGE FAILURE
```

If attachment is required:

``` bash
kubectl get volumeattachments
kubectl describe volumeattachment <volume-attachment>
```

Record:

``` text
VolumeAttachment:
PV:
Node:
Attacher:
Attached:
Attach Error:
Detach Error:
```

Rule:

``` text
VOLUMEATTACHMENT EXISTS ≠ ATTACHMENT SUCCEEDED
```

------------------------------------------------------------------------

# 12. Multi-Attach Safety

If multi-attach is reported, record:

``` text
PVC:
PV:
Access Mode:
Old Pod:
Old Node:
New Pod:
New Node:
VolumeAttachment:
Infrastructure Attachment:
Application Write State:
```

Rule:

``` text
MULTI-ATTACH ERROR ≠ PERMISSION TO FORCE DETACH
```

Force detach is prohibited in this lab.

------------------------------------------------------------------------

# 13. Topology Validation

``` bash
kubectl get pod <pod> -n <namespace> -o wide
kubectl get node <node> --show-labels
```

Capture only relevant topology labels.

Compare:

``` text
Pod Scheduling Constraints
Selected Node
Node Region/Zone
StorageClass Topology
PV Node Affinity
Infrastructure Volume Topology
```

Record:

``` text
Topology Compatible:
YES / NO / UNKNOWN
```

------------------------------------------------------------------------

# 14. Events

``` bash
kubectl describe pod <pod> -n <namespace>
kubectl get events -n <namespace> --sort-by=.metadata.creationTimestamp
```

Relevant reasons can include:

``` text
FailedScheduling
FailedAttachVolume
FailedMount
SuccessfulAttachVolume
MountVolume
Provisioning errors
```

Record timestamp and exact object.

``` text
TIMING CORRELATION ≠ ROOT CAUSE PROOF
```

------------------------------------------------------------------------

# 15. Four Diagnostic Gates

Classify each:

## Gate 1 --- Provisioning

``` text
PVC → StorageClass → CSI/Provisioner → PV → Infrastructure Storage

PASS / FAIL / UNKNOWN
```

## Gate 2 --- Placement / Attachment

``` text
Consumer Pod → Scheduler → Node/Topology → [VolumeAttachment]

PASS / FAIL / UNKNOWN / NOT APPLICABLE
```

## Gate 3 --- Node Presentation

``` text
Infrastructure Storage → CSI Node Path → Filesystem/Block Device → Container

PASS / FAIL / UNKNOWN
```

## Gate 4 --- Application

``` text
Container-visible Storage → Permissions → Expected Path → I/O → Data

PASS / FAIL / UNKNOWN
```

Central rule:

``` text
PVC BOUND
≠
ATTACHMENT HEALTHY
≠
MOUNT HEALTHY
≠
APPLICATION STORAGE HEALTHY
```

------------------------------------------------------------------------

# 16. Application Storage Access

Use existing safe evidence:

``` text
Application logs
Metrics
Health telemetry
Approved read-only health endpoint
Existing observability
```

Record:

``` text
Expected Path/Device:
Storage Presented:
Application Access:
Application Permission Error:
```

Do not create test files.

``` text
TEST WRITE = PRODUCTION DATA MUTATION
```

Never use `touch`, `echo >`, `mkdir`, `rm`, or equivalent to prove
storage health in this lab.

------------------------------------------------------------------------

# 17. Sensitive Data Guardrail

Do not copy or inspect:

``` text
Database files
Business/customer records
Model credentials
Private keys
Application secrets
Mounted Secret content
```

Rule:

``` text
READ-ONLY KUBERNETES COMMAND
≠
PERMISSION TO READ APPLICATION DATA
```

Use metadata/control-plane evidence first.

------------------------------------------------------------------------

# 18. Capacity Model

Distinguish:

``` text
PVC Requested Capacity
PV Capacity
Infrastructure Volume Capacity
Filesystem Capacity
Filesystem Free Bytes
Filesystem Free Inodes
Application Quota
Application-Consumable Capacity
```

Record only evidence available through approved observability.

Rules:

``` text
PVC SIZE ≠ FILESYSTEM FREE SPACE
VOLUME SIZE ≠ APPLICATION-USABLE CAPACITY
```

No resize is performed.

------------------------------------------------------------------------

# 19. Inode Validation

If approved metrics expose inode state:

``` text
Filesystem Free Bytes:
Filesystem Free Inodes:
Application File-Creation Error:
```

Rule:

``` text
FREE DISK SPACE ≠ FILESYSTEM CAN CREATE FILES
```

Do not create files to test inode availability.

------------------------------------------------------------------------

# 20. Performance Validation

Collect existing metrics when relevant:

``` text
Application Latency:
Storage Latency:
IOPS:
Throughput:
Queue Depth:
Throttle Events:
Observation Interval:
```

Rules:

``` text
MOUNTED ≠ PERFORMANT
LOW CPU ≠ NO STORAGE BOTTLENECK
APPLICATION SLOW ≠ STORAGE ROOT CAUSE
```

Correlate application and storage evidence before attributing causality.

------------------------------------------------------------------------

# 21. StatefulSet Identity

For StatefulSets:

``` bash
kubectl get statefulset <workload> -n <namespace>
kubectl get pods -n <namespace> -o wide
kubectl get pvc -n <namespace>
```

Build:

``` text
Replica | PVC | PV | Node | Storage State | Application State
--------|-----|----|------|---------------|------------------
        |     |    |      |               |
```

Rule:

``` text
POD-0 STORAGE ≠ POD-1 STORAGE
ONE REPLICA STORAGE FAILURE ≠ STATEFULSET-WIDE STORAGE FAILURE
```

------------------------------------------------------------------------

# 22. Shared Storage Blast Radius

Record:

``` text
Shared Storage:
YES / NO / UNKNOWN

Known Consumers:
Namespaces:
Workloads:
Inventory Complete:
YES / NO / UNKNOWN
```

Rules:

``` text
ONE POD STORAGE SYMPTOM ≠ ONE-POD BLAST RADIUS
ONE POD FAILURE ≠ SHARED STORAGE FAILURE
```

------------------------------------------------------------------------

# 23. Data Criticality Gate

Classify without unnecessarily reading data:

``` text
Disposable
Reconstructable
Stateful
Unique
UNKNOWN
```

Record:

``` text
Data Criticality:
Evidence:
Application Owner:
```

Rule:

``` text
UNKNOWN DATA CRITICALITY
=
NO DESTRUCTIVE STORAGE ACTION
```

------------------------------------------------------------------------

# 24. Reclaim Policy and Deletion Gate

Record:

``` text
Reclaim Policy:
Delete / Retain / Other / UNKNOWN

Infrastructure Volume Identified:
Backup Available:
Restore Validated:
Authoritative Owner:
```

Rules:

``` text
PVC DELETE = DATA-LIFECYCLE EVENT
PVC DELETE ≠ SAFE RESET
UNKNOWN RECLAIM BEHAVIOR = NO PVC DELETION
```

No PVC/PV deletion occurs in this lab.

------------------------------------------------------------------------

# 25. Backup / Recovery Status

Record:

``` text
Backup Configured:
Recent Backup Available:
Snapshot Available:
Restore Validated:
```

Use `UNKNOWN` where evidence is unavailable.

Rules:

``` text
PERSISTENCE ≠ BACKUP
SNAPSHOT EXISTS ≠ RECOVERY VALIDATED
BACKUP COMPLETED ≠ RESTORE TESTED
```

Do not create snapshots or perform restores.

------------------------------------------------------------------------

# 26. Expansion Validation

For an expansion already requested outside this lab, compare:

``` text
StorageClass Allows Expansion:
PVC Requested Size:
PV/Backend Size:
Filesystem Size:
Application-Visible Capacity:
```

Classify:

``` text
PVC Request:
Backend Expansion:
Filesystem Expansion:
Application Validation:
```

Rules:

``` text
VOLUME EXPANSION ≠ VOLUME SHRINK
PVC SIZE UPDATED ≠ BACKEND EXPANSION COMPLETE
BACKEND EXPANSION COMPLETE ≠ FILESYSTEM EXPANSION COMPLETE
FILESYSTEM EXPANSION COMPLETE ≠ APPLICATION VALIDATION COMPLETE
```

Do not initiate expansion.

------------------------------------------------------------------------

# 27. Wrong StorageClass

Compare:

``` text
Expected Storage Profile:
Expected StorageClass:
Rendered PVC StorageClass:
PV StorageClass:
Infrastructure Storage Type:
```

Classify:

``` text
MATCH / MISMATCH / UNKNOWN
```

Rule:

``` text
STORAGECLASS CHANGE
=
ARCHITECTURE + DATA + AVAILABILITY + COST EVENT
```

Do not switch StorageClasses during diagnosis.

------------------------------------------------------------------------

# 28. Missing Data Investigation

If expected data appears missing, first prove:

``` text
Environment
Workload
Pod UID
PVC UID
PV
Infrastructure Volume
Mount/Device
StatefulSet Ordinal
Application Path
```

Possible causes:

``` text
Wrong environment
Wrong Pod
Wrong PVC
Recreated PVC
Wrong PV
Wrong mount path
Wrong replica
Application path change
Empty replacement volume
Restore issue
Actual deletion/corruption
```

Rule:

``` text
DATA NOT VISIBLE ≠ DATA LOST
```

Do not inspect business data merely to prove identity.

------------------------------------------------------------------------

# 29. Incident Timeline

Record:

``` text
T1 Last Known Healthy:
T2 Workload/Configuration Change:
T3 Node/Scheduling Change:
T4 Storage Event:
T5 Symptom Began:
T6 Alert:
T7 Investigation Began:
```

Use `UNKNOWN` rather than guessing.

------------------------------------------------------------------------

# 30. Five-State Comparison

Record:

``` text
Desired Storage:
Rendered Storage:
Provisioned Storage:
Attached/Mounted Storage:
Application-Observed Storage:
```

Assign:

``` text
PROVEN / SUPPORTED / UNKNOWN / ASSUMED
```

Rule:

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

------------------------------------------------------------------------

# 31. Desired-State Ownership

Possible authoritative sources:

``` text
TrueFoundry
GitOps
Helm
Kubernetes Manifest
Operator
Terraform / OpenTofu
Application Repository
Platform Automation
Other
UNKNOWN
```

Record:

``` text
Observed PVC:
Authoritative Configuration Source:
Authoritative Configuration Owner:
Confidence:
```

Rules:

``` text
OBSERVED PVC ≠ AUTHORITATIVE CONFIGURATION SOURCE
Observed Drift ≠ Permission to Patch
```

------------------------------------------------------------------------

# 32. Lowest Proven Healthy Layer

Use:

``` text
Desired Configuration
→ Rendered Workload
→ PVC
→ StorageClass / Static PV
→ Provisioner
→ PV
→ Infrastructure Storage
→ [Attachment]
→ Node Presentation
→ Filesystem / Block Device
→ Application Access
→ Capacity / Performance
→ Expected Application Data
```

Record:

``` text
Lowest Proven Healthy Layer:
```

Only evidence-backed layers qualify.

------------------------------------------------------------------------

# 33. First Failed Transition

Examples:

``` text
PVC → Provisioner
Provisioner → Infrastructure Volume
PV → Volume Attachment
Volume Attached → Node Presentation
Node Presentation → Application Access
Application Access → Expected Data
```

Record:

``` text
First Failed Transition:
```

Avoid unsupported labels such as only "storage issue" or "Kubernetes
issue."

------------------------------------------------------------------------

# 34. Minimum Supported Blast Radius

Choose the smallest evidence-supported scope:

``` text
One mount/device
One Pod
One PVC
One PV
One StatefulSet replica
One Node
One Availability Zone
One StorageClass
One CSI component
One shared filesystem
One storage backend
One namespace
One cluster
Multiple clusters
```

Record:

``` text
Minimum Supported Blast Radius:
Evidence:
```

Rule:

``` text
ONE PVC FAILURE ≠ CLUSTER STORAGE FAILURE
```

------------------------------------------------------------------------

# 35. Evidence Confidence

Use:

``` text
PROVEN
SUPPORTED
UNKNOWN
ASSUMED
```

Example:

``` text
PVC Bound                         PROVEN
Expected StorageClass             PROVEN
Pod reports FailedMount           PROVEN
CSI node path implicated          SUPPORTED
Disk corruption                   UNKNOWN
```

Rule:

``` text
ASSUMED ≠ ROOT CAUSE
```

------------------------------------------------------------------------

# 36. Current Actionable Owner

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

Record:

``` text
Authoritative Configuration Owner:
Current Actionable Owner:
Requested Action:
```

Potential owners include application, TrueFoundry/platform,
Kubernetes/SRE, cloud infrastructure, storage, security/IAM, or
CSI/add-on teams.

Do not infer organizational ownership solely from architecture.

------------------------------------------------------------------------

# 37. Evidence Preservation

Before approved remediation, preserve as relevant:

``` text
Original Error
Timestamp
Workload
Pod UID
Node
PVC Name/UID
PV
StorageClass
CSI Driver
Infrastructure Volume Identifier
VolumeAttachment
PVC/Pod Events
Topology
Capacity/Inode Evidence
Performance Evidence
Data Criticality
Backup/Restore Status
Lowest Proven Healthy Layer
First Failed Transition
Minimum Supported Blast Radius
Evidence Confidence
Owner/Handoff
```

Rule:

``` text
RESTART OR RESCHEDULE CAN DESTROY INCIDENT EVIDENCE
```

------------------------------------------------------------------------

# 38. Production Scenario Checks

Validate evidence without causing the scenario.

## Scenario 1 --- Pending with WaitForFirstConsumer

Record binding mode, consumer, scheduling state, topology, and
provisioning errors.

## Scenario 2 --- Provisioning Failure

Record PVC events, StorageClass, provisioner/CSI, cloud authorization,
topology, and capacity evidence.

## Scenario 3 --- Bound PVC + FailedMount

Record PV, node, attachment if applicable, Pod events, and CSI node
evidence.

## Scenario 4 --- Multi-Attach

Record old/new Pod and node, access mode, VolumeAttachment, and
infrastructure attachment. Force detach = NO.

## Scenario 5 --- Wrong StorageClass

Compare desired storage profile, rendered PVC, effective StorageClass,
PV, and backend. Patch = NO.

## Scenario 6 --- Volume Full

Compare PVC/PV/backend/filesystem/application capacity. Resize = NO.

## Scenario 7 --- Inode Exhaustion

Use existing inode metrics. Test file = NO.

## Scenario 8 --- Permission Failure

Compare mount evidence, container identity/security context, expected
path, and application error.

## Scenario 9 --- Reschedule Recovery Failure

Compare old node/attachment, new node/attachment, topology, CSI events,
and infrastructure state.

## Scenario 10 --- Storage Latency

Correlate application latency, storage latency, IOPS, throughput,
queueing, throttling, and timestamps.

## Scenario 11 --- StatefulSet Replica Failure

Map ordinal → PVC → PV → node → storage state. Prove smallest blast
radius.

## Scenario 12 --- PVC Deletion Proposed

Record data criticality, reclaim policy, backend volume, backup, restore
validation, owner. PVC deleted = NO.

## Scenario 13 --- Expansion Incomplete

Compare requested PVC size, backend size, filesystem size, and
application-visible capacity. Expansion initiated by lab = NO.

## Scenario 14 --- Data Appears Missing

Prove environment, workload, Pod UID, PVC UID, PV, backend volume,
mount/device, replica, and application path. Data loss remains UNKNOWN
unless proven.

------------------------------------------------------------------------

# 39. Evidence Handoff Contract

Complete:

``` text
Environment:
TrueFoundry Workspace:

Cluster:
Kubernetes Context:
Namespace:

Workload:
Workload Type:

Pod:
Pod UID:
Node:

Mount Path:
Device Path:
Volume Mode:

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

Node Presentation / Mount Status:

Filesystem Status:

Filesystem Capacity:
Filesystem Free Capacity:
Inode Status:

Storage Performance:

Data Criticality:
Disposable / Reconstructable / Stateful / Unique / UNKNOWN

Backup Available:
YES / NO / UNKNOWN

Snapshot Available:
YES / NO / UNKNOWN

Restore Validated:
YES / NO / UNKNOWN

Original Symptom:
Relevant Events:
Relevant Timestamps:

Desired Storage:
Rendered Storage:
Provisioned Storage:
Attached/Mounted Storage:
Application-Observed Storage:

Lowest Proven Healthy Layer:

First Failed Transition:

Minimum Supported Blast Radius:

Evidence Confidence:
PROVEN / SUPPORTED / UNKNOWN / ASSUMED

Authoritative Configuration Source:
Authoritative Configuration Owner:
Current Actionable Owner:
Requested Action:

PVC Modified: NO
PVC Deleted: NO
PV Modified: NO
StorageClass Modified: NO
Volume Force-Detached: NO
Volume Manually Attached: NO
Filesystem Repaired: NO
Test File Written: NO
Application Data Copied: NO
Secret Decoded: NO
Pod Restarted: NO
CSI Component Restarted: NO
Infrastructure Storage Modified: NO
```

------------------------------------------------------------------------

# 40. Acceptance Checklist

The lab passes when relevant items are proven or explicitly `UNKNOWN` /
`NOT APPLICABLE`.

``` text
[ ] Correct context proven.
[ ] Namespace proven.
[ ] Original symptom captured.
[ ] Workload and Pod identified.
[ ] Pod UID captured.
[ ] Volume reference and mount/device identified.
[ ] Volume mode identified.
[ ] PVC and PVC UID identified.
[ ] PVC phase classified.
[ ] Requested capacity/access mode captured.
[ ] StorageClass/binding mode identified.
[ ] WaitForFirstConsumer evaluated when relevant.
[ ] PV identified.
[ ] PVC/PV identity proven.
[ ] Reclaim policy captured.
[ ] CSI driver identified.
[ ] Infrastructure volume identified when safely possible.
[ ] Attachment requirement classified.
[ ] VolumeAttachment checked when applicable.
[ ] Node/topology evaluated.
[ ] Storage events reviewed.
[ ] Provisioning classified.
[ ] Attachment classified.
[ ] Node presentation/mount classified.
[ ] Application access classified.
[ ] Capacity/inodes classified when relevant.
[ ] Performance classified when relevant.
[ ] StatefulSet identity checked when relevant.
[ ] Shared-storage blast radius considered.
[ ] Data criticality classified.
[ ] Backup and restore status recorded.
[ ] Five storage states compared.
[ ] Authoritative source/owner identified or UNKNOWN.
[ ] Timeline captured.
[ ] Lowest Proven Healthy Layer identified.
[ ] First Failed Transition identified.
[ ] Minimum Supported Blast Radius identified.
[ ] Evidence Confidence assigned.
[ ] Current Actionable Owner identified.
[ ] Requested Action recorded.
[ ] PVC Modified = NO.
[ ] PVC Deleted = NO.
[ ] PV Modified = NO.
[ ] StorageClass Modified = NO.
[ ] Volume Force-Detached = NO.
[ ] Test File Written = NO.
[ ] Application Data Copied = NO.
[ ] Secret Decoded = NO.
[ ] Pod Restarted = NO.
[ ] Infrastructure Storage Modified = NO.
```

------------------------------------------------------------------------

# 41. Investigation Flow

``` text
CAPTURE ORIGINAL SYMPTOM
→ VERIFY CONTEXT / NAMESPACE
→ IDENTIFY WORKLOAD / POD
→ CAPTURE POD UID
→ IDENTIFY VOLUME REFERENCE
→ IDENTIFY PVC / PVC UID
→ CLASSIFY PVC STATE
→ IDENTIFY STORAGECLASS / STATIC PV
→ CHECK BINDING MODE
→ IDENTIFY PV
→ PROVE PVC → PV IDENTITY
→ IDENTIFY CSI DRIVER
→ IDENTIFY INFRASTRUCTURE VOLUME
→ VALIDATE PROVISIONING
→ VALIDATE TOPOLOGY
→ DETERMINE WHETHER ATTACHMENT APPLIES
→ VALIDATE ATTACHMENT WHEN REQUIRED
→ VALIDATE NODE PRESENTATION
→ VALIDATE FILESYSTEM / BLOCK DEVICE
→ VALIDATE APPLICATION ACCESS
→ VALIDATE CAPACITY / INODES
→ VALIDATE PERFORMANCE
→ VALIDATE STORAGE IDENTITY / EXPECTED DATA PATH
→ CLASSIFY DATA CRITICALITY
→ IDENTIFY LOWEST PROVEN HEALTHY LAYER
→ IDENTIFY FIRST FAILED TRANSITION
→ PROVE MINIMUM SUPPORTED BLAST RADIUS
→ IDENTIFY AUTHORITATIVE OWNER
→ IDENTIFY CURRENT ACTIONABLE OWNER
→ HAND OFF APPROVED REMEDIATION
```

------------------------------------------------------------------------

# 42. Final Guardrails

``` text
PVC EXISTS ≠ PVC BOUND
PVC BOUND ≠ VOLUME ATTACHED
VOLUME ATTACHED ≠ VOLUME MOUNTED
VOLUME MOUNTED ≠ APPLICATION STORAGE HEALTHY
MOUNTED ≠ PERFORMANT
PVC PENDING ≠ PVC BROKEN
ReadWriteOnce ≠ Exactly One Pod
ACCESS MODE ≠ COMPLETE STORAGE TOPOLOGY
PERSISTENT VOLUME ≠ ALWAYS A FILESYSTEM
NO VolumeAttachment OBJECT ≠ STORAGE FAILURE
DATA NOT VISIBLE ≠ DATA LOST
RESOURCE NAME ≠ RESOURCE IDENTITY
SAME PVC NAME ≠ SAME STORAGE IDENTITY
PERSISTENCE ≠ BACKUP
SNAPSHOT EXISTS ≠ RECOVERY VALIDATED
BACKUP COMPLETED ≠ RESTORE TESTED
PVC DELETE ≠ SAFE RESET
MULTI-ATTACH ERROR ≠ PERMISSION TO FORCE DETACH
FREE DISK SPACE ≠ FILESYSTEM CAN CREATE FILES
PVC SIZE ≠ FILESYSTEM FREE SPACE
CSI POD RUNNING ≠ CSI CLOUD AUTHORIZATION HEALTHY
UNKNOWN DATA CRITICALITY = NO DESTRUCTIVE STORAGE ACTION
UNKNOWN RECLAIM BEHAVIOR = NO PVC DELETION
OBSERVED PVC ≠ AUTHORITATIVE CONFIGURATION SOURCE
Observed Drift ≠ Permission to Patch
READ-ONLY KUBERNETES COMMAND ≠ PERMISSION TO READ APPLICATION DATA
TEST WRITE = PRODUCTION DATA MUTATION
RESTART OR RESCHEDULE CAN DESTROY INCIDENT EVIDENCE
TIMING CORRELATION ≠ ROOT CAUSE PROOF
ASSUMED ≠ ROOT CAUSE
SYMPTOM ≠ FAILURE DOMAIN ≠ OWNER
```

------------------------------------------------------------------------

# 43. Completion Standard

The engineer must be able to answer:

``` text
What was the original symptom?
What workload and Pod are affected?
What is the Pod UID?
What PVC is referenced?
What is the PVC UID?
What PV and infrastructure volume serve it?
What StorageClass and CSI driver serve it?
What volume/access mode applies?
Is WaitForFirstConsumer relevant?
Did provisioning succeed?
Is topology compatible?
Does attachment apply?
Did attachment succeed?
Did node presentation/mount succeed?
Can the application use the expected path?
Is capacity healthy?
Are inodes healthy?
Is performance healthy?
What is the data criticality?
Is backup available?
Has restore been validated?
Is this the expected storage identity?
What is the Lowest Proven Healthy Layer?
What is the First Failed Transition?
What is the Minimum Supported Blast Radius?
What is the Evidence Confidence?
Who owns authoritative configuration?
Who is the Current Actionable Owner?
What action is requested next?

PVC modified? NO
PVC deleted? NO
PV modified? NO
StorageClass modified? NO
Volume force-detached? NO
Test file written? NO
Application data copied? NO
Secret decoded? NO
Pod restarted? NO
Infrastructure storage modified? NO
```

The lab ends with an evidence-backed handoff, not experimental
remediation.
