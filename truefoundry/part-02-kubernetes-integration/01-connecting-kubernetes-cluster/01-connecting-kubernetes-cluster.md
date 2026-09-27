# Part 2.1 — Connecting a Kubernetes Cluster to TrueFoundry

## Purpose

Connecting Kubernetes to TrueFoundry establishes a **Compute Plane** where application, model, and AI workloads can execute.

Production onboarding must establish the correct target, supported integration model, operational prerequisites, validation evidence, failure boundaries, and ownership before the Compute Plane is accepted for production use.

```text
Verified Target
      ↓
Supported Integration
      ↓
Healthy Compute Plane
      ↓
Validated Workload Capability
      ↓
Operational Ownership
```

## Architecture

```text
TrueFoundry Control Plane
          │
          │ Management / orchestration
          ↓
Cluster Integration
      tfy-agent
      supporting components
          │
          ↓
Kubernetes Cluster
     Compute Plane
          │
          ├── Kubernetes API
          ├── CPU / GPU nodes
          ├── Networking
          ├── Storage
          ├── Platform add-ons
          └── Workloads
```

Keep these boundaries clear:

```text
TrueFoundry Control Plane ≠ Kubernetes Control Plane
tfy-agent ≠ Compute Plane
```

`tfy-agent` is part of the integration between the Compute Plane and TrueFoundry.

## Compute Plane

The Compute Plane is the customer Kubernetes environment in which workloads execute. Depending on configuration and enabled capabilities, it can include cluster integration, delivery/reconciliation, networking, autoscaling, observability, workflow and GPU components.

Technologies such as Argo CD, Argo Workflows, KEDA, Istio, or NVIDIA GPU Operator may participate depending on the supported deployment architecture. Do not assume every component is mandatory.

## New Cluster vs Existing Cluster

Before onboarding, determine whether the organization is creating a new Kubernetes cluster or attaching an existing cluster.

For an existing cluster, establish:

```text
What already exists?
What will TrueFoundry install?
What will TrueFoundry integrate with?
What remains externally managed?
Could multiple controllers manage the same resource?
```

## Provider-Specific Onboarding

The exact procedure can differ across AWS/EKS, Azure/AKS, GCP/GKE, OpenShift, on-premises Kubernetes, and other supported environments.

Use:

```text
COMMON PLATFORM MODEL
        +
PROVIDER-SPECIFIC PROCEDURE
```

Validate the current vendor-supported procedure for the target environment before performing installation.

## Deployment Identity

Record the expected target:

```text
Operational Environment:
Cloud Provider:
Account / Subscription / Project:
Region:
Kubernetes Cluster:
Kubernetes Version:
TrueFoundry Control Plane:
Compute Plane:
Expected Workspace:
Expected Namespace Mapping:
```

A useful identity model is:

```text
Environment + Cloud Account/Subscription/Project + Region
+ Cluster + Compute Plane + Workspace + Namespace
= Deployment Target
```

## Verify the Kubernetes Target

Start with safe observations:

```bash
kubectl config current-context
kubectl cluster-info
kubectl get nodes
```

Do not proceed merely because `kubectl` works. Prove that it points to the intended target.

```text
UNKNOWN TARGET = NO MUTATION
```

## Workspace and Namespace Mapping

Do not infer Kubernetes namespace placement solely from a Workspace name. Verify the actual mapping for the deployed TrueFoundry version and configuration.

```text
TrueFoundry Workspace
        ↓
Configured Deployment Target
        ↓
Kubernetes Namespace
```

## Common Prerequisites

Validate:

- supported Kubernetes version and API health
- healthy nodes
- CPU and memory capacity
- GPU capacity where required
- network connectivity, DNS and TLS
- cloud IAM and Kubernetes RBAC
- container registry access
- storage
- ingress/gateway prerequisites
- observability

Also validate provider-specific prerequisites from current vendor documentation.

## Pre-Change Baseline

Before onboarding an existing cluster, capture the current state:

- cluster identity and Kubernetes version
- node count, readiness and node pools
- CPU/memory and GPU resources
- namespaces and StorageClasses
- ingress/gateway
- GitOps and autoscaling components
- GPU components
- existing TrueFoundry components
- relevant Kubernetes events

```text
BEFORE STATE
      ↓
ONBOARDING CHANGE
      ↓
AFTER STATE
```

## Existing Component Inventory

Inventory components such as Argo CD, ingress/gateway implementations, cert-manager, KEDA, Prometheus, NVIDIA components, CSI drivers, External Secrets, service mesh and DNS controllers.

Classify each as:

```text
Already Exists
TrueFoundry Will Install
TrueFoundry Will Integrate
Externally Managed
Not Required
UNKNOWN
```

```text
UNKNOWN COMPONENT OWNERSHIP = NO INSTALLATION
```

until ownership and reconciliation are understood.

## Capacity and Headroom

Use read-only evidence such as:

```bash
kubectl get nodes
kubectl describe node <NODE_NAME>
```

Consider CPU, memory, ephemeral storage, Pod capacity, node pools, labels, taints and GPU resources.

Production planning should consider:

```text
Existing Workload Demand
+ Platform Overhead
+ New Workload Demand
+ Failure / Recovery Capacity
= Required Capacity
```

## Network Connectivity

The integration path can include:

```text
Compute Plane
      ↓
Cluster Integration
      ↓
DNS
      ↓
Network / Proxy / Firewall
      ↓
TLS
      ↓
TrueFoundry Control Plane
```

Validate DNS, outbound connectivity, firewall/proxy requirements, TLS trust, certificate validity, NetworkPolicy and relevant cloud network controls.

Do not classify a connectivity problem as a TrueFoundry software defect until the failed transition is identified.

## Identity and Authorization

Keep these boundaries separate:

```text
TrueFoundry Authorization ≠ Kubernetes RBAC ≠ Cloud IAM
```

TrueFoundry access does not prove Kubernetes authorization, and `kubectl` access does not prove sufficient installation permissions.

## Storage

Observe configured storage classes:

```bash
kubectl get storageclass
```

But:

```text
StorageClass Exists ≠ Persistent Storage Works
```

Detailed PVC/PV lifecycle and failure analysis belongs in Part 2.7.

## GPU Prerequisites

Where GPU workloads are required, verify that the target architecture includes appropriate GPU-capable infrastructure. Deep NVIDIA driver, device-plugin, GPU Operator, CUDA, MIG and GPU scheduling coverage belongs in Part 4.

## Reconciliation

Model platform state as:

```text
Platform Intent
      ↓
Authoritative Configuration
      ↓
Cluster Integration
      ↓
Kubernetes Desired State
      ↓
Kubernetes Reconciliation
      ↓
Observed Runtime
```

Before modifying a Kubernetes object, determine who created it, who updates it, who can delete/recreate it, and where its authoritative configuration lives.

```text
REMEDIATE THROUGH THE AUTHORITATIVE SOURCE
```

## Production Change Gate

Before onboarding, confirm:

- target verified
- supported architecture and vendor prerequisites verified
- existing components inventoried
- component ownership identified
- network prerequisites verified
- IAM/RBAC reviewed
- capacity/headroom reviewed
- pre-change baseline captured
- rollback strategy documented
- monitoring available
- change approval/window confirmed

## Conceptual Onboarding Lifecycle

```text
Choose Onboarding Model
        ↓
Create New Cluster OR Attach Existing Cluster
        ↓
Validate Provider Prerequisites
        ↓
Configure Compute Plane
        ↓
Install / Integrate Cluster Components
        ↓
Establish Control Plane Communication
        ↓
Reconcile Required State
        ↓
Validate Installation
        ↓
Validate Integration
        ↓
Validate Workload Capability
```

Use the current vendor-supported procedure for actual installation commands.

## Installation Validation

Safe observations include:

```bash
kubectl get pods -A
kubectl get deployments -A
kubectl describe pod <POD> -n <NAMESPACE>
kubectl logs <POD> -n <NAMESPACE>
```

Do not expose credentials or Secret values.

## Three Production Acceptance Gates

### Gate 1 — Installation

Verify expected resources exist, Pods schedule, and containers start.

### Gate 2 — Integration

Verify cluster integration, Control Plane communication, required permissions and reconciliation.

### Gate 3 — Production Acceptance

Verify a representative workload, networking, DNS/TLS, required storage, GPU capability where required, observability, and the health of existing workloads.

```text
Installation Success
        ≠
Integration Success
        ≠
Production Acceptance
```

## Post-Change Regression Validation

For an existing cluster, validate that onboarding has not negatively affected existing workloads, ingress, DNS, TLS, storage, autoscaling, GitOps, monitoring, GPU scheduling, node capacity or cluster events.

```text
TrueFoundry Connected
        +
Existing Cluster Healthy
        +
Representative Workload Healthy
        =
Production Acceptance
```

## Minimum Supported Blast Radius

Determine the smallest affected scope supported by evidence:

```text
Container → Pod → Deployment → Platform Component → Namespace
→ Node → Node Pool → Compute Plane → Multiple Compute Planes → Control Plane
```

Do not claim a platform-wide outage when evidence supports only a narrower failure.

## Evidence Confidence

Classify investigation statements as:

```text
PROVEN
SUPPORTED
UNKNOWN
ASSUMED
```

Never turn an assumption into RCA language without evidence.

## Lowest Proven Healthy Layer and First Failed Transition

Example:

```text
Pod created                 ✅
Pod scheduled               ✅
Container started           ✅
DNS resolution              ❌
TLS                         UNKNOWN
Control Plane connectivity  UNKNOWN
```

Therefore:

```text
Lowest Proven Healthy Layer:
Container Runtime

First Failed Transition:
Container → DNS

Failure Domain:
DNS / Networking
```

This is stronger than simply reporting that `tfy-agent` cannot connect.

## Common Failure Domains

1. Deployment Identity
2. TrueFoundry Control Plane
3. Cluster Integration
4. Kubernetes API / Reconciliation
5. Kubernetes RBAC
6. Cloud IAM
7. Scheduling / Capacity
8. Node / Container Runtime
9. DNS
10. Network / Firewall / Proxy
11. TLS / Certificate Trust
12. Container Registry
13. Storage
14. Configuration
15. GPU Infrastructure
16. External Dependency
17. Unknown

## Failure Examples

### Pod Pending

Use:

```bash
kubectl describe pod <POD> -n <NAMESPACE>
```

If the evidence shows `FailedScheduling`, investigate CPU, memory, taints/tolerations, selectors, affinity, Pod capacity and required devices.

### Running but Cannot Connect

A Running Pod does not prove integration health:

```text
Process → DNS → Network → Proxy → TLS → Control Plane
```

### Forbidden

Determine which authorization boundary rejected the operation. For Kubernetes API access, investigate Kubernetes RBAC and the authoritative RBAC configuration rather than automatically broadening permissions.

### ImagePullBackOff

Investigate image identity, registry availability/authentication, DNS/network, image existence and node architecture. `ImagePullBackOff` is an observed condition, not a complete root cause.

### TLS Failure

Investigate certificate chain, trust store, hostname, DNS, proxy behavior, validity and system time.

## Timestamped Evidence

Record:

```text
Timestamp
Environment
Cluster
Namespace
Object
Observed State
Reason
Relevant Message
```

Correlate change, deployment, Pod, node, platform, user-impact and recovery timestamps. Time correlation supports investigation but does not prove causation.

## Sensitive Data Handling

```text
DO NOT DECODE OR DISPLAY SECRETS
```

Prefer object names, metadata, references, status, events and safe configuration identifiers. Do not expose tokens, passwords, private keys, cloud credentials, registry credentials or Secret values.

## Current Actionable Owner

Use:

```text
Evidence
   ↓
Failure Domain
   ↓
Next Required Action
   ↓
Current Actionable Owner
```

The architecture identifies the failure boundary; the organization's support model identifies the owner.

## Escalation Evidence Contract

Before escalation, provide:

- timestamp and environment
- Compute Plane and cluster identity
- affected component and symptom
- Minimum Supported Blast Radius
- relevant Kubernetes state/events
- sanitized logs
- Lowest Proven Healthy Layer
- First Failed Transition
- failure-domain hypothesis
- actions already attempted
- explicit Requested Action

## Rollback Planning

Before onboarding, determine what will be created or modified, what controllers will reconcile state, how integration can be disabled safely, what state must survive, how existing workloads will be protected, and how rollback success will be validated.

```text
Uninstalling New Components ≠ Restoring Previous State
```

Rollback means restoring the expected operational state.

## Production Onboarding Record

Maintain a record containing:

```text
Environment:
Cloud:
Account / Subscription / Project:
Region:
TrueFoundry Control Plane:
Compute Plane:
Kubernetes Cluster:
Kubernetes Version:
Workspace:
Namespace Mapping:
Platform Namespace(s):
Cluster Integration Components:
CPU Node Pools:
GPU Node Pools:
Ingress / Gateway:
DNS:
TLS:
StorageClasses:
Registry:
Kubernetes Identity Model:
Cloud IAM Model:
Authoritative Configuration:
Reconciliation Owner:
Observability:
Support Owner:
Escalation Path:
Baseline Captured:
Rollback Plan:
Validation Date:
Validated By:
```

Never store credentials in this record.

## Production Onboarding Flow

```text
DEFINE EXPECTED TARGET
        ↓
VERIFY OBSERVED TARGET
        ↓
SELECT ONBOARDING MODEL
        ↓
INVENTORY EXISTING COMPONENTS
        ↓
IDENTIFY AUTHORITATIVE OWNERS
        ↓
VALIDATE VENDOR PREREQUISITES
        ↓
VALIDATE CAPACITY / HEADROOM
        ↓
CAPTURE PRE-CHANGE BASELINE
        ↓
REVIEW CHANGE + ROLLBACK
        ↓
ONBOARD COMPUTE PLANE
        ↓
VALIDATE INSTALLATION
        ↓
VALIDATE INTEGRATION
        ↓
VALIDATE REPRESENTATIVE WORKLOAD
        ↓
VALIDATE EXISTING WORKLOADS
        ↓
CAPTURE POST-CHANGE STATE
        ↓
PRODUCTION ACCEPTANCE
```

## Incident Investigation Flow

```text
SYMPTOM
   ↓
VERIFY DEPLOYMENT IDENTITY
   ↓
MINIMUM SUPPORTED BLAST RADIUS
   ↓
COLLECT TIMESTAMPED EVIDENCE
   ↓
COMPARE AUTHORITATIVE / DESIRED / OBSERVED STATE
   ↓
LOWEST PROVEN HEALTHY LAYER
   ↓
FIRST FAILED TRANSITION
   ↓
FAILURE DOMAIN
   ↓
NEXT REQUIRED ACTION
   ↓
CURRENT ACTIONABLE OWNER
   ↓
AUTHORITATIVE REMEDIATION
   ↓
END-TO-END VALIDATION
```

## Production Rules

1. Verify the target before mutation.
2. Determine whether the cluster is new or existing.
3. Use the provider-specific supported onboarding procedure.
4. Inventory existing controllers before installing new ones.
5. Understand who owns authoritative configuration.
6. Verify Workspace-to-namespace placement instead of inferring it from naming.
7. Separate TrueFoundry authorization, Kubernetes RBAC and cloud IAM.
8. Capture a pre-change baseline.
9. Validate capacity headroom.
10. Validate DNS, network, proxy and TLS prerequisites.
11. Treat installation and integration as separate states.
12. Require production acceptance beyond `Connected`.
13. Observe before mutate.
14. Never expose Secret values during troubleshooting.
15. Use timestamped evidence.
16. Determine the Minimum Supported Blast Radius.
17. Find the Lowest Proven Healthy Layer.
18. Identify the First Failed Transition.
19. Assign the Current Actionable Owner from the next required action.
20. Remediate through the authoritative configuration source.
21. Validate existing workloads after onboarding.
22. Have a rollback strategy before production changes.
23. Treat Kubernetes conditions as evidence, not automatic root causes.
24. Treat correlation as evidence, not automatic causation.
25. Validate end-to-end recovery before closing the change or incident.

## Key Takeaway

Connecting Kubernetes to TrueFoundry is not simply:

```text
Install Agent → Cluster Connected
```

Production onboarding is:

```text
Correct Target
      +
Supported Architecture
      +
Known Ownership
      +
Validated Prerequisites
      +
Controlled Change
      +
Healthy Integration
      +
Representative Workload
      +
No Regression
      +
Operational Evidence
      =
Production-Ready Compute Plane
```

When something fails:

```text
Symptom
  ↓
Evidence
  ↓
Failure Boundary
  ↓
Next Required Action
  ↓
Current Actionable Owner
```
