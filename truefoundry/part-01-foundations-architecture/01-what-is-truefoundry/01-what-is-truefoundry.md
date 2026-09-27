# Part 1.1 — What Is TrueFoundry?

**Status:** Canonical  
**Track:** TrueFoundry — Foundations & Architecture  
**Audience:** SRE, Platform Engineering, DevOps, ML Infrastructure  
**Prerequisites:** Basic Kubernetes concepts

---

## 1. Introduction

TrueFoundry is an AI platform engineering layer used to deploy, operate, and manage machine-learning and AI workloads on infrastructure such as Kubernetes.

For an SRE or platform engineer, the important point is that TrueFoundry does not replace Kubernetes, cloud infrastructure, GPU drivers, networking, storage, or the model server. It provides a platform layer above these systems.

A production engineer therefore needs to understand both TrueFoundry and the infrastructure underneath it.

---

## 2. Learning Objectives

After completing this tutorial, you should be able to:

- explain where TrueFoundry fits in an AI platform architecture;
- distinguish the platform layer from the underlying Kubernetes infrastructure;
- identify the control-plane and compute-plane concepts at a high level;
- explain the role of `tfy-agent`;
- understand where CPU, GPU, CUDA, and model-serving components fit;
- identify operational ownership boundaries;
- use a failure-domain model when troubleshooting production incidents.

---

## 3. What Is TrueFoundry?

TrueFoundry provides a platform for deploying and operating AI and machine-learning workloads.

From an infrastructure perspective, a useful simplified model is:

```text
Developer / ML Engineer
        |
        v
TrueFoundry Platform
        |
        v
Kubernetes
        |
        v
Cloud / Compute / GPU / Network / Storage
```

TrueFoundry provides higher-level workflows while Kubernetes remains responsible for scheduling and running containers.

This distinction is critical during troubleshooting.

An error visible through the TrueFoundry platform does not automatically mean that TrueFoundry caused the failure.

---

## 4. Why Use a Platform Layer?

Running AI workloads directly on Kubernetes requires application and ML teams to understand many infrastructure concerns, including:

- Deployments
- Services
- Ingress
- ConfigMaps
- Secrets
- CPU and memory requests
- GPU resource requests
- node scheduling
- autoscaling
- container registries
- observability
- model-serving configuration

A platform layer can provide standardized workflows over these infrastructure components.

The objective is not to eliminate Kubernetes. The objective is to make common AI deployment and operational workflows easier to consume while preserving Kubernetes as the underlying orchestration platform.

---

## 5. TrueFoundry and Kubernetes

A useful SRE mental model is:

```text
TrueFoundry
     |
     | creates / manages workloads
     v
Kubernetes API
     |
     +--> Deployments
     +--> Pods
     +--> Services
     +--> ConfigMaps / Secrets
     +--> Storage
     +--> CPU / Memory
     +--> GPU resources
```

When troubleshooting, engineers should inspect both the platform view and the Kubernetes resources created for the workload.

For example, if an AI service fails to start, the underlying problem could be:

- pod scheduling;
- insufficient CPU or memory;
- unavailable GPU capacity;
- image pull failure;
- missing secret;
- RBAC failure;
- storage failure;
- network or DNS failure;
- NVIDIA driver/runtime failure;
- CUDA compatibility;
- model-server configuration.

---

## 6. Architecture Overview

At a high level, think about the environment as two major areas:

```text
+-----------------------------+
| TrueFoundry Platform        |
| Control / Management Layer  |
+-------------+---------------+
              |
              v
+-----------------------------+
| Customer Compute Plane      |
| Kubernetes Cluster          |
|                             |
| tfy-agent                   |
| Application Workloads       |
| Model Servers               |
| CPU / GPU Pods              |
+-------------+---------------+
              |
              v
+-----------------------------+
| Cloud Infrastructure        |
| Nodes / GPU / Network       |
| Storage / IAM               |
+-----------------------------+
```

The exact implementation can vary by deployment model and platform version, but this separation is a useful operational starting point.

---

## 7. Control Plane

The control or management plane provides the platform-facing experience used to configure and operate workloads.

From an SRE perspective, think of it as the layer where deployment intent and platform configuration originate.

The compute resources themselves still run in the Kubernetes environment.

Later tutorials will examine the control-plane and compute-plane interaction in more detail.

---

## 8. Compute Plane

The compute plane is where application and model-serving workloads actually execute.

Typical components include:

- Kubernetes worker nodes;
- CPU nodes;
- GPU nodes;
- application pods;
- model-serving pods;
- networking;
- storage;
- container runtime;
- NVIDIA runtime components.

This means many failures surfaced through an AI platform are ultimately infrastructure failures inside the compute plane.

---

## 9. What Is `tfy-agent`?

`tfy-agent` is an important integration component between TrueFoundry and the Kubernetes environment.

For an SRE, it should be treated as part of the platform-to-cluster integration path.

A simplified model is:

```text
TrueFoundry Platform
        |
        v
    tfy-agent
        |
        v
Kubernetes Cluster
        |
        v
AI / ML Workloads
```

During platform troubleshooting, validating the health and placement of `tfy-agent` is therefore an important early diagnostic step.

Do not assume every workload failure is an agent failure. The agent is only one layer in the end-to-end execution path.

---

## 10. CPU Workloads

CPU-based workloads ultimately depend on normal Kubernetes compute resources.

Typical dependencies include:

```text
TrueFoundry
   |
Kubernetes
   |
Pod
   |
CPU / Memory
```

Failures may involve resource requests, scheduling, container startup, networking, storage, application configuration, or application code.

---

## 11. GPU Workloads

GPU workloads introduce additional infrastructure layers.

A simplified dependency chain is:

```text
TrueFoundry
      |
Kubernetes
      |
GPU Pod
      |
NVIDIA Device Plugin / GPU Operator
      |
NVIDIA Driver
      |
CUDA Runtime
      |
Physical GPU
```

Depending on the environment, additional runtime and operator components may exist.

This is why a message such as "GPU not detected" must not immediately be classified as a TrueFoundry platform problem.

---

## 12. Where Does vLLM Fit?

vLLM is a model-serving engine commonly used for large language model inference.

It runs inside the workload layer rather than replacing the platform or Kubernetes.

A useful model is:

```text
Client
  |
  v
Model Endpoint
  |
  v
vLLM
  |
  v
PyTorch / CUDA
  |
  v
NVIDIA GPU
```

TrueFoundry may be used to deploy and operate the workload, while vLLM is responsible for model-serving behavior.

This distinction becomes important when troubleshooting startup errors, CUDA compatibility, GPU memory, model loading, and inference performance.

---

## 13. SRE Responsibility Boundaries

Production incidents become easier to investigate when ownership boundaries are explicit.

### Application / Model Team

Typically owns:

- model selection;
- application logic;
- model configuration;
- model-server parameters;
- application dependencies.

### TrueFoundry Platform Layer

Typically provides:

- platform workflows;
- workload deployment abstractions;
- platform-to-cluster integration;
- platform configuration and management capabilities.

### Platform / SRE Team

Typically owns or supports:

- Kubernetes health;
- workload scheduling;
- namespaces;
- RBAC;
- networking;
- ingress;
- storage;
- resource capacity;
- observability;
- cluster-level platform components.

### Cloud / Infrastructure Layer

Typically includes:

- compute instances;
- GPU instances;
- networking;
- storage;
- IAM;
- infrastructure capacity.

Exact ownership varies by organization. The important practice is to identify the failing layer before assigning the incident to a team.

---

## 14. Production Failure-Domain Model

Use the following model during incident investigation:

```text
Application / Model
        |
        v
Model Server / vLLM
        |
        v
TrueFoundry
        |
        v
Kubernetes
        |
        v
Container / GPU Runtime
        |
        v
NVIDIA / CUDA
        |
        v
Compute / Network / Storage
```

Troubleshooting should move through these layers using evidence.

Avoid conclusions such as:

> The error appears in TrueFoundry, therefore TrueFoundry is the root cause.

The platform may simply be exposing an error originating from a lower layer.

---

## 15. Example: GPU / vLLM Failure

Consider a workload that reports:

```text
No CUDA runtime is found
```

or:

```text
Failed to infer device type
```

The failure should be investigated across several layers.

### Workload

Check whether the pod requested a GPU.

```bash
kubectl describe pod <pod> -n <namespace>
```

Look for:

```text
nvidia.com/gpu
```

### Scheduling

Check where the pod is running.

```bash
kubectl get pod <pod> -n <namespace> -o wide
```

### Node

Check whether the node advertises GPU capacity.

```bash
kubectl get nodes -o custom-columns='NODE:.metadata.name,GPU:.status.allocatable.nvidia\.com/gpu'
```

### NVIDIA Components

Check GPU-related platform components.

```bash
kubectl get pods -A | grep -Ei 'nvidia|gpu'
```

### Application Runtime

After infrastructure has been validated, investigate:

- CUDA compatibility;
- PyTorch build;
- vLLM version;
- container image;
- model-server configuration.

This approach prevents premature attribution of an infrastructure problem to the AI platform.

---

## 16. First-Response SRE Checks

Start with read-only discovery.

```bash
kubectl config current-context
```

```bash
kubectl get nodes -o wide
```

```bash
kubectl get pods -A | grep -Ei 'truefoundry|tfy'
```

```bash
kubectl get pods -A | grep -i 'tfy-agent'
```

```bash
kubectl get events -A --sort-by='.lastTimestamp'
```

For GPU environments:

```bash
kubectl get nodes -o custom-columns='NODE:.metadata.name,GPU:.status.allocatable.nvidia\.com/gpu'
```

```bash
kubectl get pods -A | grep -Ei 'nvidia|gpu'
```

These commands are intentionally read-only.

---

## 17. Hands-On Lab

Complete:

```text
labs/truefoundry-discovery-check.md
```

The lab is classified:

```text
[SAFE-READ]
```

It focuses on discovering TrueFoundry-related Kubernetes resources without changing cluster state.

---

## 18. Validation Checklist

After completing the tutorial and lab, you should be able to answer:

- What role does TrueFoundry play?
- Does TrueFoundry replace Kubernetes?
- Where do workloads actually run?
- What role does `tfy-agent` play?
- Which layers are involved in GPU workloads?
- Where does vLLM fit?
- Why should an error displayed by TrueFoundry not automatically be attributed to TrueFoundry?
- What are the first read-only Kubernetes checks during an incident?

---

## 19. Production Takeaways

The most important operational principles are:

1. TrueFoundry is a platform layer over infrastructure such as Kubernetes.
2. Kubernetes remains a critical execution and troubleshooting layer.
3. GPU workloads add NVIDIA, CUDA, and hardware dependencies.
4. vLLM belongs to the model-serving layer.
5. `tfy-agent` is part of the platform-to-cluster integration path.
6. Platform-visible errors may originate from lower infrastructure layers.
7. Troubleshooting should follow evidence across failure domains.
8. Production investigation should begin with safe, read-only discovery whenever possible.

---

## 20. Next Tutorial

**Part 1.2 — TrueFoundry Architecture: Control Plane, Compute Plane & `tfy-agent`**

The next tutorial will examine the architecture in more detail, including:

- control-plane responsibilities;
- compute-plane responsibilities;
- `tfy-agent`;
- workload deployment flow;
- Kubernetes interaction;
- SRE health checks;
- common failure points.
