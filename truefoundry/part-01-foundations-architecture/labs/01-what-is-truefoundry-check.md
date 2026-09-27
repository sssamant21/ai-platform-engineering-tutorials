# Lab 1.1 — TrueFoundry Kubernetes Discovery

**Classification:** `[SAFE-READ]`  
**Associated Tutorial:** Part 1.1 — What Is TrueFoundry?  
**Audience:** SRE / Platform Engineering  
**Objective:** Discover TrueFoundry-related Kubernetes infrastructure without modifying cluster state.

---

## Safety Rules

This lab is discovery-only.

Do **not**:

- create resources
- delete resources
- patch resources
- edit resources
- scale workloads
- restart workloads
- perform rollouts
- modify ConfigMaps or Secrets
- change node labels or taints

Follow your organization's access, security, and production-change policies.

---

## 1. Confirm Kubernetes Context

```bash
kubectl config current-context
```

Record the result:

```text
Context:
Environment:
Cluster:
```

Do not continue against an unexpected cluster.

---

## 2. Inspect Nodes

```bash
kubectl get nodes -o wide
```

Validate:

- node count
- node readiness
- Kubernetes version
- node placement information available in your environment

---

## 3. Inspect Namespaces

```bash
kubectl get namespaces
```

Look for namespaces related to:

```text
truefoundry
tfy
monitoring
nvidia
gpu
```

Names vary by installation. Absence of one of these names does not prove that the associated capability is absent.

---

## 4. Discover TrueFoundry-Related Pods

```bash
kubectl get pods -A | grep -Ei 'truefoundry|tfy'
```

Record:

```text
Namespace:
Pod:
Status:
Ready:
Restarts:
```

If no results are returned, do not assume the platform is absent. Resource names and deployment architecture may differ.

---

## 5. Discover `tfy-agent`

```bash
kubectl get pods -A | grep -i 'tfy-agent'
```

If present, record:

```text
Namespace:
Pod:
Status:
Ready:
Restarts:
```

Inspect the pod:

```bash
kubectl describe pod <tfy-agent-pod> -n <namespace>
```

Focus on:

- node placement
- container state
- readiness
- restart count
- events

Do not modify the pod.

---

## 6. Inspect Related Deployments

```bash
kubectl get deployments -A | grep -Ei 'truefoundry|tfy'
```

Record any relevant resources.

---

## 7. Inspect Related Services

```bash
kubectl get svc -A | grep -Ei 'truefoundry|tfy'
```

Record:

```text
Namespace:
Service:
Type:
Cluster IP:
Ports:
```

---

## 8. Review Pod Placement

```bash
kubectl get pods -A -o wide
```

For relevant workloads, identify:

```text
Pod -> Namespace -> Node -> Status
```

This establishes where the workload is actually running.

---

## 9. Discover GPU Capacity

```bash
kubectl get nodes \
  -o custom-columns='NODE:.metadata.name,GPU:.status.allocatable.nvidia\.com/gpu'
```

Example interpretation:

```text
NODE            GPU
worker-cpu-01   <none>
worker-gpu-01   1
worker-gpu-02   4
```

Do not assume a node is GPU-capable solely from its name. Use resource evidence.

---

## 10. Discover NVIDIA / GPU Components

```bash
kubectl get pods -A | grep -Ei 'nvidia|gpu'
```

Depending on the environment, you may see GPU infrastructure components.

Record what actually exists rather than assuming a particular GPU architecture.

---

## 11. Review Kubernetes Events

```bash
kubectl get events -A --sort-by='.lastTimestamp'
```

Look for evidence related to:

- scheduling failures
- insufficient CPU
- insufficient memory
- insufficient GPU
- image pull failures
- mount failures
- readiness failures
- node conditions

Events are evidence. Do not assign root cause from a single event without correlating it with the workload state.

---

## 12. Build the Environment Map

Using the evidence collected above, create a simple map:

```text
TrueFoundry Control Plane
          |
          v
      tfy-agent
          |
          v
 Kubernetes Cluster
          |
     +----+----+
     |         |
     v         v
 CPU Nodes   GPU Nodes
                |
                v
          AI Workloads
```

If a component was not observed, mark it:

```text
Not observed / requires further validation
```

Do not invent missing topology.

---

## 13. Failure-Domain Exercise

For each example, identify the first layer you would investigate.

### Scenario A

```text
Pod remains Pending.
```

Consider:

```text
Kubernetes scheduling
CPU / memory availability
GPU availability
node selectors
taints / tolerations
```

### Scenario B

```text
Pod is Running, but vLLM reports that it cannot infer a device type.
```

Consider:

```text
GPU request
pod placement
GPU visibility
NVIDIA runtime
CUDA / framework
vLLM
```

### Scenario C

```text
Expected Kubernetes workload was never created.
```

Consider:

```text
TrueFoundry deployment/orchestration
tfy-agent
permissions / RBAC
Kubernetes API connectivity
```

These are investigation starting points, not predetermined root causes.

---

## 14. Lab Validation

The lab passes when you can provide evidence for the following:

- [ ] Kubernetes context verified
- [ ] Nodes discovered
- [ ] Namespaces inspected
- [ ] TrueFoundry-related resources searched
- [ ] `tfy-agent` searched and inspected if present
- [ ] Pod placement reviewed
- [ ] GPU allocatable resources checked
- [ ] NVIDIA/GPU components searched
- [ ] Kubernetes events reviewed
- [ ] Environment topology documented
- [ ] No cluster resources modified

---

## 15. Expected Outcome

You should now be able to explain:

```text
TrueFoundry
     |
     v
tfy-agent / platform integration
     |
     v
Kubernetes
     |
     +--> CPU workloads
     |
     +--> GPU workloads
             |
             v
       Model-serving runtime
```

Most importantly, you should be able to distinguish:

```text
Where an error is observed
```

from:

```text
Which layer actually caused the error
```

That distinction is fundamental to production SRE troubleshooting.
