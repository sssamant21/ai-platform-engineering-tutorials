# Part 1.5 — Workspaces, Environments & Deployment Concepts

**Status:** Canonical  
**Track:** Part 1 — Foundations & Architecture  
**Audience:** SRE, Platform Engineering, DevOps, ML Infrastructure  
**Prerequisites:** Parts 1.1–1.4

## 1. Purpose

Part 1.5 establishes how an SRE identifies what workload is deployed, where it is deployed, which artifact and configuration it is running, and which authoritative source owns that deployment.

```text
TrueFoundry Control Plane
        ↓
Target Cluster / Compute Plane
        ↓
Workspace
        ↓
Workload
        ↓
Kubernetes Resources
        ↓
Runtime
```

An organization's lifecycle model can be overlaid on that structure:

```text
Development / Staging / Production
              ↓
     Target Compute Plane
              ↓
          Workspace
              ↓
           Workload
```

The exact mapping is organization-specific.

```text
Workspace ≠ Environment
Environment ≠ Cluster
Workspace ≠ Namespace
```

unless the deployed architecture explicitly establishes such a relationship.

## 2. Learning Objectives

After completing this tutorial, you should be able to:

- explain the operational purpose of a Workspace
- distinguish Workspace, environment, Compute Plane, cluster, and namespace
- identify the target execution environment for a workload
- understand deployment identity
- distinguish artifact identity from runtime configuration
- understand environment-specific configuration
- compare staging and production safely
- identify configuration provenance and ownership
- recognize deployment and environment drift
- determine deployment blast radius
- correlate deployments with incidents without prematurely declaring root cause
- distinguish a healthy runtime from a correct runtime
- identify the authoritative remediation path
- validate deployments end-to-end

## 3. Workspace

A Workspace provides a TrueFoundry deployment and organizational context for workloads.

```text
Compute Plane / Cluster
        ↓
Workspace
        ↓
Workloads
   ├── Service
   ├── Job
   ├── Model workload
   └── Other supported workloads
```

Workspaces can also participate in platform access and RBAC boundaries. Detailed RBAC mechanics are covered later in the security modules.

## 4. Workspace Is Not Automatically a Kubernetes Namespace

TrueFoundry and Kubernetes expose different views.

Platform view:

```text
Compute Plane
     ↓
Workspace
     ↓
Workload
```

Kubernetes view:

```text
Cluster
   ↓
Namespace
   ↓
Deployment / Job / StatefulSet
   ↓
Pod
   ↓
Container
```

Do not assume `Workspace = Namespace`. Discover the actual mapping from deployment and runtime evidence.

## 5. Operational Environment

In this tutorial, **environment** primarily means an operational lifecycle context such as development, testing, staging, or production. Do not assume that `Environment` is a universal first-class TrueFoundry platform object.

An organization might implement:

```text
Development → dev-cluster → dev-workspace
Production  → prod-cluster → prod-workspace
```

Another organization may use shared infrastructure. The mapping must be discovered rather than assumed.

## 6. Logical Environment vs Physical Infrastructure

Logical lifecycle concepts include development, staging, and production. Physical/runtime infrastructure may include cloud accounts or subscriptions, VPCs/VNets, Kubernetes clusters, node pools, GPU pools, namespaces, storage, and networks.

Therefore:

```text
Environment ≠ Cluster
Environment ≠ Namespace
```

unless architecture explicitly defines a one-to-one relationship.

## 7. Compute Plane

A TrueFoundry Compute Plane is the Kubernetes environment where workloads execute.

```text
TrueFoundry Control Plane
        │
        ├── Compute Plane A → Kubernetes A
        └── Compute Plane B → Kubernetes B
```

Multiple Compute Planes may be attached to the platform. They remain distinct runtime and failure boundaries.

```text
Compute Plane A healthy ≠ Compute Plane B healthy
```

## 8. Deployment Target

A deployment target answers: **Where should this workload execute?**

```text
Workload
   ↓
Deployment Configuration
   ↓
Target Compute Plane / Cluster
   ↓
Workspace
   ↓
Kubernetes
```

In multi-cluster architectures, the target should be treated as an explicit deployment decision. Do not assume that TrueFoundry automatically selects the optimal attached cluster or transparently performs cross-cluster failover.

## 9. Workload Portability

A workload definition may be deployable to different Compute Planes, but a portable workload definition does not guarantee a portable runtime.

Validate destination prerequisites:

```text
Target Compute Plane
       ↓
Prerequisites
   ├── CPU / Memory
   ├── GPU
   ├── Container registry access
   ├── Secrets
   ├── Storage
   ├── Networking
   ├── Identity / IAM
   └── Dependencies
       ↓
Deploy
```

Changing the target does not eliminate infrastructure differences.

## 10. Deployment Identity

For production operations:

```text
Operational Environment
        +
Compute Plane / Cluster
        +
Workspace
        +
Workload
        +
Artifact
        +
Configuration
        +
Deployment Version
        =
Deployment Identity
```

A statement such as `patient-api is failing` is insufficient for incident response. Determine which environment, Compute Plane, cluster, Workspace, namespace, workload, artifact, configuration, and deployment version are affected.

## 11. Intended Target vs Observed Target

A deployment can be healthy but deployed to the wrong place. Compare the intended operational environment, Compute Plane, Workspace, cluster, and namespace with what is actually observed.

The Pods may all be healthy while the deployment is still incorrect.

## 12. Environment Identity Before Investigation

Before production investigation, establish:

- Kubernetes context
- Compute Plane
- cluster
- Workspace
- namespace
- workload

Start with:

```bash
kubectl config current-context
```

Production operating rule:

```text
UNKNOWN TARGET = NO MUTATION
```

If environment identity cannot be proven, remain read-only.

## 13. Artifact

An artifact may be a container image, model artifact, or application package. The same artifact can potentially be deployed to multiple lifecycle environments. The artifact itself is not the environment.

## 14. Immutable Artifact Identity

Image tags alone may not prove artifact identity. For containers, compare repository, tag, and digest.

```text
STAGING
app:v2.4.1
sha256:AAA

PRODUCTION
app:v2.4.1
sha256:BBB
```

The tags match, but the artifacts do not.

```text
Same tag ≠ Proven same artifact
```

Where available, use the image digest as stronger evidence of immutable container identity.

## 15. Build Once, Promote

Where the organization's software-delivery model supports it, prefer:

```text
BUILD ONCE
    ↓
VERIFY
    ↓
PROMOTE
```

rather than independently rebuilding artifacts for every environment. This is an SRE/software-delivery practice rather than a requirement that every TrueFoundry workflow must follow.

## 16. Artifact vs Configuration

The same artifact may run with different configuration.

```text
Artifact
   +
Configuration
   +
Infrastructure
   +
Dependencies
   =
Runtime Behavior
```

Therefore, the same artifact does not imply the same runtime.

## 17. Environment-Specific Configuration

Environment-specific differences may include CPU, memory, GPU, replica count, autoscaling, node placement, environment variables, Secret references, ServiceAccounts, IAM/workload identity, network configuration, storage, endpoints, dependency endpoints, and observability configuration.

Differences should be intentional, controlled, and explainable.

## 18. Secret Isolation

Different environments should resolve credentials according to their security boundaries.

```text
Development → Development Secret Reference
Staging     → Staging Secret Reference
Production  → Production Secret Reference
```

```text
Same logical configuration key ≠ Same credential value
```

During investigation, inspect Secret references and metadata only when possible. Never place credential values in terminal history, Slack, incident channels, tickets, screenshots, tutorial labs, or RCA documents.

## 19. Configuration Provenance

When configuration is observed at runtime, determine where that configuration originated.

```text
Runtime Configuration
        ↑
Authoritative Source
```

Depending on architecture, the authoritative source may be TrueFoundry configuration, a Git repository, a GitOps repository, CI/CD configuration, Infrastructure-as-Code, or a Secret-management integration.

Do not assume the Kubernetes object itself is authoritative.

## 20. Configuration Ownership

Configuration source and configuration owner are separate concepts.

```text
Observed Difference
       ↓
Authoritative Source
       ↓
Configuration Owner
       ↓
Approved Remediation
```

A memory limit might be owned by the application team while GPU node placement may be owned by the platform/SRE team.

## 21. Desired vs Observed State

```text
Authoritative Configuration
          ↓
Desired Deployment
          ↓
Kubernetes
          ↓
Observed Runtime State
```

Troubleshooting compares all three.

## 22. Deployment Drift

If the authoritative source expects four replicas but Kubernetes shows two, possible causes include incomplete reconciliation, failed deployment, manual modification, wrong target, wrong configuration version, GitOps synchronization problems, or controller failure.

Drift is evidence. It is not itself a complete root-cause statement.

## 23. Environment Drift

Two environments may unintentionally differ even when the artifact is identical. Differences in memory, GPU type, capacity, networking, identity, storage, or dependencies can produce different runtime behavior.

## 24. Four-Way Environment Comparison

When comparing staging and production, compare four areas:

1. **Artifact** — image repository, tag, digest, model/application version.
2. **Configuration** — CPU, memory, GPU request, replicas, autoscaling, environment variables, Secret references, ServiceAccount, node placement.
3. **Infrastructure** — Compute Plane, cluster, node pool, GPU type, available capacity, storage, network.
4. **Dependencies** — database, Redis, object storage, external APIs, model registry, artifact registry, and other platform services.

## 25. Capacity Is Part of the Environment

Identical configuration can behave differently when available capacity differs.

```text
STAGING
GPU requested: 1
GPU available: 3

PRODUCTION
GPU requested: 1
GPU available: 0
```

```text
Same configuration ≠ Same schedulability
```

## 26. Staging Success Does Not Guarantee Production Success

Production can differ in traffic, concurrency, dataset size, compute capacity, GPU availability/type, networking, IAM, Secrets, storage, and external dependencies.

```text
Staging Success ≠ Guaranteed Production Success
```

Staging provides evidence; it does not eliminate production-specific failure domains.

## 27. Healthy Runtime vs Correct Runtime

Kubernetes may report a running and Ready Pod while the workload uses the wrong image, model, database, Secret reference, environment, or configuration.

```text
Healthy Runtime ≠ Correct Runtime
```

## 28. Deployment-Identity Failure Domain

Deployment-identity failures include:

- wrong artifact
- wrong configuration
- wrong Compute Plane
- wrong Workspace
- wrong namespace
- wrong deployment version
- wrong dependency
- wrong Secret reference
- wrong ServiceAccount

These failures can exist while Kubernetes reports healthy resources.

## 29. Deployment Change Correlation

Build a timeline:

```text
T0  Service healthy
T1  Deployment/configuration change
T2  New Pods created
T3  Readiness/traffic behavior changes
T4  Errors or latency begin
```

Temporal correlation is an investigation lead, not automatic proof of root cause.

## 30. Deployment Version and Rollout

During a rollout, distinguish old and new workload versions, Ready and unavailable replicas, and restarting Pods. Use deployment/version terminology appropriate to the actual TrueFoundry and Kubernetes objects being inspected.

## 31. Blast Radius

Before declaring a platform outage, determine whether the symptom affects one container, Pod, workload, namespace, Workspace, Compute Plane, multiple Compute Planes, or the entire platform.

A failure in one Compute Plane does not automatically imply failure in another.

## 32. Management Path vs Runtime Path

Management path:

```text
Engineer / CI/CD
        ↓
TrueFoundry Control Plane
        ↓
Cluster Integration
        ↓
Kubernetes
```

Runtime path:

```text
Client
  ↓
DNS / Gateway / Load Balancer
  ↓
Kubernetes Service
  ↓
Pod
  ↓
Application / Model
  ↓
Dependencies
```

A management-plane problem does not automatically prove that an already-running application's runtime request path has failed.

## 33. Evidence Hierarchy

Use evidence across layers:

1. Authoritative deployment configuration
2. TrueFoundry workload/deployment state
3. Kubernetes desired state
4. Kubernetes observed state
5. Kubernetes events
6. Container state
7. Application logs
8. Metrics and traces
9. Dependency evidence
10. User-facing transaction

No single layer automatically provides the complete answer.

## 34. Promotion Validation

```text
Artifact selected
       ↓
Target verified
       ↓
Configuration resolved
       ↓
Prerequisites validated
       ↓
Deployment reconciled
       ↓
Pods Ready
       ↓
Endpoint Ready
       ↓
Dependencies reachable
       ↓
Synthetic/business transaction succeeds
```

Only then is end-to-end deployment validation complete.

## 35. Rollback

Rollback should not automatically be the first response to every post-deployment problem.

```text
Incident after deployment
        ↓
Determine impact
        ↓
Correlate recent changes
        ↓
Capture critical evidence
        ↓
Rollback appropriate?
       / \
     Yes  No
      ↓    ↓
Rollback  Alternative remediation
      ↓
Validate
```

Where possible, rollback should use the authoritative deployment mechanism rather than an unmanaged Kubernetes mutation.

## 36. Rollback Validation

Validate the expected previous version, Pod readiness, endpoint health, dependencies, error rate, latency, and a business/synthetic transaction. A completed rollout command alone is not sufficient evidence of recovery.

## 37. Production Investigation Workflow

```text
SYMPTOM
   ↓
IDENTIFY
workload
   ↓
VERIFY TARGET
environment / Compute Plane / Workspace
   ↓
VERIFY DEPLOYMENT IDENTITY
artifact / digest / configuration / version
   ↓
DETERMINE BLAST RADIUS
   ↓
COMPARE
authoritative / desired / observed
   ↓
CORRELATE
recent changes
   ↓
PROVE
lowest healthy layer
   ↓
LOCATE
first failed transition
   ↓
CLASSIFY
failure domain + owner
   ↓
REMEDIATE
through authoritative source
   ↓
VALIDATE
runtime + dependencies + transaction
```

## 38. Production Incident Snapshot

```text
Timestamp:

Operational Environment:
Compute Plane:
Cluster:
Kubernetes Context:

Workspace:
Namespace:
Workload:

Artifact:
Image Tag:
Image Digest:
Deployment Version:

Authoritative Configuration Source:
Configuration Owner:

Desired Replicas:
Ready Replicas:

Pod Status:
Restart Count:
Node:

Recent Deployment:
Recent Configuration Change:

Events:
Application Evidence:
Infrastructure Evidence:
Dependency Evidence:

Lowest Proven Healthy Layer:
First Failed Transition:

Blast Radius:
Failure Domain:
Current Owner:

Remediation Path:
Validation Result:
```

Never record Secret values.

## 39. Incident Communication

Avoid vague statements such as `Production is broken.` Prefer scope-based evidence, for example: `The affected workload is isolated to the production Compute Plane. Other validated Compute Planes are not currently showing the same symptom.`

Avoid `Staging works, so infrastructure is fine.` Prefer: `Staging and production are running the same artifact digest. Configuration, infrastructure, capacity, and dependencies are being compared for environment-specific differences.`

Avoid declaring a deployment the root cause based only on timing. Treat it as a correlated change until artifact, configuration, runtime, and dependency evidence establishes causation.

## 40. Production Anti-Patterns

Avoid:

```text
Workspace = namespace
Environment = cluster
Same image tag = same artifact
Same artifact = same runtime
Staging success = production guarantee
Pod Ready = correct deployment
Deployment completed = application healthy
Recent deployment = proven root cause
kubectl state = authoritative configuration
Restart = root-cause analysis
Manual production mutation = permanent fix
```

## 41. Production Rules

1. Identify the operational environment before troubleshooting.
2. Verify the Compute Plane and Kubernetes context.
3. Never assume Workspace equals namespace.
4. Never assume environment equals cluster.
5. Establish complete deployment identity.
6. Prefer immutable artifact identity where available.
7. Compare artifact, configuration, infrastructure, and dependencies.
8. Include capacity when comparing environments.
9. Determine configuration provenance.
10. Determine configuration ownership.
11. Compare authoritative, desired, and observed state.
12. Treat drift as evidence requiring investigation.
13. Determine blast radius before declaring platform-wide impact.
14. Treat deployment timing as correlation until causation is proven.
15. Keep Secret values out of investigation artifacts.
16. Do not mutate an environment whose identity has not been proven.
17. Prefer remediation through the authoritative configuration path.
18. Preserve useful evidence before remediation when operationally appropriate.
19. Validate both runtime health and deployment correctness.
20. Validate the user-facing transaction before declaring recovery.

## 42. Final SRE Operating Model

```text
                 DEPLOYMENT INTENT

Operational Environment
          ↓
Target Compute Plane
          ↓
Workspace
          ↓
Workload
          ↓
Artifact + Configuration
          ↓
Authoritative Source
          ↓
Deployment / Reconciliation
          ↓
Kubernetes
          ↓
Runtime
          ↓
Dependencies
          ↓
User Transaction
```

During an incident:

```text
IDENTIFY
   ↓
VERIFY TARGET
   ↓
VERIFY DEPLOYMENT
   ↓
DETERMINE BLAST RADIUS
   ↓
COMPARE
   ↓
CORRELATE
   ↓
PROVE
   ↓
CLASSIFY
   ↓
REMEDIATE
   ↓
VALIDATE
```

The central production principle is:

```text
Healthy Runtime
      ≠
Correct Deployment
      ≠
Healthy End-to-End Service
```

All three must be validated.

## Next Tutorial

**Part 1.6 — SRE Ownership, Failure Domains & Support Model**
