# Part 1.2 — TrueFoundry Architecture: Control Plane, Compute Plane & `tfy-agent`

**Status:** Canonical  
**Track:** Part 1 — Foundations & Architecture  
**Audience:** SRE, Platform Engineering, DevOps, ML Infrastructure  
**Prerequisite:** Part 1.1 — What Is TrueFoundry?

## 1. Purpose

TrueFoundry uses a split-plane architecture in which platform-management capabilities are separated from the Kubernetes environments where workloads execute.

For an SRE, this separation is fundamental. A failure visible through the platform does not automatically mean the TrueFoundry control plane caused the failure. The problem may exist in cluster integration, Kubernetes, the workload, the container/GPU runtime, networking, storage, or the underlying infrastructure.

## 2. Learning Objectives

After completing this tutorial, you should be able to:

- Explain the Control Plane and Compute Plane architecture.
- Describe the operational role of `tfy-agent`.
- Distinguish the management path from the runtime request path.
- Trace a deployment from TrueFoundry to Kubernetes.
- Understand reconciliation as an operational consideration.
- Identify major production failure domains.
- Isolate failures using evidence rather than platform assumptions.
- Perform safe first-response Kubernetes checks.

## 3. Architecture Mental Model

```text
              TRUEFOUNDRY CONTROL PLANE
          ┌─────────────────────────────┐
          │ UI / API                    │
          │ Metadata                    │
          │ RBAC                        │
          │ Deployment Configuration    │
          │ Platform Management         │
          └──────────────┬──────────────┘
                         │
                  secure outbound
                    connection
                         ▲
                         │
          ┌──────────────┴──────────────┐
          │        COMPUTE PLANE        │
          │                             │
          │ tfy-agent / integration     │
          │ Platform Controllers/Addons │
          │ Kubernetes                  │
          │                             │
          │ Deployments / Pods          │
          │ Services                    │
          │ ConfigMaps / Secrets        │
          │ CPU Workloads               │
          │ GPU Workloads               │
          │ vLLM / Model Servers        │
          └──────────────┬──────────────┘
                         │
                         ▼
          CLOUD / DATACENTER
          CPU / GPU / Network
          Storage / IAM
```

This is a foundational operational model. Additional platform components may exist depending on the TrueFoundry installation and product capabilities in use.

## 4. Control Plane

The Control Plane provides the platform-management layer.

Its responsibilities include platform-facing capabilities such as:

- UI and API interactions
- metadata
- access-control functions
- deployment configuration
- orchestration and platform management

The important SRE distinction is:

```text
Control Plane
     ≠
Kubernetes worker nodes
     ≠
application runtime
```

The Control Plane coordinates operations against connected compute environments. Customer application containers ultimately execute in the Compute Plane.

## 5. Compute Plane

The Compute Plane is the Kubernetes environment where workloads execute.

It can contain:

- Kubernetes worker nodes
- namespaces
- platform integration components
- Deployments
- Pods
- Services
- ConfigMaps
- Secrets
- persistent storage resources
- application services
- model servers
- CPU workloads
- GPU workloads

Multiple compute environments may be connected to a TrueFoundry Control Plane depending on the platform architecture.

## 6. `tfy-agent`

`tfy-agent` is part of the integration between the TrueFoundry Control Plane and a connected Kubernetes Compute Plane.

A simplified model is:

```text
Control Plane
     ▲
     │ secure outbound connection
     │
tfy-agent
     │
     ▼
Kubernetes
```

The cluster-side integration establishes communication outward toward the Control Plane rather than requiring the Kubernetes API to be broadly exposed inbound for platform management.

For SRE operations, `tfy-agent` should be considered an important dependency in the management and reconciliation path.

It should not automatically be treated as the cause of every TrueFoundry-visible workload failure.

## 7. Management Path vs Runtime Path

This distinction is one of the most important architecture concepts.

### 7.1 Management Path

```text
Engineer / CI/CD
       ↓
TrueFoundry Control Plane
       ↓
Cluster Integration
       ↓
tfy-agent / related components
       ↓
Kubernetes
       ↓
Desired Resources
```

This path is involved in deployment, platform configuration, and platform-driven reconciliation.

### 7.2 Runtime Request Path

Once a service is running, its application request path is different:

```text
Client
   ↓
Ingress / Gateway
   ↓
Kubernetes Service
   ↓
Application / Model Server
   ↓
Model / GPU / Dependencies
```

The TrueFoundry Control Plane is not inherently part of every application request.

Therefore:

```text
Control-plane problem
        ≠
automatic running-service outage
```

Management operations may be affected even when already-running workloads continue serving traffic.

## 8. Deployment Flow

A simplified deployment sequence is:

```text
User / CI/CD
     ↓
TrueFoundry
     ↓
Control Plane
     ↓
Cluster Integration
     ↓
Kubernetes
     ↓
Resources Created / Reconciled
     ↓
Scheduler
     ↓
Worker Node
     ↓
Container
     ↓
Application / Model Server
```

For GPU model serving:

```text
TrueFoundry
    ↓
Kubernetes
    ↓
GPU Pod
    ↓
Container / GPU Runtime
    ↓
NVIDIA Driver
    ↓
CUDA
    ↓
Framework / Runtime
    ↓
vLLM
    ↓
Model
```

Every layer introduces a separate potential failure domain.

## 9. Kubernetes Resources

A TrueFoundry-managed workload may ultimately depend on Kubernetes resources such as:

- Deployments
- Pods
- Services
- ConfigMaps
- Secrets
- ServiceAccounts
- ingress/networking resources
- persistent storage
- resource requests and limits
- node selectors and affinity
- taints and tolerations

The exact resources depend on the workload type and platform configuration.

An SRE should maintain both views:

```text
Platform View
TrueFoundry service / deployment / model

              +

Infrastructure View
Kubernetes objects and underlying resources
```

Production troubleshooting should use evidence from both.

## 10. Reconciliation

Platform integration is not merely passive monitoring.

TrueFoundry platform components can participate in reconciling desired platform state with Kubernetes resources.

This becomes particularly important during:

- cluster migration
- restore
- disaster recovery
- maintenance
- manual Kubernetes intervention

Do not assume:

```text
tfy-agent = monitoring agent only
```

A better operational model is:

```text
Desired Platform State
        ↓
Cluster Integration
        ↓
Reconciliation
        ↓
Kubernetes State
```

Before manually changing or restoring platform-managed resources, understand whether platform reconciliation could recreate, update, or otherwise modify those resources.

## 11. Production Failure-Domain Model

Use this model when investigating incidents:

```text
TrueFoundry Control Plane
          ↓
Cluster Integration
          ↓
tfy-agent / related components
          ↓
Kubernetes API
          ↓
Kubernetes Scheduler
          ↓
Worker Node
          ↓
Container
          ↓
Application / Model Server
          ↓
Runtime / CUDA
          ↓
GPU / Infrastructure
```

Do not collapse these layers into one failure domain.

For example:

```text
TrueFoundry deployment failed
```

does not establish:

```text
TrueFoundry Control Plane failed
```

Evidence must identify where execution stopped.

## 12. Dependency Isolation

A useful production question is:

> What is the lowest layer that we can prove is healthy?

Example:

```text
Deployment submitted          YES
        ↓
Kubernetes object created     YES
        ↓
Pod scheduled                 YES
        ↓
Container started             YES
        ↓
vLLM started                  NO
```

The evidence points away from initial Control Plane, agent, and scheduler investigation and toward the application/model-runtime layer.

Another example:

```text
Deployment submitted          YES
        ↓
Kubernetes object created     NO
```

The management/integration path now deserves investigation.

## 13. `tfy-agent` First-Response Investigation

Start with discovery:

```bash
kubectl get pods -A | grep -Ei 'truefoundry|tfy'
```

Locate the relevant component and namespace.

Then inspect the pod:

```bash
kubectl describe pod <tfy-agent-pod> -n <namespace>
```

Review Kubernetes events:

```bash
kubectl get events -A --sort-by='.lastTimestamp'
```

Where operational policy permits log access:

```bash
kubectl logs <tfy-agent-pod> -n <namespace> --tail=100
```

Look for evidence such as:

- container restarts
- image failures
- resource pressure
- authentication or authorization failures
- connectivity problems
- errors reported by the integration component

Do not restart or modify the component as the first troubleshooting action.

## 14. Workload Troubleshooting

If Kubernetes resources were successfully created, move down the dependency chain.

```bash
kubectl get pods -n <namespace> -o wide
```

For a `Pending` pod:

```bash
kubectl describe pod <pod> -n <namespace>
```

Investigate:

- scheduler events
- CPU and memory capacity
- GPU availability
- taints and tolerations
- affinity
- storage constraints

For `ImagePullBackOff`, investigate image and registry configuration.

For `CrashLoopBackOff`, inspect the application/runtime layer:

```bash
kubectl logs <pod> -n <namespace>
```

## 15. GPU Failure Isolation

For GPU workloads:

```bash
kubectl get nodes -o custom-columns='NODE:.metadata.name,GPU:.status.allocatable.nvidia\.com/gpu'
```

Check GPU-related components:

```bash
kubectl get pods -A | grep -Ei 'nvidia|gpu'
```

Follow the dependency chain:

```text
Pod requested GPU?
       ↓
GPU node available?
       ↓
Pod scheduled to GPU node?
       ↓
NVIDIA components healthy?
       ↓
GPU exposed to container?
       ↓
CUDA compatible?
       ↓
Framework/runtime compatible?
       ↓
vLLM healthy?
       ↓
Model loads?
```

This prevents GPU/runtime failures from being incorrectly attributed to the TrueFoundry Control Plane.

## 16. Production First-Response Flow

Use this sequence:

```text
Observe symptom
      ↓
Determine management vs runtime problem
      ↓
Confirm Kubernetes object existence
      ↓
Check workload state
      ↓
Check Kubernetes events
      ↓
Check scheduling
      ↓
Check container/application
      ↓
Check network/storage
      ↓
Check GPU/runtime if applicable
      ↓
Investigate TrueFoundry integration
when evidence points there
```

## 17. Safe Discovery Commands

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get pods -A | grep -Ei 'truefoundry|tfy'
kubectl get deployments -A | grep -Ei 'truefoundry|tfy'
kubectl get svc -A | grep -Ei 'truefoundry|tfy'
kubectl get events -A --sort-by='.lastTimestamp'
```

These commands form the basis of the Part 1.2 `[SAFE-READ]` lab.

## 18. Production Rules

1. Separate the Control Plane from the Compute Plane.
2. Separate management traffic from runtime traffic.
3. Treat `tfy-agent` as part of the management/reconciliation path.
4. Do not assume a platform-visible error originated in the platform.
5. Determine whether expected Kubernetes resources were created.
6. Troubleshoot downward through dependency layers.
7. Understand reconciliation before manually modifying platform-managed resources.
8. Use evidence before assigning ownership.
9. Start production investigation with read-only discovery whenever possible.

## 19. Architecture Summary

```text
                 MANAGEMENT
                     │
                     ▼
            TrueFoundry Control Plane
                     │
             secure integration
                     ▲
                     │
                 tfy-agent
                     │
                     ▼
             Kubernetes Compute Plane
              │              │
              │              └── GPU workloads
              │                     │
              │                 NVIDIA/CUDA
              │                     │
              └── Applications      └── vLLM
                     │
                     ▼
              RUNTIME TRAFFIC
```

The SRE objective is not simply to know the components.

It is to understand **which path is failing, where execution stopped, and which layer owns the failure**.

## 20. Next Tutorial

Continue with:

**Part 1.3 — TrueFoundry Platform Components & Terminology**
