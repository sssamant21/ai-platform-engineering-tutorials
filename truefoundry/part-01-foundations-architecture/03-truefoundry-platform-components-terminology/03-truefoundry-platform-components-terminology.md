# Part 1.3 — TrueFoundry Platform Components & Terminology

**Status:** Canonical  
**Track:** Part 1 — Foundations & Architecture  
**Audience:** SRE, Platform Engineering, DevOps, ML Infrastructure  
**Prerequisites:** Parts 1.1–1.2

## 1. Purpose

Part 1.3 establishes the operational vocabulary used throughout the TrueFoundry track.

For an SRE, knowing a product term is not enough. The important skill is translating:

```text
TrueFoundry Object
       ↓
Desired Configuration
       ↓
Kubernetes Resources
       ↓
Runtime
       ↓
Infrastructure
```

This allows platform-visible symptoms to be correlated with observable Kubernetes and infrastructure evidence.

## 2. Learning Objectives

After completing this tutorial, you should be able to:

- explain common TrueFoundry components and workload terminology
- distinguish platform objects from Kubernetes objects
- identify Workspaces, Services, Jobs, and other workload types
- understand desired state versus observed state
- correlate platform workloads with Kubernetes resources
- recognize lifecycle differences between workload types
- identify the appropriate failure domain during an incident

## 3. Platform Hierarchy

Use this high-level model:

```text
TrueFoundry Platform
│
├── Control Plane
│
├── Compute Plane
│   ├── Cluster Integration
│   ├── tfy-agent
│   ├── Platform Components/Add-ons
│   └── Kubernetes
│
├── Workspace
│
└── Workloads
    ├── Service
    ├── Job
    ├── LLM / Model Serving
    ├── Agent
    ├── MCP Server
    └── Workflow
```

Exact capabilities and supporting components depend on platform version, installation model, and enabled functionality.

## 4. Platform Object vs Infrastructure Object

TrueFoundry terminology and Kubernetes terminology are related but not interchangeable.

```text
PLATFORM VIEW

TrueFoundry Workload
        ↓
Desired Configuration


INFRASTRUCTURE VIEW

Kubernetes Resources
        ↓
Pod / Container
```

For example:

```text
"The TrueFoundry service is unhealthy."
```

describes a platform-level symptom.

```text
"Pod is CrashLoopBackOff."
```

describes observed Kubernetes state.

These statements describe different layers of the same workload.

## 5. Workspace

A Workspace provides a TrueFoundry context into which workloads are deployed and through which related platform configuration and access can be organized.

Conceptually:

```text
TrueFoundry
    │
    ├── Workspace A
    │      ├── Service
    │      ├── Job
    │      └── Model workload
    │
    └── Workspace B
           ├── Service
           └── Job
```

Organizations may use Workspaces to help separate teams, environments, or workload groups, but an SRE should not automatically assume:

```text
Workspace = Environment
```

without understanding the organization's platform design.

## 6. Workloads

A workload represents something the platform deploys or executes.

Current TrueFoundry capabilities can support several workload patterns:

```text
Workloads
│
├── Service
├── Job
├── LLM / Model Serving
├── Agent
├── MCP Server
└── Workflow
```

Later tutorials will examine these workload types in greater depth.

For Part 1, the important concept is that different workload types have different execution and health models.

## 7. Service

A TrueFoundry Service represents a continuously running workload.

Examples can include an API, web application, backend service, inference API, or model server.

Conceptually:

```text
TrueFoundry Service
        ↓
Workload Configuration
        ↓
Kubernetes Resources
        ↓
Pods
        ↓
Network Access
```

A critical distinction is:

```text
TrueFoundry Service
        ≠
Kubernetes Service
```

A TrueFoundry Service is a platform workload abstraction.

A Kubernetes `Service` is a Kubernetes networking abstraction.

A TrueFoundry Service may ultimately rely on several Kubernetes resources:

```text
TrueFoundry Service
        │
        ├── Workload configuration
        ├── Deployment configuration
        └── Runtime/network configuration
                     ↓
              Kubernetes objects
                     │
             ┌───────┴───────┐
             ▼               ▼
        Deployment         Service
             │
             ▼
            Pods
```

Therefore, the statement:

```text
"The service is down."
```

is operationally ambiguous.

## 8. Job

A Job represents task-oriented execution rather than a continuously running service.

Examples include batch processing, training workloads, scheduled AI tasks, and batch inference.

Conceptually:

```text
TrueFoundry Job
      ↓
Execution
      ↓
Kubernetes Workload
      ↓
Pod(s)
      ↓
Completed / Failed
```

The lifecycle differs significantly from a Service.

```text
SERVICE

Deploy
  ↓
Running
  ↓
Ready
  ↓
Serving
  ↓
Continuously Healthy
```

Compared with:

```text
JOB

Trigger
  ↓
Pending
  ↓
Running
  ↓
Completed
```

For a Job:

```text
Completed = potentially successful
```

For a continuously running Service:

```text
Unexpected process exit = normally unhealthy
```

The workload type must therefore be identified before interpreting health.

## 9. Model and LLM Serving

Model-serving workloads introduce additional runtime dependencies.

```text
Client
   ↓
Network Endpoint
   ↓
Model Server
   ↓
Framework / Runtime
   ↓
Model
   ↓
CPU / GPU
```

For GPU inference:

```text
Model Workload
      ↓
GPU Resource Requirement
      ↓
Kubernetes Scheduler
      ↓
GPU Node
      ↓
NVIDIA Runtime
      ↓
CUDA
      ↓
Model Server / vLLM
      ↓
Model
```

A model deployment failure can therefore originate from many layers.

```text
Model deployment failed
        ≠
TrueFoundry Control Plane failed
```

## 10. Agents, MCP Servers, and Workflows

Modern AI applications can include workload patterns beyond conventional applications and model servers.

TrueFoundry's broader workload model can include agents, MCP servers, and workflows.

For this foundational tutorial, treat them as workload abstractions that ultimately depend on runtime infrastructure.

```text
Platform Workload
       ↓
Configuration
       ↓
Compute Plane
       ↓
Kubernetes
       ↓
Runtime
```

Their detailed architecture will be covered later rather than overloaded into Part 1.

## 11. Deployment

Deployment is the process of moving desired workload configuration toward a running workload.

```text
Desired Configuration
       ↓
TrueFoundry
       ↓
Cluster Integration
       ↓
Kubernetes Resources
       ↓
Scheduler
       ↓
Pod
       ↓
Runtime
```

Remember:

```text
Deployment operation
        ≠
Running application
```

A deployment operation can fail without proving that an already-running application is unavailable.

## 12. Desired State vs Observed State

This is one of the most important SRE concepts.

```text
DESIRED STATE

TrueFoundry Configuration
        ↓
What should exist
```

Compare it with:

```text
OBSERVED STATE

Kubernetes
        ↓
What actually exists
```

Troubleshooting often means comparing the two:

```text
Desired State
      │
      │ compare
      ▼
Observed State
      │
      ├── Matches → Healthy candidate
      │
      └── Differs → Investigate
```

A difference may represent deployment failure, reconciliation delay, configuration drift, or another infrastructure problem.

## 13. Reconciliation

As established in Part 1.2:

```text
Desired Platform State
        ↓
Cluster Integration
        ↓
Reconciliation
        ↓
Kubernetes State
```

This means manually modifying Kubernetes resources can conflict with the authoritative desired state.

Before changing platform-managed resources, determine where configuration ownership resides.

## 14. Build vs Runtime

Workload delivery can involve:

```text
Source / Artifact
       ↓
Build
       ↓
Container Image
       ↓
Registry
       ↓
Deployment
       ↓
Container Runtime
       ↓
Application / Model
```

These are separate failure domains.

```text
Build failure
     ≠
Image-pull failure
     ≠
Scheduling failure
     ≠
Container startup failure
     ≠
Application failure
```

## 15. Compute Resources

A workload ultimately executes against Kubernetes and infrastructure capacity.

For CPU workloads:

```text
Workload
   ↓
CPU / Memory Requirement
   ↓
Kubernetes Scheduler
   ↓
Compatible Node
```

For GPU workloads:

```text
Workload
   ↓
GPU Requirement
   ↓
Kubernetes Scheduler
   ↓
GPU-capable Node
```

TrueFoundry expresses workload intent, while Kubernetes and the underlying infrastructure determine whether suitable resources are available.

## 16. Environment Configuration

Runtime configuration may be supplied through environment variables and other configuration mechanisms.

```text
Platform Configuration
        ↓
Runtime Configuration
        ↓
Container
        ↓
Application
```

A pod can therefore be `Running` while the application remains functionally unhealthy because its configuration is incorrect.

## 17. Secrets

Secrets require strict operational handling.

During troubleshooting, verify:

```text
Secret object/reference exists
        ↓
Expected reference configured
        ↓
Workload can use required reference
```

Do not retrieve Secret YAML or decode credentials for routine lab evidence.

Do not paste tokens into Slack, attach credentials to incident tickets, or store Secret values in troubleshooting documentation.

The objective is to validate the **reference and availability**, not expose the secret.

## 18. Network Access Path

A deployed workload may be reached through several networking layers.

```text
Client
   ↓
DNS
   ↓
Ingress / Gateway
   ↓
Kubernetes Service
   ↓
Pod
   ↓
Application
```

Therefore:

```text
Endpoint unavailable
       ≠
Pod unhealthy
```

A healthy application can still be unreachable because of DNS, TLS, ingress, routing, Kubernetes Service, or network-policy problems.

## 19. Compute-Plane Components

Do not reduce the Compute Plane to:

```text
TrueFoundry = tfy-agent
```

A better operational model is:

```text
Compute Plane
│
├── tfy-agent / cluster integration
├── platform controllers
├── observability components
├── autoscaling components
├── networking components
├── GPU components
├── Kubernetes
└── workload resources
```

Exact components vary by installation and enabled capabilities.

## 20. Platform-to-Kubernetes Correlation

An SRE should learn to correlate:

```text
Workspace
    ↓
Workload
    ↓
Deployment / Version
    ↓
Kubernetes Namespace
    ↓
Deployment / Job / Pod
    ↓
Container
    ↓
Node
```

This should be treated as an investigation method rather than assuming every installation exposes a fixed one-to-one mapping.

## 21. Incident Object Identity

Before troubleshooting, identify the affected object.

Ask:

```text
Which Control Plane?
Which Compute Plane / cluster?
Which Workspace?
Which workload?
Which workload type?
Which deployment/version?
Which Kubernetes namespace?
Which Pod?
Which node?
Which network path?
```

This converts a broad platform complaint into a bounded investigation.

## 22. Ownership and Failure Domains

Use this conceptual mapping:

| Layer | Investigation area |
|---|---|
| Control Plane | Platform/API/management |
| Cluster integration | `tfy-agent`, connectivity, reconciliation |
| Kubernetes | Scheduling, Pods, Services, storage |
| Application | Startup, configuration, application behavior |
| Model server | Inference runtime/vLLM |
| GPU runtime | NVIDIA/CUDA |
| Infrastructure | Compute, network, storage, IAM |

The responsible team depends on organizational design.

Therefore:

```text
Identify failure domain
        ↓
Gather evidence
        ↓
Route to responsible owner
```

## 23. Example — Service Failure

Reported symptom:

```text
TrueFoundry service is unhealthy
```

Observed state:

```text
Pod = CrashLoopBackOff
```

Container evidence:

```text
Application configuration error
```

Failure domain:

```text
Application/runtime configuration
```

The evidence does not support describing the incident simply as a TrueFoundry platform outage.

## 24. Example — GPU Deployment

Reported symptom:

```text
Model deployment remains Pending
```

Kubernetes investigation:

```bash
kubectl get pods -n <namespace>
kubectl describe pod <pod> -n <namespace>
```

Suppose Kubernetes reports:

```text
Insufficient nvidia.com/gpu
```

The evidence points toward:

```text
Kubernetes scheduling / GPU capacity
```

rather than automatically toward the TrueFoundry Control Plane.

## 25. Example — Network Failure

Suppose:

```text
Workload deployed       YES
Pod Running             YES
Pod Ready               YES
Application healthy     YES
External request        FAIL
```

Move the investigation toward:

```text
DNS
 ↓
Ingress / Gateway
 ↓
TLS
 ↓
Kubernetes Service
 ↓
Network policy / routing
```

Do not restart a healthy application simply because the external endpoint is unavailable.

## 26. SRE Communication

Avoid:

```text
TrueFoundry is down.
```

Prefer:

```text
The workload pod is in CrashLoopBackOff.
```

Avoid:

```text
TrueFoundry cannot deploy.
```

Prefer:

```text
The deployment request was submitted, but the expected
Kubernetes workload has not been observed in the target
compute environment.
```

Avoid:

```text
GPU is broken.
```

Prefer:

```text
The inference pod remains Pending and Kubernetes reports
insufficient allocatable nvidia.com/gpu.
```

Avoid:

```text
Service is down.
```

Prefer:

```text
The application pod is Running and Ready, but requests
through the external network path are failing.
```

Evidence-based terminology makes escalation considerably more effective.

## 27. Production Troubleshooting Model

Use:

```text
WHAT?
Identify platform object
       ↓
WHERE?
Identify workspace / compute plane
       ↓
WHAT TYPE?
Service / Job / Model / Agent / Workflow
       ↓
WHAT SHOULD EXIST?
Desired state
       ↓
WHAT ACTUALLY EXISTS?
Observed Kubernetes state
       ↓
WHERE DID IT STOP?
Failure domain
       ↓
WHO OWNS THAT LAYER?
Escalation
```

## 28. Production Rules

1. Identify the workload type before interpreting its health.
2. Never confuse a TrueFoundry Service with a Kubernetes `Service`.
3. Translate platform symptoms into Kubernetes evidence.
4. Compare desired state with observed state.
5. Understand reconciliation before manually changing resources.
6. Treat build, deployment, and runtime as different failure domains.
7. Protect Secret values during troubleshooting.
8. Separate application health from network reachability.
9. Identify the failing layer before assigning ownership.
10. Use precise evidence-based incident language.

## 29. Production Takeaway

The key skill from Part 1.3 is:

```text
Platform Terminology
        ↓
Object Identity
        ↓
Desired State
        ↓
Observed Kubernetes State
        ↓
Runtime Evidence
        ↓
Failure Domain
        ↓
Ownership
```

An SRE should be able to move between TrueFoundry terminology and infrastructure evidence without confusing the two layers.

## 30. Next Tutorial

Continue with:

**Part 1.4 — TrueFoundry and Kubernetes: How They Work Together**
