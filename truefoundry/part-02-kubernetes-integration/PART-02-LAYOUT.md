# Part 2 — Kubernetes Integration — Master Layout

## Part Objective

Part 2 provides a production-focused understanding of how TrueFoundry integrates with Kubernetes.

The part deliberately separates platform concepts from Kubernetes primitives so that readers can troubleshoot using evidence rather than assuming that the layer where an error is displayed is the layer that failed.

## Scope Boundary

Part 2 covers the Kubernetes integration foundation:

- Compute Plane onboarding
- `tfy-agent` and cluster integration
- namespaces and workload placement
- ServiceAccounts and Kubernetes RBAC
- ConfigMaps, Secrets, and runtime configuration
- Services, ingress, DNS, and TLS
- persistent storage

The following subjects are introduced only where required and covered deeply in later parts:

- CPU/memory scheduling and node placement — Part 3
- GPU infrastructure — Part 4
- application deployment — Part 5
- model serving — Parts 6–7
- scaling/performance — Part 8
- advanced networking — Part 9
- broader security architecture — Part 10
- observability — Part 11
- production operations — Part 12
- incident troubleshooting — Parts 13–14

This boundary avoids unnecessary duplication.

---

## 2.1 — Connecting a Kubernetes Cluster to TrueFoundry

### Objective

Explain how a Kubernetes cluster becomes a TrueFoundry Compute Plane and establish the production prerequisites and validation model for cluster onboarding.

### Core Topics

- Control Plane and Compute Plane relationship
- cluster identity
- supported Kubernetes prerequisites
- cluster registration/onboarding concepts
- cluster integration components
- outbound connectivity
- Kubernetes permissions
- namespace considerations
- capacity prerequisites
- reconciliation
- installation success vs integration health
- production onboarding record

### SRE Focus

- verify the target before mutation
- distinguish management, deployment, and runtime impact
- identify onboarding failure domains
- validate the integration layer-by-layer
- establish operational ownership

### Lab

`labs/01-connecting-kubernetes-cluster-check.md`

The production validation portion must be `[SAFE-READ]`.

---

## 2.2 — Understanding `tfy-agent` Deployment

### Objective

Explain the role of `tfy-agent` in the cluster integration path without treating it as the entire Compute Plane or the entire TrueFoundry platform.

### Core Topics

- `tfy-agent` purpose
- placement in the architecture
- deployment/runtime identity
- communication toward the Control Plane
- Kubernetes resources associated with the agent
- configuration references
- health and readiness evidence
- logs and events
- lifecycle and upgrade considerations
- version-aware vendor terminology

### SRE Focus

- agent running vs integration healthy
- DNS/network/TLS failure boundaries
- authorization failures
- scheduling and container startup
- desired vs observed state
- safe evidence collection

### Lab

`labs/02-understanding-tfy-agent-deployment-check.md`

---

## 2.3 — Namespaces & Workload Placement

### Objective

Explain Kubernetes namespace boundaries and how TrueFoundry-managed workloads are located and correlated with Kubernetes resources.

### Core Topics

- Kubernetes namespaces
- TrueFoundry Workspace vs namespace
- platform namespaces
- workload namespaces
- namespace isolation
- resource discovery
- labels and annotations
- workload-to-resource correlation
- multi-environment namespace strategies

### SRE Focus

- avoid assuming Workspace = namespace
- identify the exact runtime target
- determine workload blast radius
- correlate platform and Kubernetes identities

### Lab

`labs/03-namespaces-workload-placement-check.md`

---

## 2.4 — ServiceAccounts, RBAC & Permissions

### Objective

Explain Kubernetes workload identity and authorization as a distinct failure domain from TrueFoundry authorization and cloud IAM.

### Core Topics

- ServiceAccounts
- Roles
- ClusterRoles
- RoleBindings
- ClusterRoleBindings
- effective authorization
- namespace-scoped vs cluster-scoped access
- least privilege
- permission troubleshooting

### SRE Focus

Maintain the distinction:

```text
TrueFoundry Authorization
        ≠
Kubernetes RBAC
        ≠
Cloud IAM
```

Troubleshooting should identify which authorization boundary rejected the operation before changing permissions.

### Lab

`labs/04-serviceaccounts-rbac-permissions-check.md`

Use safe authorization inspection such as `kubectl auth can-i` where appropriate; do not change bindings in the production validation lab.

---

## 2.5 — ConfigMaps, Secrets & Runtime Configuration

### Objective

Explain how runtime configuration reaches Kubernetes workloads and how SREs can validate configuration safely without exposing credentials.

### Core Topics

- ConfigMaps
- Secrets
- environment variables
- mounted configuration
- references from Pod specifications
- configuration ownership
- authoritative source
- desired vs observed configuration
- reconciliation and drift
- rollout implications

### SRE Focus

- never print Secret values during routine validation
- distinguish Secret existence from Secret correctness
- identify configuration ownership
- compare references and metadata safely
- remediate through the authoritative source

### Lab

`labs/05-configmaps-secrets-runtime-configuration-check.md`

The lab must not decode or display Secret data.

---

## 2.6 — Services, Ingress, DNS & TLS

### Objective

Establish the Kubernetes network path beneath TrueFoundry-managed endpoints.

### Core Topics

- Kubernetes Services
- selectors and endpoints
- ingress/gateway concepts
- internal and external access
- DNS resolution
- TLS certificates
- certificate trust
- network path correlation
- endpoint dependencies

### SRE Focus

Use the layered principle:

```text
Pod Healthy
    ≠
Service Reachable
    ≠
Endpoint Healthy
    ≠
Business Transaction Healthy
```

Identify the First Failed Transition across the request path rather than assigning root cause from the visible symptom.

### Lab

`labs/06-services-ingress-dns-tls-check.md`

---

## 2.7 — Storage, PVCs & Persistent Workloads

### Objective

Explain Kubernetes persistent storage dependencies relevant to TrueFoundry workloads.

### Core Topics

- StorageClasses
- PersistentVolumes
- PersistentVolumeClaims
- dynamic provisioning
- access modes
- volume binding
- attachment and mount lifecycle
- capacity
- reclaim behavior
- stateful workload considerations

### SRE Focus

Differentiate:

```text
PVC Requested
    ↓
PVC Bound
    ↓
Volume Attached
    ↓
Volume Mounted
    ↓
Application Can Use Storage
```

A failure at one stage must not be generalized to the entire storage stack.

### Lab

`labs/07-storage-pvcs-persistent-workloads-check.md`

---

# Standard Tutorial Workflow

Every Part 2 tutorial follows:

```text
Draft
  ↓
Technical/Vendor Validation
  ↓
Production/SRE Review
  ↓
Revised Final
  ↓
Canonical
  ↓
Hands-On Lab
  ↓
Repository Validation
```

## Draft

Create the full teaching flow and operational model.

## Technical/Vendor Validation

Validate version-sensitive TrueFoundry statements against current official TrueFoundry documentation.

Validate Kubernetes behavior against current official Kubernetes documentation when applicable.

Avoid presenting tutorial-created SRE methods as TrueFoundry product terminology.

## Production/SRE Review

Review for:

- production safety
- operational completeness
- failure-domain clarity
- observability
- evidence quality
- ownership boundaries
- escalation readiness
- authoritative remediation
- recovery validation

## Revised Final

Incorporate technical and SRE review findings.

## Canonical

Use the explicit canonical filename defined by the Part 2 naming convention.

Do not replace tutorial filenames with generic `README.md`.

## Hands-On Lab

Create the matching lab under `labs/`.

Production validation labs are `[SAFE-READ]` unless explicitly identified as an approved mutation exercise.

## Repository Validation

Validate:

- expected file paths
- naming convention
- no accidental files
- tutorial/lab pairing
- links and navigation where applicable
- Git diff
- staged scope
- clean commit
- push to `main`

---

# Naming Convention

## Tutorial

```text
NN-topic-name/NN-topic-name.md
```

Example:

```text
01-connecting-kubernetes-cluster/
└── 01-connecting-kubernetes-cluster.md
```

## Lab

```text
labs/NN-topic-name-check.md
```

Example:

```text
labs/01-connecting-kubernetes-cluster-check.md
```

---

# Production Safety Standard

For production investigation:

```text
OBSERVE BEFORE MUTATE
```

and:

```text
UNKNOWN TARGET = NO MUTATION
```

Safe evidence normally includes:

```text
kubectl config current-context
kubectl get
kubectl describe
kubectl logs
kubectl auth can-i
```

where the exact command does not expose sensitive data or change state.

Mutating commands must not be included in `[SAFE-READ]` production validation.

---

# Part 2 Operational Model

The recurring investigation model is:

```text
TrueFoundry Control Plane
          ↓
Cluster Integration
          ↓
Kubernetes API / Reconciliation
          ↓
Scheduling
          ↓
Node / Container Runtime
          ↓
Kubernetes Workload
          ↓
Service / Network / Storage Dependencies
          ↓
User Transaction
```

For an incident:

```text
Verify Deployment Identity
        ↓
Determine Minimum Supported Blast Radius
        ↓
Collect Timestamped Evidence
        ↓
Find Lowest Proven Healthy Layer
        ↓
Find First Failed Transition
        ↓
Classify Failure Domain
        ↓
Define Next Required Action
        ↓
Identify Current Actionable Owner
        ↓
Remediate Through Authoritative Source
        ↓
Validate End-to-End Recovery
```

---

# Part 2 Completion Gate

Part 2 is complete when tutorials 2.1 through 2.7 each have:

```text
Draft                         ✅
Technical/Vendor Validation  ✅
Production/SRE Review         ✅
Revised Final                 ✅
Canonical                     ✅
Hands-On Lab                  ✅
Repository Validation         ✅
```

After 2.7 repository validation:

1. update `truefoundry/STATUS-TRACKER.md` once for the completed Part 2;
2. validate the final Part 2 repository diff;
3. commit the completed tracker state;
4. push and verify `main`;
5. proceed to Part 3 only after Part 2 is closed.
