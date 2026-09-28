# Part 2.5 --- ConfigMaps, Secrets & Runtime Configuration

## Purpose

Part 2.5 teaches production-safe investigation of runtime configuration
for TrueFoundry-managed Kubernetes workloads.

The central incident question is:

> **WHAT configuration was expected, WHAT configuration did the affected
> workload actually receive, WHEN did it receive it, WHERE is the first
> proven mismatch, and WHO owns the authoritative configuration?**

The central rule is:

``` text
CONFIGURATION EXISTS ≠ WORKLOAD CONSUMED CONFIGURATION
```

------------------------------------------------------------------------

## Learning Objectives

By the end of this tutorial, an SRE or platform engineer should be able
to:

-   Identify how a workload receives runtime configuration.
-   Differentiate ConfigMaps, Secrets, environment variables, mounted
    configuration, and external secret integrations.
-   Trace configuration from authoritative desired state to the
    application.
-   Distinguish desired, rendered, delivered, and effective
    configuration.
-   Determine whether an affected Pod actually received expected
    configuration.
-   Identify mixed configuration generations across replicas.
-   Understand environment-variable, volume, and `subPath` update
    behavior.
-   Investigate external-secret reconciliation without exposing
    credentials.
-   Identify configuration ownership and reconciliation boundaries.
-   Determine the Lowest Proven Healthy Layer and First Failed
    Transition.
-   Establish the Minimum Supported Blast Radius.
-   Investigate production safely without mutating configuration,
    restarting workloads, dumping environments, or decoding Secrets.

------------------------------------------------------------------------

## 1. Production Configuration Chain

Use the following model:

``` text
Authoritative Desired State
        ↓
Rendered Kubernetes Configuration
        ↓
Workload Controller / Pod Template
        ↓
Pod
        ↓
Container Configuration
        ↓
Application Configuration Loader
        ↓
Effective Runtime Configuration
        ↓
External Dependency
```

The authoritative source may be TrueFoundry configuration, GitOps, Helm,
Terraform/OpenTofu, an application repository, an external secret
provider, a cloud secret manager, Vault, platform automation, or another
deployment system.

Never assume ownership from the Kubernetes object alone.

``` text
DISCOVER ACTUAL AUTHORITATIVE CONFIGURATION PATH
```

------------------------------------------------------------------------

## 2. Four Configuration States

Production troubleshooting must distinguish four states:

``` text
DESIRED
   ↓
RENDERED
   ↓
DELIVERED
   ↓
EFFECTIVE
```

### Desired

What the authoritative configuration source intends.

### Rendered

What exists in Kubernetes, such as a Deployment, StatefulSet, Job,
CronJob, ConfigMap, Secret, ExternalSecret, or Pod template.

### Delivered

What reaches a particular Pod/container through environment variables,
mounted files, projected volumes, command-line arguments, or another
runtime mechanism.

### Effective

What the application actually uses after defaults, precedence, parsing,
caching, command-line overrides, reload behavior, or remote
configuration are considered.

``` text
DESIRED CONFIGURATION
        ≠
RENDERED CONFIGURATION
        ≠
DELIVERED CONFIGURATION
        ≠
EFFECTIVE CONFIGURATION
```

------------------------------------------------------------------------

## 3. Establish Configuration Identity First

Record the exact target before troubleshooting:

``` text
Environment:
TrueFoundry Workspace:

Cluster:
Kubernetes context:
Namespace:

Controller:
Controller type:

Pod:
Container:

Observation timestamp:
```

Production rules:

``` text
UNKNOWN TARGET = NO MUTATION
UNKNOWN WORKLOAD = NO CONFIGURATION CHANGE
```

A configuration name without cluster and namespace context is
incomplete.

------------------------------------------------------------------------

## 4. ConfigMaps

ConfigMaps store non-confidential configuration such as application
modes, feature flags, service endpoints, logging configuration, model
paths, runtime identifiers, and non-sensitive integration settings.

Example:

``` yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: model-runtime-config
data:
  MODEL_PATH: /models/llama
  LOG_LEVEL: INFO
```

Important:

``` text
CONFIGMAP ≠ SECRET
CONFIDENTIAL DATA → DO NOT STORE IN CONFIGMAP
```

The operational identity of a ConfigMap is:

``` text
Cluster + Namespace + ConfigMap
```

Therefore:

``` text
CONFIGMAP NAME ≠ COMPLETE CONFIGURATION IDENTITY
```

### Safe inspection

``` bash
kubectl get configmap <configmap> -n <namespace>
kubectl describe configmap <configmap> -n <namespace>
```

When policy permits inspection of non-sensitive configuration:

``` bash
kubectl get configmap <configmap> -n <namespace> -o yaml
```

Existence alone proves very little:

``` text
ConfigMap Exists ≠ Workload Uses ConfigMap
ConfigMap Exists ≠ Required Key Exists
```

------------------------------------------------------------------------

## 5. ConfigMap References

An individual key may be injected as an environment variable:

``` yaml
env:
  - name: MODEL_PATH
    valueFrom:
      configMapKeyRef:
        name: model-runtime-config
        key: MODEL_PATH
```

Trace every transition:

``` text
Container variable
      ↓
configMapKeyRef
      ↓
ConfigMap
      ↓
Key
```

A workload can also use `envFrom`:

``` yaml
envFrom:
  - configMapRef:
      name: model-runtime-config
```

Do not assume every ConfigMap key becomes a valid environment variable.

``` text
KEY EXISTS IN CONFIGMAP
        ≠
KEY BECAME CONTAINER ENVIRONMENT VARIABLE
```

------------------------------------------------------------------------

## 6. Environment-Variable Update Semantics

Environment variables are established for the container process.
Updating a source ConfigMap does not rewrite the environment of an
already-running container.

``` text
CONFIGMAP UPDATED
        ≠
EXISTING CONTAINER ENVIRONMENT UPDATED
```

The same principle applies to Secret-derived environment variables:

``` text
SECRET UPDATED
        ≠
EXISTING CONTAINER ENVIRONMENT UPDATED
```

Pod creation time therefore becomes important incident evidence.

------------------------------------------------------------------------

## 7. Mixed Configuration Generations

A single workload can temporarily contain Pods with different effective
configuration.

Example:

``` text
ConfigMap changed at 14:00

Pod A created 13:30 → older configuration
Pod B created 13:31 → older configuration
Pod C created 14:05 → newer configuration
```

Therefore:

``` text
SAME DEPLOYMENT ≠ ALL RUNNING PODS HAVE SAME RUNTIME CONFIGURATION
SAME DEPLOYMENT ≠ SAME EFFECTIVE CONFIGURATION
```

When failures are intermittent, compare healthy and failing replicas.

Capture:

``` text
Pod:
Creation timestamp:
Controller revision:
Configuration references:
Container restart count:
Node:
Observed behavior:
```

Ask:

``` text
WHAT differs between healthy and failing replicas?
```

------------------------------------------------------------------------

## 8. Mounted ConfigMaps

Mounted configuration follows a different path:

``` text
ConfigMap
   ↓
Volume
   ↓
volumeMount
   ↓
Container filesystem
   ↓
Application
```

Keep each transition separate:

``` text
ConfigMap Exists ≠ Volume Correct
Volume Correct ≠ Mount Current
Mount Current ≠ Application Configuration Current
```

Mounted ConfigMap content is not an instantaneous application update.

``` text
CONFIGMAP UPDATED ≠ MOUNT UPDATED IMMEDIATELY
MOUNT UPDATED ≠ APPLICATION RELOADED
```

The application may watch the file, poll it, reload on signal, read it
only during startup, or cache it.

Always establish the application reload model.

------------------------------------------------------------------------

## 9. `subPath`

`subPath` is a critical exception.

Example:

``` yaml
volumeMounts:
  - name: config
    mountPath: /app/config.yaml
    subPath: config.yaml
```

A ConfigMap or Secret mounted through `subPath` does not receive normal
projected-volume source updates in the running container.

``` text
Normal Volume Update Behavior ≠ subPath Update Behavior
```

For stale mounted configuration:

``` text
CHECK subPath
```

Record:

``` text
Volume source:
Mount path:
subPath used:
YES / NO
```

------------------------------------------------------------------------

## 10. Kubernetes Secrets

Secrets are intended for sensitive information such as passwords, API
tokens, credentials, certificates, and private keys.

A workload may consume Secrets through `secretKeyRef`, `secretRef`,
Secret-backed volumes, external secret integrations, or other supported
mechanisms.

Safe initial existence check:

``` bash
kubectl get secret <secret> -n <namespace>
```

Existence does not prove correctness:

``` text
SECRET EXISTS ≠ SECRET CONTENT CORRECT
SECRET EXISTS ≠ WORKLOAD USES SECRET
Correct Secret Reference ≠ Valid Credential
```

------------------------------------------------------------------------

## 11. Base64 Is Not Encryption

Kubernetes Secret representations commonly use base64 encoding.

``` text
BASE64 ENCODING ≠ ENCRYPTION
```

Also:

``` text
KUBERNETES SECRET OBJECT ≠ PROOF OF ENCRYPTION AT REST
```

Encryption at rest is a separate cluster-security concern.

------------------------------------------------------------------------

## 12. Secret-Safe Investigation

Routine production validation should not decode credentials.

``` text
SECRET EXISTS ≠ SECRET SHOULD BE DECODED
CONFIGURATION VALIDATION ≠ SECRET EXTRACTION
```

Collect only necessary metadata:

-   Secret name
-   Namespace
-   Type
-   Workload reference
-   Required key name when policy permits
-   Materialization state
-   Controller conditions
-   Relevant timestamps

Do not collect:

-   Passwords
-   Tokens
-   API keys
-   Private keys
-   Decoded Secret values

The goal is to prove configuration delivery, not print credentials.

------------------------------------------------------------------------

## 13. Do Not Dump the Container Environment

Avoid unrestricted runtime commands such as a broad `printenv` during
routine production validation.

They can expose passwords, tokens, API keys, cloud credentials, database
credentials, and internal service information.

``` text
RUNTIME CONFIGURATION VALIDATION ≠ FULL ENVIRONMENT DUMP
```

Prefer controller specifications, Pod specifications, configuration
references, source-object metadata, conditions, events, and
application-safe observability.

------------------------------------------------------------------------

## 14. Non-Secret Does Not Mean Unrestricted

ConfigMaps may contain internal endpoints, hostnames, architecture
details, environment topology, and operational identifiers.

``` text
NON-SECRET ≠ UNRESTRICTED DISCLOSURE
```

Use the minimum evidence necessary for diagnosis.

------------------------------------------------------------------------

## 15. Secret Delivery Is a Security Design Decision

Do not teach one universal Secret-delivery mechanism as always safest.

``` text
SECRET DELIVERY MECHANISM = SECURITY DESIGN DECISION
```

and:

``` text
ENVIRONMENT VARIABLE CONVENIENCE
        ≠
UNIVERSALLY SAFEST SECRET DELIVERY
```

The broader security design is addressed later in the security track.

------------------------------------------------------------------------

## 16. External Secret Systems

Some environments use External Secrets Operator, Vault, cloud secret
managers, CSI-based integrations, or other secret-management systems.

Do not assume one mechanism is mandatory.

``` text
DISCOVER ACTUAL SECRET DELIVERY MECHANISM
```

Conceptually:

``` text
External Provider
      ↓
Secret Integration / Controller
      ↓
Kubernetes Secret
      ↓
Workload Reference
      ↓
Pod
      ↓
Application
```

A more detailed reconciliation path is:

``` text
External provider
        ↓
Provider authentication
        ↓
Remote object/property lookup
        ↓
Secret controller reconciliation
        ↓
Kubernetes Secret materialization
        ↓
Workload reference
        ↓
Pod delivery
        ↓
Application consumption
```

Every transition must be independently proven.

------------------------------------------------------------------------

## 17. Controller Health vs Reconciliation Health

A healthy secret-controller Pod does not prove that an individual secret
reconciled successfully.

``` text
SECRET CONTROLLER RUNNING
        ≠
SECRET RECONCILIATION SUCCEEDED

ExternalSecret Exists
        ≠
ExternalSecret Ready

ExternalSecret Ready
        ≠
Application Authentication Works
```

When external-secret materialization fails, inspect status, conditions,
events, SecretStore or ClusterSecretStore references, reconciliation
state, and controller logs when necessary before interacting with
provider credentials.

Possible failure boundaries include:

``` text
Provider authentication
Provider authorization
Remote object missing
Remote property missing
Provider connectivity
Store configuration
Secret materialization
Workload reference
Application authentication
```

------------------------------------------------------------------------

## 18. Optional Configuration

A missing ConfigMap, Secret, or key does not universally imply that a
Pod must fail.

Determine whether the reference is:

``` text
Required
Optional
Unknown
```

Then determine whether the application has a default, fallback behavior,
degraded behavior, or fatal startup requirement.

``` text
MISSING CONFIGURATION ≠ AUTOMATIC OUTAGE
POD RUNNING ≠ REQUIRED BUSINESS CONFIGURATION PRESENT
```

------------------------------------------------------------------------

## 19. Immutable ConfigMaps and Secrets

Kubernetes supports immutable ConfigMaps and Secrets.

If:

``` yaml
immutable: true
```

is configured, modification behavior is intentionally restricted.

``` text
CONFIGURATION OBJECT EXISTS ≠ CONFIGURATION OBJECT IS MUTABLE
FAILED CONFIG UPDATE ≠ AUTOMATICALLY RBAC FAILURE
```

Check immutability before diagnosing an update failure as an
authorization failure.

------------------------------------------------------------------------

## 20. Controller Desired State vs Running Pod State

Always distinguish current controller configuration from the state of a
running Pod.

``` text
CURRENT DEPLOYMENT CONFIGURATION
        ≠
PROOF OF RUNNING POD CONFIGURATION

CONTROLLER DESIRED STATE
        ≠
RUNNING POD STATE
```

Useful evidence includes:

``` text
ConfigMap / Secret resourceVersion
Configuration modification timestamp
Controller generation
Controller observedGeneration
ReplicaSet / revision when applicable
Pod creation timestamp
Container restart count
Incident start timestamp
```

However:

``` text
KUBERNETES resourceVersion ≠ APPLICATION CONFIGURATION VERSION
```

Treat `resourceVersion` as Kubernetes-object evidence, not an
application configuration version.

------------------------------------------------------------------------

## 21. Application Configuration Precedence

Applications may combine defaults, files, environment variables,
command-line arguments, and remote configuration.

Example:

``` text
ConfigMap:
LOG_LEVEL=INFO

Command line:
--log-level=DEBUG
```

If the application gives command-line arguments higher precedence, the
effective value is `DEBUG`.

``` text
CONFIGMAP VALUE ≠ EFFECTIVE APPLICATION VALUE
EXPECTED SOURCE ≠ EFFECTIVE SOURCE
```

Troubleshooting must identify the effective source rather than the most
visible Kubernetes object.

------------------------------------------------------------------------

## 22. Startup-Only vs Dynamic Configuration

Classify the application reload model:

``` text
STARTUP-ONLY
DYNAMIC
UNKNOWN
```

This determines whether a source update can affect a running process.

``` text
SOURCE UPDATED ≠ APPLICATION UPDATED
```

------------------------------------------------------------------------

## 23. Application Parsing Failures

Kubernetes can deliver configuration successfully while the application
rejects it.

Example:

``` text
ConfigMap exists              PROVEN
Required key exists           PROVEN
Pod reference                 PROVEN
Container starts              PROVEN
Application parses value      FAILED
```

Then:

``` text
Lowest Proven Healthy Layer:
Runtime configuration delivery

First Failed Transition:
Runtime configuration → Application configuration parser
```

Do not modify the ConfigMap blindly when delivery has already been
proven.

------------------------------------------------------------------------

## 24. Configuration Ownership

A Kubernetes object may be downstream of another desired-state system.

Possible owners include:

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
Manual Kubernetes configuration
UNKNOWN
```

``` text
KUBERNETES OBJECT ≠ AUTHORITATIVE CONFIGURATION SOURCE
```

Before recommending a change, ask:

``` text
WHO OWNS THIS CONFIGURATION?
```

If the authoritative owner is unknown:

``` text
NO DIRECT CONFIGURATION CHANGE
```

until ownership is established.

------------------------------------------------------------------------

## 25. Reconciliation and Manual Fixes

Consider:

``` text
Engineer patches Kubernetes object
        ↓
Application recovers
        ↓
GitOps / operator reconciliation
        ↓
Authoritative desired state restored
        ↓
Failure returns
```

Therefore:

``` text
MANUAL FIX WORKED ≠ AUTHORITATIVE CONFIGURATION FIXED
TEMPORARY RECOVERY ≠ DURABLE REMEDIATION
```

Keep the production rule:

``` text
DO NOT PATCH THE SYMPTOM
WHEN ANOTHER SYSTEM OWNS DESIRED STATE
```

------------------------------------------------------------------------

## 26. Configuration Drift

Compare:

``` text
Expected source      vs Observed source
Expected object      vs Observed object
Expected key         vs Observed key
Expected namespace   vs Observed namespace
Desired state        vs Rendered state
Rendered state       vs Pod state
Delivered state      vs Effective application state
```

But:

``` text
Observed Drift ≠ Permission to Patch
```

------------------------------------------------------------------------

## 27. Shared Configuration and Blast Radius

A ConfigMap or Secret may be shared by multiple consumers.

``` text
ConfigMap
   ├── Service A
   ├── Service B
   └── CronJob C
```

``` text
Secret
   ├── API
   ├── Worker
   └── Batch Job
```

Therefore:

``` text
CONFIGMAP CHANGE ≠ SINGLE-WORKLOAD CHANGE
SECRET CHANGE ≠ SINGLE-WORKLOAD CHANGE
CONFIGURATION CHANGE ≠ SINGLE-WORKLOAD CHANGE
```

unless consumer scope has been proven.

Secret rotation can also affect an external authentication system.

``` text
SECRET ROTATION
        =
CONFIGURATION + SECURITY + AVAILABILITY EVENT
```

------------------------------------------------------------------------

## 28. Incident Timeline

Configuration incidents benefit from explicit timelines.

``` text
T1 Configuration changed
T2 Controller reconciled
T3 New Pod created
T4 Old Pod terminated
T5 Application became unhealthy
T6 Alert fired
T7 Investigation started
```

Ask:

``` text
Which transition correlates with the symptom?
```

But:

``` text
TIMING CORRELATION ≠ ROOT CAUSE PROOF
```

------------------------------------------------------------------------

## 29. Restart Is Remediation, Not Diagnosis

A restart changes state. It may consume newer configuration, move the
workload to another node, refresh credentials, clear process state,
rebuild connections, or replace stale Pods.

``` text
RESTART = STATE-CHANGING REMEDIATION
```

not:

``` text
RESTART = READ-ONLY DIAGNOSTIC TEST
```

If a replacement Pod becomes healthy:

``` text
RESTART RESTORED SERVICE ≠ ROOT CAUSE IDENTIFIED
SERVICE RESTORED ≠ ROOT CAUSE REMEDIATED
Replacement Pod Healthy ≠ Root Cause Proven
```

Preserve evidence before approved remediation.

------------------------------------------------------------------------

## 30. Avoid `kubectl exec` as the Default

Runtime execution can require additional authorization and can expose
sensitive state.

Prefer:

``` text
Controller
   ↓
Pod specification
   ↓
Configuration references
   ↓
Source objects
   ↓
Conditions / events / logs
```

Use runtime execution only when necessary, approved, and safe.

------------------------------------------------------------------------

## 31. Lowest Proven Healthy Layer

Example:

``` text
ExternalSecret definition       PROVEN
SecretStore                     PROVEN
Remote retrieval                PROVEN
Kubernetes Secret               PROVEN
Pod reference                   PROVEN
Application authentication      DENIED
```

Then:

``` text
Lowest Proven Healthy Layer:
Workload configuration delivery
```

Do not repeatedly investigate already-proven upstream layers.

------------------------------------------------------------------------

## 32. First Failed Transition

For the previous example:

``` text
Application credential
        ↓
External service authentication
        X
```

Record:

``` text
First Failed Transition:
Application credential → External authentication
```

This is more actionable than a generic label such as `Secret problem`.

------------------------------------------------------------------------

## 33. Minimum Supported Blast Radius

Use the smallest scope supported by evidence:

``` text
One key
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

If two older Pods fail while a newly created Pod is healthy, evidence
may support:

``` text
Minimum Supported Blast Radius:
Pods from the older configuration generation
```

Do not expand scope without evidence.

------------------------------------------------------------------------

## 34. Evidence Confidence

Continue the standard evidence model:

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

``` text
ASSUMED ≠ ROOT CAUSE
```

------------------------------------------------------------------------

## 35. Current Actionable Owner

Determine the actionable owner from:

``` text
First Failed Transition
+
Authoritative Configuration Owner
```

Example:

``` text
Symptom:
Application authentication failure

Kubernetes Secret:
PROVEN

Workload reference:
PROVEN

Pod configuration:
CURRENT

External authentication:
DENIED

Authoritative credential source:
External secret provider

Current Actionable Owner:
Secret-management owner

Requested Action:
Validate expected credential version and approved rotation path
```

``` text
SYMPTOM ≠ FAILURE DOMAIN ≠ OWNER
```

------------------------------------------------------------------------

## 36. Production Scenarios

### Scenario A --- Missing ConfigMap

``` text
Deployment                         PROVEN
ConfigMap reference                PROVEN
ConfigMap                          NOT FOUND
Pod configuration                 UNSATISFIED
```

Possible result:

``` text
Lowest Proven Healthy Layer:
Workload configuration reference

First Failed Transition:
ConfigMap reference → ConfigMap lookup
```

Verify required/optional semantics before declaring outage.

### Scenario B --- Missing Key

``` text
ConfigMap                    PROVEN
Workload reference           PROVEN
Required key                 NOT FOUND
```

Possible:

``` text
First Failed Transition:
ConfigMap → Required configuration key
```

### Scenario C --- Wrong ConfigMap Reference

``` text
Expected:
model-runtime-config

Observed:
model-runtime-config-old
```

If the workload is proven to reference the old object:

``` text
First Failed Transition:
Desired configuration → rendered workload reference
```

### Scenario D --- Mixed Replicas

``` text
ConfigMap changed: 14:00

Pod A created: 13:30 → failing
Pod B created: 13:31 → failing
Pod C created: 14:05 → healthy
```

With environment-variable delivery, investigate configuration-generation
divergence.

Possible:

``` text
Minimum Supported Blast Radius:
Pods created before configuration update
```

### Scenario E --- Stale `subPath`

``` text
ConfigMap updated             PROVEN
Pod reference                 PROVEN
Volume mount                  PROVEN
subPath                       PROVEN
Application sees old config   SUPPORTED
```

Possible:

``` text
First Failed Transition:
Updated ConfigMap → running subPath-mounted content
```

### Scenario F --- External Secret Reconciliation Failure

``` text
ExternalSecret                PROVEN
SecretStore                   PROVEN
Controller running            PROVEN
Remote lookup                 DENIED
Kubernetes Secret             NOT UPDATED
Application                   FAILING
```

Possible:

``` text
Lowest Proven Healthy Layer:
External secret reconciliation request

First Failed Transition:
Secret controller → external provider authorization
```

### Scenario G --- Secret Exists but Authentication Fails

``` text
Secret                        PROVEN
Workload reference            PROVEN
Pod configuration             PROVEN
Application started           PROVEN
External authentication       DENIED
```

Possible remaining causes include expired credentials, incorrect
credentials, wrong credential version, wrong endpoint, application
consumption issues, or external authorization.

Do not decode the Secret as the automatic next step.

### Scenario H --- Application Precedence

``` text
ConfigMap                     PROVEN
Required key                  PROVEN
Pod reference                 PROVEN
Pod generation                CURRENT
CLI override                  PROVEN
```

Possible:

``` text
First Failed Transition:
Expected configuration source → application precedence
```

### Scenario I --- Restart Restored Service

``` text
Old Pod failing
      ↓
Pod replaced
      ↓
New Pod healthy
```

Possible changed factors include configuration generation, credential,
node, application process state, connection state, or external
dependency state.

``` text
Replacement Pod Healthy ≠ Root Cause Proven
```

------------------------------------------------------------------------

## 37. Production Configuration Evidence Handoff Contract

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

## 38. Production Investigation Workflow

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
APPROVED REMEDIATION
        ↓
REGRESSION VALIDATION
```

------------------------------------------------------------------------

## 39. Canonical Production Rules

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

CONFIGMAP ≠ SECRET

CONFIDENTIAL DATA ≠ CONFIGMAP DATA

CONFIGMAP NAME
≠
COMPLETE CONFIGURATION IDENTITY

ConfigMap Exists
≠
Workload Uses ConfigMap

ConfigMap Exists
≠
Required Key Exists

KEY EXISTS IN CONFIGMAP
≠
KEY BECAME CONTAINER ENVIRONMENT VARIABLE

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

CONFIGMAP UPDATED
≠
MOUNT UPDATED IMMEDIATELY

MOUNT CURRENT
≠
APPLICATION CONFIGURATION CURRENT

MOUNT UPDATED
≠
APPLICATION RELOADED

Normal Volume Update Behavior
≠
subPath Update Behavior

SECRET EXISTS
≠
SECRET CONTENT CORRECT

SECRET EXISTS
≠
WORKLOAD USES SECRET

SECRET EXISTS
≠
SECRET SHOULD BE DECODED

CONFIGURATION VALIDATION
≠
SECRET EXTRACTION

Correct Secret Reference
≠
Valid Credential

BASE64 ENCODING
≠
ENCRYPTION

KUBERNETES SECRET OBJECT
≠
PROOF OF ENCRYPTION AT REST

RUNTIME CONFIGURATION VALIDATION
≠
FULL ENVIRONMENT DUMP

NON-SECRET
≠
UNRESTRICTED DISCLOSURE

SECRET CONTROLLER RUNNING
≠
SECRET RECONCILIATION SUCCEEDED

ExternalSecret Exists
≠
ExternalSecret Ready

ExternalSecret Ready
≠
Application Authentication Works

MISSING CONFIGURATION
≠
AUTOMATIC OUTAGE

POD RUNNING
≠
REQUIRED BUSINESS CONFIGURATION PRESENT

CONFIGURATION OBJECT EXISTS
≠
CONFIGURATION OBJECT IS MUTABLE

FAILED CONFIG UPDATE
≠
AUTOMATICALLY RBAC FAILURE

CURRENT DEPLOYMENT CONFIGURATION
≠
PROOF OF RUNNING POD CONFIGURATION

CONTROLLER DESIRED STATE
≠
RUNNING POD STATE

KUBERNETES resourceVersion
≠
APPLICATION CONFIGURATION VERSION

CONFIGMAP VALUE
≠
EFFECTIVE APPLICATION VALUE

EXPECTED SOURCE
≠
EFFECTIVE SOURCE

SOURCE UPDATED
≠
APPLICATION UPDATED

KUBERNETES OBJECT
≠
AUTHORITATIVE CONFIGURATION SOURCE

Observed Drift
≠
Permission to Patch

DO NOT PATCH THE SYMPTOM
WHEN ANOTHER SYSTEM OWNS DESIRED STATE

MANUAL FIX WORKED
≠
AUTHORITATIVE CONFIGURATION FIXED

TEMPORARY RECOVERY
≠
DURABLE REMEDIATION

CONFIGURATION CHANGE
≠
SINGLE-WORKLOAD CHANGE

SECRET ROTATION
=
CONFIGURATION + SECURITY + AVAILABILITY EVENT

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

## 40. Acceptance Standard

A production engineer completing Part 2.5 should be able to answer:

``` text
WHAT workload is affected?

WHAT configuration was expected?

WHAT is the authoritative configuration source?

WHAT Kubernetes configuration was rendered?

WHAT configuration did the affected Pod receive?

HOW was it delivered?

WHEN was the configuration changed?

WHEN was the affected Pod created?

ARE replicas running different configuration generations?

IS the reference required or optional?

IS the object immutable?

IS subPath involved?

DOES the application reload configuration dynamically?

WHAT configuration source has effective precedence?

IS configuration shared with other workloads?

WHAT is the Lowest Proven Healthy Layer?

WHERE is the First Failed Transition?

WHAT is the Minimum Supported Blast Radius?

WHO owns authoritative desired state?

WHO is the Current Actionable Owner?

WERE Secret values exposed?
NO

WAS the environment dumped?
NO

WAS runtime configuration mutated?
NO

WAS the workload restarted during diagnosis?
NO
```

------------------------------------------------------------------------

## Summary

Runtime configuration troubleshooting is not simply checking whether a
ConfigMap or Secret exists. Production investigation must trace
configuration from its authoritative desired-state source through
Kubernetes rendering and Pod delivery to the configuration actually used
by the application.

The most important operational principle is:

``` text
DESIRED CONFIGURATION
        ≠
RENDERED CONFIGURATION
        ≠
DELIVERED CONFIGURATION
        ≠
EFFECTIVE CONFIGURATION
```

Always prove the first failed transition, preserve secret safety,
establish blast radius, identify the authoritative owner, and avoid
state-changing remediation until the evidence has been captured.
