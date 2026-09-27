# Part 1.4 — TrueFoundry and Kubernetes: How They Work Together

**Status:** Canonical
**Track:** Part 1 — Foundations & Architecture
**Audience:** SRE, Platform Engineering, DevOps, ML Infrastructure
**Prerequisites:** Parts 1.1–1.3

## 1. Purpose

TrueFoundry provides a platform abstraction for deploying and operating applications, AI workloads, and model-serving workloads, while Kubernetes provides workload orchestration within the Compute Plane.

```text
TrueFoundry
     ↓
Platform intent and lifecycle management
     ↓
Cluster integration
     ↓
Kubernetes
     ↓
Workload orchestration
     ↓
Runtime
     ↓
Infrastructure
```

TrueFoundry does not replace Kubernetes. For an SRE, the key skill is translating a TrueFoundry-visible symptom into Kubernetes, runtime, and infrastructure evidence.

## 2. Learning Objectives

After completing this tutorial, you should be able to:
- explain how TrueFoundry and Kubernetes work together
- distinguish the TrueFoundry Control Plane from the Kubernetes Control Plane
- explain the role of the Compute Plane and `tfy-agent`
- correlate platform workloads with Kubernetes resources
- distinguish authoritative, desired, and observed state
- understand reconciliation and GitOps concepts
- distinguish management-path failures from runtime failures
- identify the lowest proven healthy layer and first failed transition
- identify a likely failure domain before remediation

## 3. TrueFoundry and Kubernetes

```text
Engineer / CI/CD
       ↓
TrueFoundry Control Plane
       ↓
Cluster Integration
       ↓
Compute Plane
       ↓
Kubernetes API
       ↓
Kubernetes Resources
       ↓
Scheduler / Controllers
       ↓
Worker Node
       ↓
Pod / Container
       ↓
Application / Model
```

TrueFoundry provides the platform-facing management experience. Kubernetes provides workload orchestration inside the Compute Plane.

## 4. Two Different Control Planes

```text
TrueFoundry Control Plane
        ↓
Platform Management
        ↓
Cluster Integration
        ↓
Kubernetes API
        ↓
Kubernetes Control Plane
        ↓
Scheduler / Controllers
        ↓
Worker Nodes
```

The TrueFoundry Control Plane and Kubernetes Control Plane are not the same. During incidents, identify the affected layer explicitly.

## 5. Compute Plane

```text
Compute Plane
│
├── Kubernetes
├── tfy-agent
├── Platform controllers
├── GitOps components
├── Networking components
├── Observability components
├── Autoscaling components
├── GPU infrastructure
└── Workloads
    ├── Applications
    ├── Jobs
    ├── Model servers
    └── AI workloads
```

Exact components depend on installation, platform version, and enabled capabilities.

## 6. Cluster Integration and tfy-agent

```text
TrueFoundry Control Plane
          ▲
          │ Secure connection initiated
          │ from the Compute Plane
          │
      tfy-agent
          │
          ▼
Platform Components
          │
          ▼
Kubernetes
```

`tfy-agent` is one part of broader cluster integration. Exact transport and implementation can vary by platform version and deployment model.

## 7. From Platform Intent to Kubernetes

```text
Workload Configuration
        ↓
TrueFoundry
        ↓
Cluster Integration
        ↓
Platform Delivery / GitOps
        ↓
Kubernetes Resources
        ↓
Scheduler
        ↓
Node
        ↓
Pod
        ↓
Container
        ↓
Application / Model
```

Platform view:

```text
Workspace → Workload → Deployment / Version
```

Kubernetes view:

```text
Namespace → Workload Resource → Pod → Container → Node
```

An SRE must be able to correlate the two.

## 8. Kubernetes Resource Graph

A TrueFoundry workload can result in or depend on multiple Kubernetes resources, including workload controllers, Pods, Services, ConfigMaps, Secret references, ServiceAccounts, ingress/gateway configuration, PVCs, and other supporting resources.

Do not assume one TrueFoundry workload equals one Kubernetes resource.

## 9. Kubernetes Scheduling

```text
Pod
 ↓
Scheduler
 ↓
CPU / Memory / GPU
Node selectors / Affinity
Taints / Tolerations
Storage dependencies
 ↓
Eligible Node
```

TrueFoundry can express workload requirements, while Kubernetes performs underlying scheduling.

## 10. GPU Failure Isolation

```text
Stage 1 — Scheduling
Stage 2 — Device Access
Stage 3 — CUDA / Application Runtime
```

A vLLM device-detection error does not automatically prove a Kubernetes scheduling problem.

## 11. Authoritative, Desired, and Observed State

```text
AUTHORITATIVE STATE
TrueFoundry / Git / Platform Configuration
                 ↓
DESIRED KUBERNETES STATE
What controllers expect
                 ↓
OBSERVED KUBERNETES STATE
What actually exists
```

The first production question should be: **Where is the authoritative configuration?**

## 12. Reconciliation and GitOps

```text
Authoritative Configuration
          ↓
Desired State
          ↓
Controller / GitOps
          ↓
Kubernetes
          ↓
Observed State
          ↓
Compare / Reconcile
```

Where GitOps is configured, Git and a controller such as Argo CD may participate in the delivery path. Kubernetes may therefore not be the authoritative configuration source.

## 13. Why Direct Kubernetes Changes Can Be Dangerous

A direct `kubectl edit`, `patch`, or `scale` can create drift when the authoritative desired configuration remains unchanged. Reconciliation may later replace the manual change.

Determine configuration ownership before modifying platform-managed resources.

## 14. Observe Before Mutate

```text
OBSERVE
   ↓
CORRELATE
   ↓
PROVE
   ↓
IDENTIFY FAILURE DOMAIN
   ↓
IDENTIFY AUTHORITATIVE SOURCE
   ↓
REMEDIATE
   ↓
VALIDATE
```

Immediate restarts can destroy useful evidence and temporarily hide the actual failure.

## 15. Workspace vs Namespace

Do not assume a TrueFoundry Workspace equals a Kubernetes namespace.

```text
Workspace
   ↓
Workload
   ↓
Compute Plane
   ↓
Kubernetes Namespace
   ↓
Kubernetes Resources
```

Discover the actual relationship from the environment.

## 16. Configuration, Secrets, and Identity

Runtime configuration can involve environment variables, ConfigMaps, Secret references, command arguments, ServiceAccounts, workload identity, and cloud IAM.

Validate Secret metadata and references without routinely displaying or decoding Secret values.

A workload can be Running and Ready while authorization to an external dependency still fails.

## 17. TrueFoundry Service vs Kubernetes Service

```text
TrueFoundry Service
        ≠
Kubernetes Service
```

A TrueFoundry Service is a platform workload abstraction. A Kubernetes Service is a networking abstraction.

## 18. Runtime Network and Storage Paths

```text
Client
  ↓
DNS
  ↓
Load Balancer / Gateway
  ↓
Ingress
  ↓
Kubernetes Service
  ↓
Pod
  ↓
Application / Model
```

Persistent workloads can also depend on:

```text
Application → Pod → PVC → Persistent Volume → Storage Infrastructure
```

Each layer can represent a distinct failure domain.

## 19. Management Path vs Runtime Path

Management:

```text
Engineer / CI/CD
       ↓
TrueFoundry Control Plane
       ↓
Cluster Integration
       ↓
GitOps / Platform Controllers
       ↓
Kubernetes API
```

Runtime:

```text
Client
   ↓
DNS / Gateway
   ↓
Kubernetes Service
   ↓
Pod
   ↓
Application / Model
   ↓
Dependencies
```

Management-path failure does not automatically mean runtime outage, and runtime outage does not automatically mean Control Plane failure.

## 20. Kubernetes API as a Dependency

If the Kubernetes API is unavailable, new deployment or reconciliation operations may fail while already-running Pods may continue executing.

## 21. Kubernetes State Is Evidence

`CrashLoopBackOff`, `Pending`, `ImagePullBackOff`, `OOMKilled`, and `FailedScheduling` are useful observed states, not necessarily root causes.

An RCA should explain why the workload reached the observed state.

## 22. Evidence Hierarchy

```text
Platform symptom
      ↓
Workload identity
      ↓
Compute Plane / cluster
      ↓
Namespace
      ↓
Workload controller
      ↓
Pod state
      ↓
Events
      ↓
Container state
      ↓
Application logs
      ↓
Node / runtime
      ↓
Network / storage / GPU / IAM
```

## 23. Lowest Proven Healthy Layer

Ask: **What is the lowest layer we can prove is healthy?**

```text
TrueFoundry request accepted       ✅
Kubernetes object created          ✅
Pod scheduled                      ✅
Container started                  ✅
Application initialization        ❌
```

Focus immediately below the lowest proven healthy layer.

## 24. First Failed Transition

```text
Platform Request
      ↓
Kubernetes Object
      ↓
Scheduled Pod
      ↓
Started Container
      ↓
Ready Application
      ↓
Reachable Endpoint
```

Find the first transition that fails and investigate that boundary.

## 25. Node, Runtime, and Dependencies

After scheduling, the investigation can move through:

```text
Pod → Node → kubelet → Container Runtime → Container
```

A Ready Pod still does not prove end-to-end transaction health. Databases, Redis, object storage, external APIs, model storage, network paths, and IAM can remain failure domains.

## 26. Production Examples

### CrashLoopBackOff

If the Kubernetes object exists, the Pod schedules, the container starts, and the application exits, investigate application/runtime configuration and dependencies rather than treating `CrashLoopBackOff` as the root cause.

### FailedScheduling

If a Pod is Pending with `FailedScheduling`, investigate CPU, memory, GPU, node selectors, affinity, taints/tolerations, and storage dependencies before application startup.

### ImagePullBackOff

Investigate image reference, registry reachability, authentication, and image availability.

### Healthy Pod, Broken Endpoint

If Pod and application health are established but external requests fail, investigate DNS, load balancer/gateway, ingress, TLS, Kubernetes Service, and routing.

### GPU Runtime Failure

If the Pod schedules to a GPU node, receives the GPU, and starts the container but CUDA initialization fails, focus on GPU runtime/model-server compatibility rather than scheduling.

## 27. Production Incident Snapshot

Before remediation capture context, cluster, workspace, workload, namespace, Kubernetes resource, desired/ready replicas, Pod state, node, restart count, recent events, container state, relevant logs, network state, and GPU requirements/capacity.

Never include credentials, tokens, Secret values, or sensitive payloads.

## 28. Production Troubleshooting Workflow

```text
TrueFoundry Symptom
        ↓
IDENTIFY workload / workspace / cluster
        ↓
OBSERVE Kubernetes state
        ↓
CORRELATE platform object ↔ Kubernetes object
        ↓
PROVE lowest healthy layer
        ↓
LOCATE first failed transition
        ↓
CLASSIFY failure domain
        ↓
IDENTIFY authoritative configuration
        ↓
REMEDIATE through correct owner/path
        ↓
VALIDATE desired + observed + end-to-end health
```

## 29. Safe First-Response Commands

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get namespaces
kubectl get deployments -A
kubectl get pods -A -o wide
kubectl get svc -A
kubectl get ingress -A
kubectl get events -A --sort-by='.lastTimestamp'
kubectl describe deployment <deployment> -n <namespace>
kubectl describe pod <pod> -n <namespace>
kubectl logs <pod> -n <namespace> --tail=100
```

## 30. Production Rules

1. TrueFoundry does not replace Kubernetes.
2. Distinguish the TrueFoundry Control Plane from the Kubernetes Control Plane.
3. Identify workload and Compute Plane before troubleshooting.
4. Do not assume Workspace equals namespace.
5. Do not assume one platform workload equals one Kubernetes resource.
6. Determine authoritative configuration before changing Kubernetes.
7. Treat direct Kubernetes modifications as potentially temporary drift.
8. Observe before mutating.
9. Treat Kubernetes states as evidence rather than automatic root causes.
10. Find the lowest proven healthy layer.
11. Find the first failed transition.
12. Separate management-path health from runtime-path health.
13. Separate GPU scheduling, device access, and runtime failures.
14. Protect Secret values.
15. Preserve evidence before restart or redeployment.
16. Validate end-to-end application health after remediation.

## 31. Production Takeaway

```text
TrueFoundry
     ↓
Platform Intent
     ↓
Authoritative Configuration
     ↓
Cluster Integration / GitOps
     ↓
Kubernetes
     ↓
Observed Workload State
     ↓
Runtime
     ↓
Infrastructure
```

```text
Platform Symptom
       ↓
Kubernetes Evidence
       ↓
Lowest Proven Healthy Layer
       ↓
First Failed Transition
       ↓
Failure Domain
       ↓
Authoritative Remediation
       ↓
End-to-End Validation
```

## Next Tutorial

**Part 1.5 — Workspaces, Environments & Deployment Concepts**
