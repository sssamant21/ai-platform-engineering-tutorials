# Lab 1.2 — TrueFoundry Architecture & Compute-Plane Validation

**Classification:** `[SAFE-READ]`  
**Associated Tutorial:** Part 1.2 — TrueFoundry Architecture: Control Plane, Compute Plane & `tfy-agent`  
**Audience:** SRE / Platform Engineering  
**Mode:** Observation only

## 1. Purpose

This lab validates the TrueFoundry architecture from the Kubernetes Compute Plane without changing cluster state.

The objective is to connect the architectural concepts from Part 1.2 with observable Kubernetes evidence.

## 2. Learning Objectives

By completing this lab, you should be able to:

- identify the active Kubernetes context
- identify cluster nodes
- discover TrueFoundry-related Kubernetes components
- locate `tfy-agent`
- inspect its health and placement
- discover supporting platform resources
- inspect recent Kubernetes events
- inspect CPU and memory capacity
- identify GPU capacity where applicable
- distinguish management-path evidence from runtime-path evidence
- complete the lab without modifying cluster state

## 3. Safety Gate

This lab is `[SAFE-READ]`.

Allowed operations include:

```text
kubectl get
kubectl describe
kubectl logs
kubectl config current-context
```

Do not perform:

```text
kubectl create
kubectl apply
kubectl delete
kubectl edit
kubectl patch
kubectl scale
kubectl rollout restart
helm install
helm upgrade
helm uninstall
```

Do not modify resources as part of this lab.

## 4. Validation Flow

```text
Current Context
      ↓
Cluster / Nodes
      ↓
TrueFoundry Components
      ↓
tfy-agent
      ↓
Supporting Components
      ↓
Kubernetes Resources
      ↓
Events
      ↓
CPU / GPU Capacity
      ↓
Architecture Mapping
```

## 5. Confirm Kubernetes Context

Run:

```bash
kubectl config current-context
```

Record the result.

Expected outcome:

- the intended cluster context is displayed
- no configuration is modified

Do not continue against an unexpected production cluster without following your organization's access and change policies.

## 6. Inspect Cluster Nodes

```bash
kubectl get nodes -o wide
```

Observe:

- node names
- node readiness
- Kubernetes versions
- internal addresses
- operating system/container runtime information when displayed

Record whether all expected nodes are `Ready`.

## 7. Discover TrueFoundry Components

```bash
kubectl get pods -A | grep -Ei 'truefoundry|tfy'
```

Record:

- namespace
- pod name
- readiness
- status
- restart count
- age

Do not assume that every TrueFoundry installation has identical component names.

## 8. Locate `tfy-agent`

```bash
kubectl get pods -A | grep -i 'tfy-agent'
```

Record:

```text
Namespace:
Pod:
Ready:
Status:
Restarts:
Age:
```

If no matching component is found, record that observation rather than making cluster changes.

## 9. Inspect `tfy-agent`

After identifying the pod:

```bash
kubectl describe pod <tfy-agent-pod> -n <namespace>
```

Review:

- node placement
- container state
- restart count
- resource requests and limits
- environment references
- volumes
- ServiceAccount
- conditions
- recent events

Do not expose or copy sensitive values into tickets or documentation.

## 10. Review Agent Logs

Where operational policy permits log access:

```bash
kubectl logs <tfy-agent-pod> -n <namespace> --tail=100
```

Look for evidence of:

- startup failures
- authentication/authorization errors
- connectivity problems
- reconciliation errors
- repeated retries
- Kubernetes API errors

Do not restart the pod as part of this lab.

## 11. Discover Supporting Deployments

```bash
kubectl get deployments -A | grep -Ei 'truefoundry|tfy'
```

Record relevant components and their readiness.

This helps establish that the Compute Plane may contain multiple platform components rather than only `tfy-agent`.

## 12. Discover Services

```bash
kubectl get svc -A | grep -Ei 'truefoundry|tfy'
```

Observe service names and types.

Do not modify networking resources.

## 13. Discover ConfigMaps

```bash
kubectl get configmaps -A | grep -Ei 'truefoundry|tfy'
```

This step identifies configuration objects by name.

Do not assume that every matching ConfigMap belongs to the same platform function.

## 14. Identify Secret Objects Safely

If required for architecture discovery:

```bash
kubectl get secrets -A | grep -Ei 'truefoundry|tfy'
```

Only identify object names and namespaces.

**Do not retrieve or display Secret contents.**

## 15. Review Kubernetes Events

```bash
kubectl get events -A --sort-by='.lastTimestamp'
```

Look for:

- scheduling failures
- image-pull failures
- probe failures
- mount failures
- node pressure
- resource constraints
- platform-component warnings

Events provide evidence for determining which failure domain deserves investigation.

## 16. Inspect CPU and Memory Capacity

```bash
kubectl get nodes -o custom-columns='NODE:.metadata.name,CPU:.status.allocatable.cpu,MEMORY:.status.allocatable.memory'
```

Record whether the expected compute capacity is visible.

This is capacity discovery only; it does not establish actual utilization.

## 17. Inspect GPU Capacity

For environments that use GPU workloads:

```bash
kubectl get nodes -o custom-columns='NODE:.metadata.name,GPU:.status.allocatable.nvidia\.com/gpu'
```

Then:

```bash
kubectl get pods -A | grep -Ei 'nvidia|gpu'
```

Record:

- GPU-capable nodes
- advertised GPU capacity
- relevant NVIDIA/GPU components

If the cluster is CPU-only, mark this section as not applicable.

## 18. Map the Observed Architecture

Using the evidence collected above, map your environment:

```text
TrueFoundry Control Plane
          │
          │ management path
          ▼
Cluster Integration
          │
      tfy-agent
          │
          ▼
Kubernetes Compute Plane
          │
    ┌─────┴─────┐
    ▼           ▼
Application   Model Server
                │
                ▼
            GPU / CUDA
```

For each observable layer, record the Kubernetes evidence that supports your conclusion.

## 19. Management vs Runtime Exercise

Classify each example before investigating it.

### Scenario A

A new deployment request is accepted, but no expected Kubernetes workload appears.

Primary investigation area:

```text
Management / integration path
```

Validate the conclusion with evidence before assigning ownership.

### Scenario B

The pod exists and is running, but the application endpoint returns errors.

Primary investigation area:

```text
Runtime / application path
```

### Scenario C

A vLLM pod is `Pending` with GPU scheduling events.

Primary investigation area:

```text
Kubernetes scheduling / GPU capacity
```

Do not initially classify this as a TrueFoundry Control Plane failure.

## 20. Evidence Collection Template

Record:

```text
Kubernetes context:
Cluster/node health:

TrueFoundry components discovered:
tfy-agent namespace:
tfy-agent pod:
tfy-agent health:
tfy-agent restarts:

Supporting deployments:
Supporting services:

Recent warning events:

CPU capacity:
Memory capacity:
GPU capacity:

Management-path observations:
Runtime-path observations:

Potential failure domain:
Evidence supporting conclusion:
```

Do not record credentials, tokens, Secret values, or other sensitive material.

## 21. Acceptance Checklist

```text
[ ] Correct Kubernetes context identified
[ ] Compute nodes identified
[ ] TrueFoundry components discovered
[ ] tfy-agent namespace identified
[ ] tfy-agent health inspected
[ ] Supporting resources discovered
[ ] Kubernetes events reviewed
[ ] CPU/memory capacity inspected
[ ] GPU capacity checked where applicable
[ ] Management path understood
[ ] Runtime path understood
[ ] Failure domains can be distinguished
[ ] No Secret contents exposed
[ ] No cluster state modified
```

## 22. Completion Criteria

The lab is complete when:

1. the engineer can identify the observable TrueFoundry Compute Plane components;
2. `tfy-agent` has been located and inspected where present;
3. Kubernetes health and recent events have been reviewed;
4. compute/GPU capacity has been inspected where applicable;
5. management and runtime paths can be distinguished; and
6. no Kubernetes state was modified.

**Lab result:** PASS only when all applicable acceptance checks are satisfied.
