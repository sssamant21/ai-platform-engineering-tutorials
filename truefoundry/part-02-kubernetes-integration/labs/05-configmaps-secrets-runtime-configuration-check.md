# Part 2.5 Lab --- ConfigMaps, Secrets & Runtime Configuration Check

> **\[SAFE-READ\] Production Validation Lab**

## Purpose

This lab validates runtime-configuration evidence for a
TrueFoundry-managed Kubernetes workload without changing production
state.

The lab follows the Part 2.5 investigation model:

``` text
DESIRED CONFIGURATION
        ↓
RENDERED CONFIGURATION
        ↓
DELIVERED CONFIGURATION
        ↓
EFFECTIVE CONFIGURATION
```

The goal is to determine:

``` text
WHAT configuration was expected?

WHAT configuration does the workload reference?

WHAT configuration source exists?

HOW is configuration delivered?

WHEN were the source and Pod created or changed?

ARE different replicas potentially using different configuration generations?

WHERE is the First Failed Transition?

WHO owns the authoritative configuration?
```

This lab is evidence collection, not remediation.

------------------------------------------------------------------------

## Safety Contract

This is a read-only production validation lab.

### Allowed

Use safe inspection commands such as:

``` bash
kubectl config current-context
kubectl cluster-info
kubectl get
kubectl describe
kubectl logs
```

Use JSONPath only to retrieve non-secret metadata and configuration
references.

### Not Allowed

Do **not** run:

``` text
kubectl create
kubectl apply
kubectl delete
kubectl edit
kubectl patch
kubectl replace
kubectl scale
kubectl rollout restart
helm install
helm upgrade
helm uninstall
```

Do not:

-   modify ConfigMaps;
-   modify Secrets;
-   modify ExternalSecrets;
-   modify SecretStores or ClusterSecretStores;
-   modify Deployments, StatefulSets, DaemonSets, Jobs, CronJobs, or
    Pods;
-   restart or recreate workloads;
-   create temporary Pods;
-   create or extract ServiceAccount tokens;
-   decode Secret values;
-   print Secret values;
-   copy credentials into tickets or chat;
-   dump a container's complete environment;
-   use unrestricted `printenv`;
-   use `kubectl exec` as a routine validation shortcut;
-   patch a Kubernetes object because drift is observed;
-   modify an object whose authoritative desired-state owner is unknown.

Production rules:

``` text
UNKNOWN TARGET = NO MUTATION

UNKNOWN WORKLOAD = NO CONFIGURATION CHANGE

SECRET EXISTS ≠ SECRET SHOULD BE DECODED

CONFIGURATION VALIDATION ≠ SECRET EXTRACTION

RUNTIME CONFIGURATION VALIDATION ≠ FULL ENVIRONMENT DUMP

Observed Drift ≠ Permission to Patch

RESTART = STATE-CHANGING REMEDIATION
```

------------------------------------------------------------------------

## Lab Variables

Before starting, identify the target values.

``` text
ENVIRONMENT=<environment>
WORKSPACE=<truefoundry-workspace>

CONTEXT=<kubernetes-context>
NAMESPACE=<namespace>

WORKLOAD_TYPE=<deployment|statefulset|daemonset|job|cronjob>
WORKLOAD=<workload-name>

POD=<affected-pod>
CONTAINER=<container-name>
```

Optional objects:

``` text
CONFIGMAP=<configmap-name>
SECRET=<secret-name>
EXTERNALSECRET=<externalsecret-name>
SECRETSTORE=<secretstore-name>
CLUSTERSECRETSTORE=<clustersecretstore-name>
```

Do not continue if the target cluster, context, namespace, or workload
is uncertain.

------------------------------------------------------------------------

# Phase 1 --- Prove the Kubernetes Target

## 1.1 Current context

``` bash
kubectl config current-context
```

Record:

``` text
Expected context:
Observed context:
Match:
YES / NO
```

If the context is unexpected:

``` text
STOP
```

Do not continue against an unknown cluster.

------------------------------------------------------------------------

## 1.2 Cluster connectivity

``` bash
kubectl cluster-info
```

Record:

``` text
Cluster reachable:
YES / NO
```

This proves API connectivity only.

``` text
KUBERNETES API REACHABLE
        ≠
WORKLOAD CONFIGURATION CORRECT
```

------------------------------------------------------------------------

## 1.3 Verify namespace

``` bash
kubectl get namespace <namespace>
```

Record:

``` text
Namespace:
Exists:
YES / NO
```

Do not infer workload health from namespace existence.

------------------------------------------------------------------------

# Phase 2 --- Identify the Workload

## 2.1 Verify controller

For a Deployment:

``` bash
kubectl get deployment <workload> -n <namespace>
```

For a StatefulSet:

``` bash
kubectl get statefulset <workload> -n <namespace>
```

For a DaemonSet:

``` bash
kubectl get daemonset <workload> -n <namespace>
```

For a Job:

``` bash
kubectl get job <workload> -n <namespace>
```

For a CronJob:

``` bash
kubectl get cronjob <workload> -n <namespace>
```

Record:

``` text
Controller:
Controller type:
Namespace:
Observed:
YES / NO
```

------------------------------------------------------------------------

## 2.2 Capture controller metadata

For a Deployment:

``` bash
kubectl get deployment <workload> -n <namespace> -o jsonpath="{.metadata.name}{'\n'}{.metadata.generation}{'\n'}{.status.observedGeneration}{'\n'}{.metadata.creationTimestamp}{'\n'}"
```

Record:

``` text
Controller generation:
Observed generation:
Creation timestamp:
```

Do not interpret `generation` as an application configuration version.

------------------------------------------------------------------------

# Phase 3 --- Identify the Affected Pod

## 3.1 List candidate Pods

Use the workload's known selector when available.

Example:

``` bash
kubectl get pods -n <namespace> -l <label-key>=<label-value> -o wide
```

Do not guess labels.

If the selector is unknown, inspect the controller:

``` bash
kubectl describe deployment <workload> -n <namespace>
```

or the equivalent controller type.

Record:

``` text
Affected Pod:
Healthy comparison Pod:
If available

Node:
Pod status:
```

------------------------------------------------------------------------

## 3.2 Capture Pod age and restart evidence

``` bash
kubectl get pod <pod> -n <namespace> -o jsonpath="{.metadata.name}{'\n'}{.metadata.creationTimestamp}{'\n'}"
```

Container restart counts:

``` bash
kubectl get pod <pod> -n <namespace> -o jsonpath="{range .status.containerStatuses[*]}{.name}{': '}{.restartCount}{'\n'}{end}"
```

Record:

``` text
Pod creation timestamp:

Container:
Restart count:
```

This is important for configuration-generation analysis.

------------------------------------------------------------------------

# Phase 4 --- Capture the Original Symptom

Record the original failure before investigating configuration.

``` text
Original symptom:

Original error:

Incident start timestamp:

Affected transaction or operation:

Affected workload:

Affected Pod if known:

Healthy comparison available:
YES / NO
```

Do not rewrite an application error into a configuration diagnosis.

``` text
SYMPTOM ≠ ROOT CAUSE
```

------------------------------------------------------------------------

# Phase 5 --- Identify Expected Configuration

Record only what is known.

``` text
Expected configuration:

Expected configuration source:

Expected key or reference:

Expected delivery mechanism:
Environment / Volume / subPath / Other / UNKNOWN

Expected application reload model:
Startup-only / Dynamic / UNKNOWN

Evidence source:
```

Evidence may come from approved deployment documentation, GitOps/IaC
configuration, platform configuration, application documentation, or an
authoritative owner.

Do not invent expected values.

------------------------------------------------------------------------

# Phase 6 --- Inspect Workload Configuration References

## 6.1 Describe the controller

For a Deployment:

``` bash
kubectl describe deployment <workload> -n <namespace>
```

Use the corresponding `describe` command for other controller types.

Inspect:

``` text
Environment references
ConfigMap references
Secret references
Volumes
Volume mounts
Command
Arguments
Annotations
Events
```

Do not copy Secret values.

------------------------------------------------------------------------

## 6.2 Inspect rendered Pod-template structure

When needed:

``` bash
kubectl get deployment <workload> -n <namespace> -o yaml
```

This is read-only, but review output carefully before sharing it because
workload manifests can contain sensitive literals.

Look specifically for:

``` text
env
envFrom
configMapKeyRef
configMapRef
secretKeyRef
secretRef
volumes
configMap
secret
volumeMounts
subPath
command
args
```

Record:

``` text
Observed configuration mechanism:

Observed ConfigMap reference:

Observed Secret reference:

Observed volume source:

Observed mount path:

subPath used:
YES / NO / UNKNOWN

Literal configuration present:
YES / NO / UNKNOWN

CLI arguments may override configuration:
YES / NO / UNKNOWN
```

------------------------------------------------------------------------

# Phase 7 --- Validate ConfigMap References

If the workload references a ConfigMap:

``` bash
kubectl get configmap <configmap> -n <namespace>
```

Record:

``` text
ConfigMap:
Namespace:
Exists:
YES / NO
```

Production rule:

``` text
ConfigMap Exists ≠ Workload Uses ConfigMap
```

------------------------------------------------------------------------

## 7.1 Capture ConfigMap metadata

``` bash
kubectl get configmap <configmap> -n <namespace> -o jsonpath="{.metadata.name}{'\n'}{.metadata.resourceVersion}{'\n'}{.metadata.creationTimestamp}{'\n'}{.immutable}{'\n'}"
```

Record:

``` text
ConfigMap resourceVersion:
Creation timestamp:
Immutable:
YES / NO / NOT SET
```

Remember:

``` text
KUBERNETES resourceVersion
        ≠
APPLICATION CONFIGURATION VERSION
```

------------------------------------------------------------------------

## 7.2 Verify required key without broad disclosure

When the expected key is non-sensitive and policy permits:

``` bash
kubectl get configmap <configmap> -n <namespace> -o jsonpath="{.data.<expected-key>}"
```

Prefer proving existence rather than broadly printing the complete
ConfigMap.

If content itself is sensitive operational information, do not print it.

Record:

``` text
Expected key:
Key exists:
YES / NO / UNKNOWN
```

Keep:

``` text
ConfigMap Exists ≠ Required Key Exists
```

------------------------------------------------------------------------

# Phase 8 --- Validate `envFrom`

If the workload uses:

``` yaml
envFrom:
  - configMapRef:
```

record:

``` text
envFrom source:
Expected environment variable:
Source key:
```

Do not assume:

``` text
KEY EXISTS IN CONFIGMAP
        =
KEY BECAME CONTAINER ENVIRONMENT VARIABLE
```

Record:

``` text
Environment-variable delivery proven:
YES / NO / UNKNOWN
```

Do not use unrestricted `printenv` to prove it.

------------------------------------------------------------------------

# Phase 9 --- Validate Secret References Safely

If a Secret is referenced:

``` bash
kubectl get secret <secret> -n <namespace>
```

Record:

``` text
Secret:
Namespace:
Exists:
YES / NO
Type:
```

Do not use:

``` text
-o jsonpath={.data...}
base64 --decode
```

for credential inspection.

Do not print `.data`.

------------------------------------------------------------------------

## 9.1 Secret metadata

Safe metadata inspection:

``` bash
kubectl get secret <secret> -n <namespace> -o jsonpath="{.metadata.name}{'\n'}{.metadata.resourceVersion}{'\n'}{.metadata.creationTimestamp}{'\n'}{.type}{'\n'}{.immutable}{'\n'}"
```

Record:

``` text
Secret resourceVersion:
Creation timestamp:
Type:
Immutable:
YES / NO / NOT SET

Secret value exposed:
NO
```

Rules:

``` text
SECRET EXISTS ≠ SECRET CONTENT CORRECT

SECRET EXISTS ≠ WORKLOAD USES SECRET

Correct Secret Reference ≠ Valid Credential
```

------------------------------------------------------------------------

# Phase 10 --- Verify Secret-Key Reference Without Extracting Value

Inspect the workload definition to identify:

``` text
secretKeyRef.name
secretKeyRef.key
```

Record:

``` text
Referenced Secret:
Referenced key name:
Reference optional:
YES / NO / UNKNOWN
```

Do not retrieve the Secret value.

If policy permits verification of key-name metadata, use an approved
metadata-only method for the environment. If not, record:

``` text
Required key existence:
UNKNOWN
```

Unknown evidence is preferable to credential exposure.

``` text
UNKNOWN ≠ FAILED
```

------------------------------------------------------------------------

# Phase 11 --- Check Optional vs Required Semantics

For ConfigMap or Secret references, inspect whether the reference is
optional.

Record:

``` text
Reference:
Optional:
YES / NO / UNKNOWN
```

Then:

``` text
Application fallback behavior:
Known / Unknown

Fatal when missing:
PROVEN / SUPPORTED / UNKNOWN
```

Production rule:

``` text
MISSING CONFIGURATION ≠ AUTOMATIC OUTAGE
```

------------------------------------------------------------------------

# Phase 12 --- Check Immutability

For relevant ConfigMaps and Secrets record:

``` text
immutable:
true / false / not set
```

Keep:

``` text
CONFIGURATION OBJECT EXISTS
        ≠
CONFIGURATION OBJECT IS MUTABLE

FAILED CONFIG UPDATE
        ≠
AUTOMATICALLY RBAC FAILURE
```

This lab does not attempt any update.

------------------------------------------------------------------------

# Phase 13 --- Check Environment-Variable Delivery

If configuration is delivered through:

``` text
env
envFrom
configMapKeyRef
secretKeyRef
```

record:

``` text
Source modification/change evidence:
Pod creation timestamp:
```

Ask:

``` text
Was the Pod created before or after the relevant configuration change?
```

Classify:

``` text
Pod definitely predates source change:
YES / NO / UNKNOWN
```

Keep:

``` text
CONFIGMAP UPDATED
        ≠
EXISTING CONTAINER ENVIRONMENT UPDATED

SECRET UPDATED
        ≠
EXISTING CONTAINER ENVIRONMENT UPDATED
```

Do not restart the Pod to test the theory.

------------------------------------------------------------------------

# Phase 14 --- Check Mounted Configuration

Inspect:

``` text
volumes
volumeMounts
mountPath
subPath
```

Record:

``` text
Volume source:
Mount path:
subPath:
YES / NO / UNKNOWN
```

Keep:

``` text
CONFIGMAP UPDATED
        ≠
MOUNT UPDATED IMMEDIATELY

MOUNT CURRENT
        ≠
APPLICATION CONFIGURATION CURRENT

MOUNT UPDATED
        ≠
APPLICATION RELOADED
```

------------------------------------------------------------------------

# Phase 15 --- Mandatory `subPath` Check

If `subPath` is present:

``` text
subPath:
PROVEN
```

Record it prominently in the evidence handoff.

Do not assume normal projected-volume update behavior.

``` text
Normal Volume Update Behavior
        ≠
subPath Update Behavior
```

Possible evidence statement:

``` text
The running Pod uses subPath for the affected configuration.
Normal ConfigMap/Secret projected-volume update assumptions
therefore do not apply to this mount.
```

Do not restart the Pod during this lab.

------------------------------------------------------------------------

# Phase 16 --- Compare Configuration Generations Across Replicas

List workload Pods:

``` bash
kubectl get pods -n <namespace> -l <label-key>=<label-value> -o wide
```

Capture creation timestamps:

``` bash
kubectl get pods -n <namespace> -l <label-key>=<label-value> -o custom-columns="NAME:.metadata.name,CREATED:.metadata.creationTimestamp,NODE:.spec.nodeName,PHASE:.status.phase"
```

Record:

``` text
Pod A:
Created:
Observed behavior:

Pod B:
Created:
Observed behavior:

Pod C:
Created:
Observed behavior:
```

Ask:

``` text
Are failing Pods older than healthy Pods?

Did a configuration change occur between their creation times?

Are multiple ReplicaSet/controller revisions present?
```

Keep:

``` text
SAME DEPLOYMENT
        ≠
SAME EFFECTIVE CONFIGURATION
```

------------------------------------------------------------------------

# Phase 17 --- Inspect ReplicaSet / Revision Evidence

For a Deployment:

``` bash
kubectl get replicasets -n <namespace>
```

When labels are known, narrow the query appropriately.

Inspect a relevant ReplicaSet:

``` bash
kubectl describe replicaset <replicaset> -n <namespace>
```

Record:

``` text
Affected Pod controller revision:
Healthy Pod controller revision:
Same:
YES / NO / UNKNOWN
```

Do not infer configuration differences from revision alone.

``` text
DIFFERENT REVISION
        ≠
CONFIGURATION DIFFERENCE PROVEN
```

------------------------------------------------------------------------

# Phase 18 --- Application Reload Model

Determine from approved application documentation or owner evidence:

``` text
Application reload model:
Startup-only / Dynamic / UNKNOWN
```

If dynamic:

``` text
Reload mechanism:
If known
```

If unknown, leave it unknown.

Do not trigger a reload during this lab.

Keep:

``` text
SOURCE UPDATED ≠ APPLICATION UPDATED
```

------------------------------------------------------------------------

# Phase 19 --- Configuration Precedence

Inspect the workload specification for:

``` text
command
args
env
envFrom
mounted configuration
```

Record:

``` text
Known configuration sources:

Default:
File:
Environment:
CLI:
Remote configuration:

Known precedence:
If documented

Effective source:
PROVEN / SUPPORTED / UNKNOWN
```

Keep:

``` text
CONFIGMAP VALUE
        ≠
EFFECTIVE APPLICATION VALUE

EXPECTED SOURCE
        ≠
EFFECTIVE SOURCE
```

------------------------------------------------------------------------

# Phase 20 --- ExternalSecret Discovery

Only perform this phase if an external-secret mechanism is actually
deployed.

Do not assume its presence.

Discovery examples may include:

``` bash
kubectl get externalsecrets -n <namespace>
```

If the resource type is not installed, do not treat that as an
application failure.

Record:

``` text
External secret mechanism deployed:
YES / NO / UNKNOWN
```

Keep:

``` text
DISCOVER ACTUAL SECRET DELIVERY MECHANISM
```

------------------------------------------------------------------------

# Phase 21 --- Inspect ExternalSecret Status

If an ExternalSecret is used:

``` bash
kubectl get externalsecret <externalsecret> -n <namespace>
```

``` bash
kubectl describe externalsecret <externalsecret> -n <namespace>
```

Inspect:

``` text
Status
Conditions
Events
Referenced SecretStore / ClusterSecretStore
Target Secret name
Refresh/reconciliation evidence
```

Record:

``` text
ExternalSecret:
Exists:
YES / NO

Ready:
YES / NO / UNKNOWN

Target Secret:

Relevant condition:

Relevant event:
```

Keep:

``` text
ExternalSecret Exists ≠ ExternalSecret Ready
```

------------------------------------------------------------------------

# Phase 22 --- Inspect SecretStore Reference

If a namespaced SecretStore is referenced:

``` bash
kubectl get secretstore <secretstore> -n <namespace>
kubectl describe secretstore <secretstore> -n <namespace>
```

If a ClusterSecretStore is referenced:

``` bash
kubectl get clustersecretstore <clustersecretstore>
kubectl describe clustersecretstore <clustersecretstore>
```

Do not retrieve provider credentials.

Record:

``` text
Store type:
SecretStore / ClusterSecretStore

Store:
Exists:
YES / NO

Ready:
YES / NO / UNKNOWN

Provider:
If safely identifiable
```

------------------------------------------------------------------------

# Phase 23 --- External Secret Failure Boundary

Use observed conditions/events to classify the first supported boundary:

``` text
Provider authentication
Provider authorization
Remote object lookup
Remote property lookup
Provider connectivity
SecretStore configuration
Secret reconciliation
Kubernetes Secret materialization
Workload reference
Application authentication
UNKNOWN
```

Record:

``` text
Observed external-secret failure boundary:

Evidence confidence:
PROVEN / SUPPORTED / UNKNOWN / ASSUMED
```

Do not retrieve the remote secret to "verify" it.

------------------------------------------------------------------------

# Phase 24 --- Controller Health vs Reconciliation

If the secret controller itself is running:

``` text
SECRET CONTROLLER RUNNING
        ≠
SECRET RECONCILIATION SUCCEEDED
```

Do not stop at controller Pod health.

Conversely, one failed ExternalSecret does not prove the entire
controller is unhealthy.

Record the smallest supported scope.

------------------------------------------------------------------------

# Phase 25 --- Application Authentication Failure

If:

``` text
Kubernetes Secret exists
Workload reference exists
Pod is running
External authentication fails
```

do not conclude:

``` text
Secret is wrong
```

Remaining possibilities include:

``` text
Expired credential
Wrong credential version
Application did not reload
Stale Pod
Wrong endpoint
Application precedence
External authorization
External service state
```

Record:

``` text
Kubernetes delivery:
PROVEN / SUPPORTED / UNKNOWN

External authentication:
ALLOWED / DENIED / UNKNOWN

Credential correctness:
PROVEN / SUPPORTED / UNKNOWN
```

Do not decode the Secret.

------------------------------------------------------------------------

# Phase 26 --- Shared ConfigMap Consumers

Before recommending any future change, identify known consumers.

Inspect workload references using approved read-only repository/platform
evidence or Kubernetes specifications.

Record:

``` text
ConfigMap:

Known consumers:

Consumer 1:
Namespace:
Workload:

Consumer 2:
Namespace:
Workload:

Consumer scope complete:
YES / NO / UNKNOWN
```

Keep:

``` text
CONFIGMAP CHANGE ≠ SINGLE-WORKLOAD CHANGE
```

If consumer scope is unknown, blast radius is not proven.

------------------------------------------------------------------------

# Phase 27 --- Shared Secret Consumers

Record known Secret consumers without exposing Secret values.

``` text
Secret:

Known consumers:

Consumer 1:
Namespace:
Workload:

Consumer 2:
Namespace:
Workload:

Consumer scope complete:
YES / NO / UNKNOWN
```

Keep:

``` text
SECRET CHANGE ≠ SINGLE-WORKLOAD CHANGE
```

and:

``` text
SECRET ROTATION
        =
CONFIGURATION + SECURITY + AVAILABILITY EVENT
```

------------------------------------------------------------------------

# Phase 28 --- Determine Desired-State Ownership

Identify the authoritative owner.

Possible sources:

``` text
TrueFoundry
GitOps
Helm
Terraform / OpenTofu
External Secrets
Vault
Cloud secret manager
Application repository
Kubernetes Operator
Platform automation
Manual Kubernetes object
UNKNOWN
```

Record:

``` text
Observed Kubernetes object:

Authoritative configuration source:

Authoritative Configuration Owner:

Ownership confidence:
PROVEN / SUPPORTED / UNKNOWN / ASSUMED
```

Keep:

``` text
KUBERNETES OBJECT
        ≠
AUTHORITATIVE CONFIGURATION SOURCE
```

If ownership is unknown:

``` text
NO DIRECT CONFIGURATION CHANGE
```

------------------------------------------------------------------------

# Phase 29 --- Check for Reconciliation Ownership

Look for read-only evidence such as:

``` text
GitOps annotations
Helm metadata
Operator ownership
TrueFoundry/platform ownership
External-secret ownership
IaC/deployment documentation
```

Record:

``` text
Reconciliation system:

Desired state external to Kubernetes:
YES / NO / UNKNOWN
```

Keep:

``` text
DO NOT PATCH THE SYMPTOM
WHEN ANOTHER SYSTEM OWNS DESIRED STATE
```

------------------------------------------------------------------------

# Phase 30 --- Build the Incident Timeline

Record:

``` text
T1 Configuration changed:

T2 Controller reconciled:

T3 Pod created:

T4 Application symptom began:

T5 Alert fired:

T6 Investigation began:
```

Use `UNKNOWN` when evidence is unavailable.

Then ask:

``` text
Which transition correlates with the symptom?
```

Keep:

``` text
TIMING CORRELATION ≠ ROOT CAUSE PROOF
```

------------------------------------------------------------------------

# Phase 31 --- Determine Lowest Proven Healthy Layer

Use the chain:

``` text
Authoritative desired state
        ↓
Rendered Kubernetes object
        ↓
Workload reference
        ↓
Pod configuration
        ↓
Container delivery
        ↓
Application configuration loader
        ↓
Effective runtime configuration
        ↓
External dependency
```

Record:

``` text
Lowest Proven Healthy Layer:
```

Examples:

``` text
Rendered ConfigMap
Workload configuration reference
Pod configuration delivery
Application configuration loader
External-secret reconciliation request
```

Do not claim a layer healthy without evidence.

------------------------------------------------------------------------

# Phase 32 --- Determine First Failed Transition

Record the first transition where evidence shows failure.

Examples:

``` text
Desired configuration
→
Rendered workload reference
```

``` text
ConfigMap reference
→
ConfigMap lookup
```

``` text
ConfigMap
→
Required key
```

``` text
Updated ConfigMap
→
Running subPath-mounted content
```

``` text
Secret controller
→
External provider authorization
```

``` text
Runtime configuration
→
Application parser
```

``` text
Application credential
→
External authentication
```

Record:

``` text
First Failed Transition:
```

Do not use a generic label such as:

``` text
Config problem
Secret issue
Kubernetes issue
TrueFoundry issue
```

unless evidence supports that exact boundary.

------------------------------------------------------------------------

# Phase 33 --- Determine Minimum Supported Blast Radius

Use the smallest evidence-supported scope:

``` text
One configuration key
One container
One Pod
One ReplicaSet / revision
One workload
Shared ConfigMap consumers
Shared Secret consumers
Namespace
Environment
Multiple environments
```

Record:

``` text
Minimum Supported Blast Radius:

Evidence:
```

Keep:

``` text
ONE AFFECTED POD
        ≠
ENTIRE WORKLOAD AFFECTED
```

and:

``` text
ONE WORKLOAD AFFECTED
        ≠
ENTIRE NAMESPACE AFFECTED
```

------------------------------------------------------------------------

# Phase 34 --- Evidence Confidence

Classify important conclusions:

``` text
PROVEN
SUPPORTED
UNKNOWN
ASSUMED
```

Example:

``` text
ConfigMap changed at 14:02               PROVEN
Affected Pod created at 13:41            PROVEN
Configuration delivered via env          PROVEN
Pod contains old runtime value            UNKNOWN
Stale configuration caused outage         ASSUMED
```

Keep:

``` text
ASSUMED ≠ ROOT CAUSE
```

------------------------------------------------------------------------

# Phase 35 --- Determine Current Actionable Owner

Use:

``` text
First Failed Transition
+
Authoritative Configuration Owner
```

Record:

``` text
Current Actionable Owner:

Requested Action:
```

Examples of requested actions:

``` text
Validate expected desired-state value.

Validate approved credential version.

Validate external-provider authorization.

Validate application configuration precedence.

Validate expected reload behavior.

Validate approved rollout/rotation mechanism.
```

Do not request:

``` text
Patch it and see.
Restart it and see.
Decode the Secret.
Grant broader permissions.
```

Keep:

``` text
SYMPTOM ≠ FAILURE DOMAIN ≠ OWNER
```

------------------------------------------------------------------------

# Phase 36 --- Healthy Replica Comparison

If a healthy replica exists, compare read-only evidence:

``` text
Affected Pod:
Healthy Pod:

Creation timestamps differ:
YES / NO

Controller revisions differ:
YES / NO

Configuration references differ:
YES / NO / UNKNOWN

Nodes differ:
YES / NO

Restart counts differ:
YES / NO

Observed application behavior differs:
YES / NO
```

Do not conclude causality solely from a difference.

Record:

``` text
Replica comparison conclusion:

Evidence confidence:
PROVEN / SUPPORTED / UNKNOWN / ASSUMED
```

------------------------------------------------------------------------

# Phase 37 --- Preserve Pre-Remediation Evidence

Before any future approved remediation, ensure the incident record
contains:

``` text
Original error
Controller metadata
Pod metadata
Pod creation timestamp
Controller revision
Configuration references
ConfigMap/Secret metadata
ExternalSecret status when applicable
Relevant events
Relevant safe logs
Incident timeline
Lowest Proven Healthy Layer
First Failed Transition
Minimum Supported Blast Radius
Authoritative Configuration Owner
```

This lab itself performs no remediation.

------------------------------------------------------------------------

# Phase 38 --- Production Configuration Evidence Handoff

Complete:

``` text
Environment:
TrueFoundry Workspace:

Cluster:
Kubernetes context:
Namespace:

Controller:
Controller type:
Controller generation:
Observed generation:
Revision:

Pod:
Pod creation timestamp:
Container:
Container restart count:

Original symptom:
Incident start timestamp:
Observation timestamp:

Expected configuration:

Expected configuration source:

Observed configuration reference:

Configuration mechanism:
Literal / ConfigMap / Secret /
External secret / Environment /
Mounted file / Other

Delivery type:
Environment / Volume / subPath / Other

ConfigMap:
If applicable

ConfigMap resourceVersion:
If applicable

Secret:
If applicable

Secret resourceVersion:
If applicable

Secret value exposed:
NO

ExternalSecret:
If applicable

SecretStore / ClusterSecretStore:
If applicable

External provider:
If applicable

Required reference optional:
YES / NO / UNKNOWN

Configuration object immutable:
YES / NO / UNKNOWN

Application reload model:
Startup-only / Dynamic / Unknown

Application configuration precedence:
If known

Healthy replica comparison:
If applicable

Affected configuration generation:
If known

Shared configuration consumers:
If applicable

Lowest Proven Healthy Layer:

First Failed Transition:

Minimum Supported Blast Radius:

Evidence Confidence:

Authoritative Configuration Owner:

Current Actionable Owner:

Requested Action:

Full environment dumped:
NO

Secret decoded:
NO

Secret value copied:
NO

Runtime configuration mutated:
NO

Workload restarted:
NO
```

------------------------------------------------------------------------

# Phase 39 --- Acceptance Checklist

The lab passes only when the following are answered with evidence or
explicitly recorded as `UNKNOWN`.

``` text
[ ] Correct Kubernetes context proven.

[ ] Namespace proven.

[ ] Workload proven.

[ ] Affected Pod identified when applicable.

[ ] Original symptom captured.

[ ] Expected configuration identified or marked UNKNOWN.

[ ] Authoritative configuration source identified or marked UNKNOWN.

[ ] Configuration delivery mechanism identified.

[ ] ConfigMap references validated when applicable.

[ ] Secret references validated without exposing values.

[ ] Optional/required semantics checked.

[ ] Immutability checked when relevant.

[ ] Environment-variable delivery semantics considered.

[ ] Mounted-volume behavior considered.

[ ] subPath explicitly checked.

[ ] Pod creation time captured.

[ ] Controller generation/revision captured when relevant.

[ ] Mixed configuration generations considered.

[ ] Healthy replica compared when available.

[ ] Application reload model identified or marked UNKNOWN.

[ ] Configuration precedence identified or marked UNKNOWN.

[ ] External-secret mechanism discovered rather than assumed.

[ ] ExternalSecret status checked when applicable.

[ ] SecretStore/ClusterSecretStore status checked when applicable.

[ ] Shared ConfigMap consumers considered.

[ ] Shared Secret consumers considered.

[ ] Desired-state/reconciliation owner identified or marked UNKNOWN.

[ ] Incident timeline recorded.

[ ] Lowest Proven Healthy Layer identified.

[ ] First Failed Transition identified.

[ ] Minimum Supported Blast Radius identified.

[ ] Evidence Confidence assigned.

[ ] Current Actionable Owner identified.

[ ] Requested Action recorded.

[ ] Secret values exposed = NO.

[ ] Secret decoded = NO.

[ ] Full environment dumped = NO.

[ ] Runtime configuration mutated = NO.

[ ] Workload restarted = NO.
```

------------------------------------------------------------------------

# Production Investigation Flow

``` text
CAPTURE ORIGINAL SYMPTOM
        ↓
VERIFY CLUSTER / CONTEXT / NAMESPACE
        ↓
IDENTIFY AFFECTED WORKLOAD / POD / CONTAINER
        ↓
COMPARE HEALTHY REPLICA WHEN AVAILABLE
        ↓
IDENTIFY EXPECTED CONFIGURATION
        ↓
IDENTIFY AUTHORITATIVE CONFIGURATION SOURCE
        ↓
IDENTIFY CONFIGURATION DELIVERY MECHANISM
        ↓
VERIFY OBSERVED CONFIGURATION REFERENCE
        ↓
VERIFY CONFIGMAP / SECRET / EXTERNAL SOURCE
        ↓
VERIFY REQUIRED KEY / REFERENCE
WITHOUT EXPOSING SECRET VALUES
        ↓
CHECK OPTIONAL / REQUIRED SEMANTICS
        ↓
CHECK IMMUTABILITY
        ↓
CHECK ENVIRONMENT / VOLUME / subPath DELIVERY
        ↓
CHECK CONTROLLER GENERATION
        ↓
CHECK POD CREATION TIME / REVISION
        ↓
CHECK FOR MIXED CONFIGURATION GENERATIONS
        ↓
CHECK APPLICATION RELOAD MODEL
        ↓
CHECK APPLICATION CONFIGURATION PRECEDENCE
        ↓
CHECK NEXT EXTERNAL DEPENDENCY
        ↓
IDENTIFY LOWEST PROVEN HEALTHY LAYER
        ↓
IDENTIFY FIRST FAILED TRANSITION
        ↓
PROVE MINIMUM SUPPORTED BLAST RADIUS
        ↓
IDENTIFY SHARED CONFIGURATION CONSUMERS
        ↓
IDENTIFY AUTHORITATIVE CONFIGURATION OWNER
        ↓
IDENTIFY CURRENT ACTIONABLE OWNER
        ↓
HAND OFF APPROVED REMEDIATION
        ↓
REGRESSION VALIDATION
```

------------------------------------------------------------------------

# Lab Guardrails

The following rules must remain true throughout this lab:

``` text
UNKNOWN TARGET = NO MUTATION

UNKNOWN WORKLOAD = NO CONFIGURATION CHANGE

Pod Running ≠ Runtime Configuration Correct

CONFIGURATION EXISTS
≠
WORKLOAD CONSUMED CONFIGURATION

DESIRED CONFIGURATION
≠
RENDERED CONFIGURATION
≠
DELIVERED CONFIGURATION
≠
EFFECTIVE CONFIGURATION

ConfigMap Exists
≠
Workload Uses ConfigMap

ConfigMap Exists
≠
Required Key Exists

CONFIGMAP CORRECT
≠
AFFECTED POD CONFIGURATION CORRECT

SAME DEPLOYMENT
≠
SAME EFFECTIVE CONFIGURATION

CONFIGMAP UPDATED
≠
EXISTING CONTAINER ENVIRONMENT UPDATED

SECRET UPDATED
≠
EXISTING CONTAINER ENVIRONMENT UPDATED

MOUNT UPDATED
≠
APPLICATION RELOADED

Normal Volume Update Behavior
≠
subPath Update Behavior

SECRET EXISTS
≠
SECRET SHOULD BE DECODED

CONFIGURATION VALIDATION
≠
SECRET EXTRACTION

BASE64 ENCODING
≠
ENCRYPTION

RUNTIME CONFIGURATION VALIDATION
≠
FULL ENVIRONMENT DUMP

NON-SECRET
≠
UNRESTRICTED DISCLOSURE

ExternalSecret Exists
≠
ExternalSecret Ready

ExternalSecret Ready
≠
Application Authentication Works

MISSING CONFIGURATION
≠
AUTOMATIC OUTAGE

FAILED CONFIG UPDATE
≠
AUTOMATICALLY RBAC FAILURE

CURRENT DEPLOYMENT CONFIGURATION
≠
PROOF OF RUNNING POD CONFIGURATION

KUBERNETES OBJECT
≠
AUTHORITATIVE CONFIGURATION SOURCE

Observed Drift
≠
Permission to Patch

DO NOT PATCH THE SYMPTOM
WHEN ANOTHER SYSTEM OWNS DESIRED STATE

CONFIGURATION CHANGE
≠
SINGLE-WORKLOAD CHANGE

RESTART
=
STATE-CHANGING REMEDIATION

RESTART RESTORED SERVICE
≠
ROOT CAUSE IDENTIFIED

SERVICE RESTORED
≠
ROOT CAUSE REMEDIATED

TIMING CORRELATION
≠
ROOT CAUSE PROOF

ASSUMED ≠ ROOT CAUSE

SYMPTOM ≠ FAILURE DOMAIN ≠ OWNER
```

------------------------------------------------------------------------

## Completion Standard

This lab is complete when the engineer can answer:

``` text
WHAT workload is affected?

WHAT configuration was expected?

WHAT configuration is rendered?

HOW is it delivered?

WHAT configuration generation can be supported by evidence?

IS subPath involved?

ARE different replicas potentially using different configuration generations?

IS an external-secret mechanism involved?

WHERE is the Lowest Proven Healthy Layer?

WHERE is the First Failed Transition?

WHAT is the Minimum Supported Blast Radius?

WHO owns authoritative configuration?

WHO is the Current Actionable Owner?

WHAT action is requested next?

WERE Secret values exposed?
NO

WAS a Secret decoded?
NO

WAS the complete runtime environment dumped?
NO

WAS production configuration changed?
NO

WAS a workload restarted?
NO
```

The purpose of the lab is not to make production healthy by changing
state. It is to produce a safe, evidence-backed handoff that identifies
the first proven failure boundary and the owner of the next approved
action.
