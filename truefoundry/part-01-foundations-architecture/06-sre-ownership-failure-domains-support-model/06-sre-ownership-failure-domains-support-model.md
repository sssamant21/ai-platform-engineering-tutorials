# Part 1.6 — SRE Ownership, Failure Domains & Support Model

**Track:** Part 1 — Foundations & Architecture  
**Status:** Canonical  
**Audience:** SRE, Platform Engineering, DevOps, ML Infrastructure  
**Prerequisites:** Parts 1.1–1.5

---

## Purpose

Production incidents involving TrueFoundry often surface through the platform even when the underlying failure exists elsewhere. Production troubleshooting therefore requires four separate questions:

```text
WHAT is affected?
WHERE is the first proven failure?
WHAT action is required next?
WHO owns that action?
```

The central principle is:

```text
SYMPTOM
   ≠
FAILURE DOMAIN
   ≠
OWNER
```

Evidence connects them.

## Learning Objectives

After completing this tutorial, you should be able to:

- identify deployment identity before troubleshooting
- determine the minimum supported blast radius
- distinguish management, deployment, and runtime impact
- separate symptom location from failure location
- identify the Lowest Proven Healthy Layer
- identify the First Failed Transition
- classify production failure domains
- distinguish platform state from Kubernetes/runtime state
- identify the next required action and Current Actionable Owner
- build evidence-based escalation packages
- distinguish mitigation, remediation, recovery, and closure
- validate recovery end to end

## TrueFoundry Production Architecture Boundary

```text
              TRUEFOUNDRY
       Management / Control Plane
                  │
          Cluster Integration
       tfy-agent / related components
                  │
                  ▼
       Compute Plane / Kubernetes
                  │
        ┌─────────┼─────────┐
        ▼         ▼         ▼
     Nodes     Runtime    Workloads
        └─────────┼─────────┘
                  ▼
          User Transactions
```

The Compute Plane is the customer Kubernetes execution environment. `tfy-agent` and related cluster-side components participate in integration with the TrueFoundry Control Plane.

Do not equate `tfy-agent` with the Compute Plane, or the TrueFoundry Control Plane with the Kubernetes Control Plane.

## Management Path vs Runtime Path

```text
MANAGEMENT PATH
Operator → TrueFoundry Control Plane → Cluster Integration → Kubernetes

RUNTIME PATH
Client → Endpoint → Application / Model → Dependencies
```

A management-path problem does not automatically prove a runtime outage.

## Three Impact Classes

Classify independently:

- **Management Impact** — can operators use platform management capabilities?
- **Deployment Impact** — can workloads be created, updated, reconciled, or deployed?
- **Runtime Impact** — are existing workload transactions affected?

```text
Management Impact ≠ Deployment Impact ≠ Runtime Impact
```

## Deployment Identity Comes First

```text
Operational Environment
 + Compute Plane / Cluster
 + Workspace
 + Namespace
 + Workload
 + Artifact
 + Configuration
 + Deployment Version
 = Deployment Identity
```

```text
UNKNOWN TARGET = NO MUTATION
```

A TrueFoundry Workspace and Kubernetes namespace are different concepts. Correlate them from the actual deployment rather than assuming they are equivalent.

## Symptom Location Is Not Failure Location

A TrueFoundry-visible `Pod Pending` condition may correlate with Kubernetes evidence such as:

```text
FailedScheduling: Insufficient nvidia.com/gpu
```

That supports a scheduling/GPU-capacity investigation, not an automatic TrueFoundry root-cause conclusion.

```text
Error visible in TrueFoundry ≠ TrueFoundry root cause
```

## Failure-Domain Model

```text
User Symptom
 ↓ Deployment Identity
 ↓ TrueFoundry Management / Control Plane
 ↓ Cluster Integration
 ↓ Kubernetes / Compute Plane
 ↓ Scheduling / Capacity
 ↓ Node / Container Runtime
 ↓ GPU / Device Runtime
 ↓ Application / Model Runtime
 ↓ Network / Storage / Identity
 ↓ External Dependencies
 ↓ User Transaction
```

This is an investigation model, not a literal request path.

Core failure domains:

1. Deployment Identity
2. TrueFoundry Management / Control Plane
3. TrueFoundry Cluster Integration
4. Kubernetes Control / Reconciliation
5. Scheduling / Capacity
6. Node / Container Runtime
7. GPU / Device / CUDA Runtime
8. Application / Model Runtime
9. Networking
10. Storage
11. Identity / RBAC / IAM
12. Configuration / Secrets
13. External Dependency
14. Unknown

`Unknown` is valid while evidence is incomplete.

## Architecture vs Organizational Ownership

Architecture helps identify **where** the failure boundary exists. The organization's support model determines **who** owns the next action.

```text
Evidence
 ↓ Failure Domain
 ↓ Next Required Action
 ↓ Organization Ownership Map
 ↓ Current Actionable Owner
```

Do not hardcode universal team ownership into the architecture.

## Ownership Terminology

Distinguish:

- **Incident Coordinator** — coordinates the incident.
- **Current Actionable Owner** — owns the next evidence-gathering or remediation action.
- **Configuration / Service Owner** — owns the authoritative configuration or service.
- **Root-Cause Owner** — owns the underlying cause once established.

These may be different.

Investigation responsibility also does not necessarily equal component ownership.

## Ownership Can Change

```text
Application failure reported
 ↓ Application investigation
 ↓ FailedScheduling discovered
 ↓ Platform investigation
 ↓ Incorrect GPU request discovered
 ↓ Configuration owner engaged
```

Ownership should follow evidence. Changing ownership is normal; unsupported ownership bouncing is not.

## Minimum Supported Blast Radius

Report only the smallest impact scope directly supported by evidence:

```text
Container
Pod
Workload
Namespace
Workspace
Compute Plane
Multiple Compute Planes
Platform
```

If only one Pod is proven affected, report that scope until broader evidence exists.

## Lowest Proven Healthy Layer

This tutorial series uses **Lowest Proven Healthy Layer** as an SRE investigation method, not as a TrueFoundry product object.

```text
Control Plane interaction      PASS
Cluster integration            PASS
Kubernetes API                 PASS
Resource creation              PASS
Scheduling                     PASS
Container startup              PASS
Application initialization     FAIL
```

Result:

```text
Lowest Proven Healthy Layer: Container startup
First Failed Transition: Container → Application initialization
```

## Evidence Quality

Absence of observed errors is not proof of health. Prefer positive evidence.

Classify important conclusions as:

```text
PROVEN
SUPPORTED
UNKNOWN
ASSUMED
```

Never communicate `ASSUMED` as `PROVEN`.

Timestamp operational evidence because infrastructure state changes during incidents.

## Domain Investigation Guidance

### Deployment Identity

Check for wrong Compute Plane, Workspace, namespace, workload, artifact/model, configuration, Secret reference, ServiceAccount, or deployment version.

```text
Pod Running + Ready ≠ Correct Deployment
```

### TrueFoundry Management / Control Plane

Investigate Control Plane/API interaction, management interface, platform-side operations, and deployment management. Validate runtime impact independently.

### Cluster Integration

Investigate `tfy-agent`, cluster connectivity, platform integration components, and reconciliation/integration behavior separately from general Kubernetes health.

### Kubernetes Control / Reconciliation

Ask whether desired state was submitted, admitted, known to Kubernetes, and reconciled. Inspect API, admission, controller, and reconciliation evidence.

### Scheduling / Capacity

Typical evidence includes `Pending`, `FailedScheduling`, insufficient CPU/memory/GPU, untolerated taints, affinity mismatch, and volume scheduling constraints. Determine whether the cause is infrastructure capacity or workload configuration.

### Node / Container Runtime

Investigate Node NotReady, kubelet/container-runtime failures, disk pressure, image filesystem pressure, and container creation failures.

### GPU / Device / CUDA Runtime

Use three stages:

```text
1. Scheduling
2. Device Access
3. CUDA / Application Runtime
```

Validate:

```text
Pod scheduled
 ↓ GPU allocated
 ↓ GPU visible
 ↓ CUDA initialized
 ↓ Model server initialized
```

`GPU workload failed` is not a complete root cause.

### Application / Model Runtime

Inspect application exceptions, configuration parsing, model loading, process exits, readiness failures, and runtime incompatibility. `CrashLoopBackOff` is observed Kubernetes behavior, not a complete root cause.

### Networking

Investigate the relevant path:

```text
Client → DNS → Gateway / Load Balancer → Service → Pod → Application
```

```text
Pod Healthy ≠ Endpoint Healthy
Endpoint Healthy ≠ Business Transaction Healthy
```

### Storage

Investigate PVC state, attach/mount failures, filesystem capacity, latency, permissions, and object-storage failures.

### Identity / Authorization

```text
TrueFoundry Authorization ≠ Kubernetes RBAC ≠ Cloud IAM
```

Ask which operation, identity, resource, permission, and policy are involved.

### Configuration / Secrets

```text
Observed configuration reference
 ↓ Expected reference
 ↓ Authoritative source
 ↓ Configuration owner
```

Never expose Secret values in terminal history, Slack, tickets, screenshots, labs, or RCA documents.

### External Dependencies

Dependencies can include databases, Redis, Kafka, object storage, external APIs, identity providers, model registries, and artifact registries.

```text
Application Process Healthy ≠ Business Transaction Healthy
```

## Change Correlation

Record relevant application, platform, cluster, node-pool, GPU-stack, IAM, network, Secret, dependency, and traffic/load changes.

```text
Change occurred before failure ≠ Change proven as root cause
```

Timing is evidence, not proof.

## Hypothesis Tracking

For significant incidents, track:

```text
Hypothesis:
Evidence For:
Evidence Against:
Validation:
Current Actionable Owner:
Status: OPEN / REJECTED / SUPPORTED / PROVEN
```

This reduces repeated investigation of eliminated theories.

## Escalation Prerequisites

Before escalation, establish as much as reasonably possible:

- what failed
- where and when
- impact and minimum supported blast radius
- what is proven healthy
- First Failed Transition
- supporting evidence
- remaining unknowns
- next required action

## Evidence Handoff Contract

```text
Incident ID:
Timestamp:

Environment:
Compute Plane:
Workspace:
Namespace:
Workload:
Artifact / Version:

Impact:
Minimum Supported Blast Radius:

Management Impact:
Deployment Impact:
Runtime Impact:

Lowest Proven Healthy Layer:
First Failed Transition:

Evidence:
Evidence Confidence:

What Has Been Ruled Out:
What Remains Unknown:

Current Failure Domain:
Current Actionable Owner:

Requested Action:
Expected Evidence / Result:
```

Always include a concrete requested action rather than only saying “Please investigate.”

## Prevent Ownership Ping-Pong

```text
Evidence
 ↓ Failure Boundary
 ↓ Next Required Action
 ↓ Action Owner
 ↓ Expected Evidence
```

If a receiving team rejects the proposed failure domain, the handoff should include evidence that moves the investigation boundary.

Formal ownership transfer should follow the organization's incident-management process.

## Vendor Escalation Package

When evidence supports TrueFoundry vendor escalation, capture:

- time window
- Compute Plane
- Workspace/workload
- affected component
- version information where available
- exact sanitized error
- relevant Kubernetes evidence
- relevant cluster-integration evidence
- minimum supported blast radius
- management/deployment/runtime impact
- troubleshooting already performed
- requested vendor action

Never include credentials, tokens, or Secret values.

## Authoritative Remediation

```text
Failure Domain
 ↓ Authoritative Configuration Source
 ↓ Configuration Owner
 ↓ Approved Change
 ↓ Reconciliation / Deployment
 ↓ Validation
```

Avoid permanent remediation through ad hoc runtime changes when an authoritative configuration system owns the state.

## Mitigation vs Remediation

**Mitigation** reduces immediate impact, for example rollback, traffic shift, failover, or temporary capacity.

**Remediation** corrects the underlying problem, for example configuration correction, application fix, infrastructure repair, IAM correction, or dependency repair.

```text
Service Restored ≠ Root Cause Remediated
```

## Recovery vs Closure

Recovery means the affected service has been restored. Closure may additionally require validated recovery, preserved evidence, recorded temporary mitigations, tracked permanent remediation, recorded ownership, and assigned follow-up actions.

## End-to-End Recovery Validation

```text
Authoritative Desired State
 ↓ Kubernetes Reconciled
 ↓ Expected Pods Healthy
 ↓ Application / Model Healthy
 ↓ Endpoint Healthy
 ↓ Dependencies Healthy
 ↓ Errors Normalized
 ↓ Latency Normalized
 ↓ User Transaction Validated
```

A green deployment status alone is not sufficient.

## Monitoring Evidence

Evidence may come from TrueFoundry, Kubernetes, application logs, metrics, APM, cloud monitoring, GPU monitoring, dependency monitoring, synthetic monitoring, and user reports.

Ask:

> **Which layer does this signal actually observe?**

No single monitoring system is universally authoritative for every layer.

## Production Incident Snapshot

```text
Timestamp:

Environment:
Compute Plane:
Workspace:
Namespace:
Workload:

Impact:
Minimum Supported Blast Radius:

Management Impact:
Deployment Impact:
Runtime Impact:

Lowest Proven Healthy Layer:
First Failed Transition:

Current Failure Domain:
Evidence Confidence:

Current Actionable Owner:
Supporting Functions:

Authoritative Configuration Source:
Configuration Owner:

Next Required Action:
Expected Evidence:

Mitigation:
Permanent Remediation:

Recovery Status:
```

## Production Anti-Patterns

Avoid these conclusions without supporting evidence:

```text
Error appears in TrueFoundry = TrueFoundry caused it
CrashLoopBackOff = Kubernetes root cause
GPU error = GPU hardware failure
Recent deployment = Proven root cause
No errors found = Layer proven healthy
Not my component = Not my next action
Service restored = Incident fully remediated
```

## Production Rules

1. Verify deployment identity first.
2. Unknown target means no mutation.
3. Determine the minimum supported blast radius.
4. Separate management, deployment, and runtime impact.
5. Separate symptom location from failure location.
6. Prefer positive evidence of health.
7. Timestamp operational evidence.
8. Mark conclusions as proven, supported, unknown, or assumed.
9. Identify the Lowest Proven Healthy Layer.
10. Identify the First Failed Transition.
11. Classify the failure domain from evidence.
12. Identify the next required action.
13. Assign the Current Actionable Owner using the organization's support model.
14. Allow ownership to change as evidence changes.
15. Separate investigation responsibility from component ownership.
16. Separate Current Actionable Owner from Root-Cause Owner.
17. Preserve evidence during handoffs.
18. Include a requested action with escalations.
19. Do not expose Secret values.
20. Treat Kubernetes states as evidence, not automatic root causes.
21. Treat timing correlation as evidence, not proof.
22. Remediate through the authoritative source.
23. Distinguish mitigation from permanent remediation.
24. Validate runtime and dependencies after recovery.
25. Validate the user transaction before declaring end-to-end recovery.

## Final Production Investigation Model

```text
USER SYMPTOM
 ↓ VERIFY DEPLOYMENT IDENTITY
 ↓ DETERMINE MINIMUM SUPPORTED BLAST RADIUS
 ↓ CLASSIFY IMPACT
   management / deployment / runtime
 ↓ COLLECT TIMESTAMPED EVIDENCE
 ↓ COMPARE authoritative / desired / observed
 ↓ IDENTIFY Lowest Proven Healthy Layer
 ↓ IDENTIFY First Failed Transition
 ↓ CLASSIFY Failure Domain
 ↓ DEFINE Next Required Action
 ↓ ASSIGN Current Actionable Owner
 ↓ MITIGATE / REMEDIATE through authoritative source
 ↓ VALIDATE runtime + dependencies + transaction
 ↓ DOCUMENT evidence + ownership + outcome
```

## Part 1 Final SRE Principles

```text
Healthy Runtime
      ≠
Correct Deployment
      ≠
Healthy End-to-End Service
```

```text
Symptom Location ≠ Failure Location
Failure Domain ≠ Organizational Owner
```

The final operating sequence is:

```text
Evidence
 ↓ Failure Boundary
 ↓ Next Required Action
 ↓ Current Actionable Owner
 ↓ Authoritative Remediation
 ↓ End-to-End Validation
```

---

**Canonical Status:** Approved after Draft → Technical/Vendor Validation → Production/SRE Review → Revised Final.
