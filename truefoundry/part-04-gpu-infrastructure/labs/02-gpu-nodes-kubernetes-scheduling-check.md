# Lab 4.2 — GPU Nodes and Kubernetes GPU Scheduling Check

**Canonical:** [Chapter 4.2](../02-gpu-nodes-kubernetes-scheduling/02-gpu-nodes-kubernetes-scheduling.md)  
**Production:** [SAFE-READ] only. **Optional exercise:** [LAB-MUTATE] in an approved disposable cluster.

## Goals
Verify node GPU resource advertisement, compare actual GPU Pod requests to allocatable capacity, identify placement exclusions and distinguish scheduler failures from container/runtime failures.

## 1. Preflight [SAFE-READ]

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get namespaces
```

Record cluster, context, timestamp, namespace, TrueFoundry deployment and Kubernetes controller. Stop if the target is unknown.

## 2. Node GPU inventory [SAFE-READ]

```bash
kubectl get nodes -L node-pool,gpu,gpu_type
kubectl describe node <gpu-node>
kubectl get node <gpu-node> -o jsonpath='{.status.capacity.nvidia\.com/gpu}{" capacity\n"}{.status.allocatable.nvidia\.com/gpu}{" allocatable\n"}'
```

Record capacity, allocatable, Ready state, `spec.unschedulable`, labels, taints and allocated-resource summary. **Do not** interpret `allocatable` as unallocated/free.

## 3. Workload scheduling evidence [SAFE-READ]

```bash
kubectl -n <namespace> get pods -o wide
kubectl -n <namespace> describe pod <gpu-pod>
kubectl -n <namespace> get pod <gpu-pod> -o yaml
kubectl -n <namespace> get events --sort-by=.lastTimestamp
```

Record PodScheduled condition, node assignment, GPU limits/requests for each container, selectors, affinity, tolerations and the most relevant scheduler event.

## 4. Candidate-node exclusion ledger [SAFE-READ]

Fill this table using observed evidence:

| Node | Ready | Unschedulable | Label/affinity | Taints | GPU allocatable vs allocated | CPU/memory/topology | Eligible |
|---|---|---|---|---|---|---|---|
| node-a | | | | | | | |
| node-b | | | | | | | |

Use `kubectl describe node <node>` and `kubectl -n <namespace> get pods --field-selector spec.nodeName=<node>` to validate resource consumers. The scheduler can exclude nodes for more than one reason.

## 5. Device-plugin registration [SAFE-READ]

```bash
kubectl get daemonsets -A
kubectl get pods -A -o wide
kubectl -n <plugin-namespace> describe daemonset <plugin-name>
kubectl -n <plugin-namespace> logs <plugin-pod> --tail=100
```

Discover actual plugin names and namespaces; do not assume `gpu-operator` or `kube-system`. If resource advertising is missing, preserve logs and escalate to the GPU platform owner.

## 6. Optional controlled lab [LAB-MUTATE]

Only use an **approved disposable GPU-enabled cluster**. Replace `<approved-cuda-image@sha256:digest>` with a real organization-approved immutable image. If your disposable cluster lacks GPUs, perform Sections 1–5 and skip this exercise.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-scheduling-ch04
  namespace: gpu-lab
spec:
  restartPolicy: Never
  containers:
    - name: check
      image: <approved-cuda-image@sha256:digest>
      command: ["/bin/sh", "-c"]
      args: ["nvidia-smi"]
      resources:
        requests:
          cpu: "100m"
          memory: "256Mi"
        limits:
          memory: "512Mi"
          nvidia.com/gpu: 1
```

Only create the namespace if it does not already exist and creation is approved:

```bash
kubectl create namespace gpu-lab
kubectl apply -f gpu-scheduling-ch04.yaml
kubectl -n gpu-lab get pod gpu-scheduling-ch04 -o wide
kubectl -n gpu-lab describe pod gpu-scheduling-ch04
kubectl -n gpu-lab logs gpu-scheduling-ch04
```

If node labels/taints require matching placement, change only the **disposable Pod manifest**, after checking the cluster policy. Never change shared nodes for the lab.

### Controlled failure scenarios (reason from evidence; no destructive fault injection)

1. **Insufficient GPU:** Explain which allocation data would prove GPU exhaustion.
2. **Selector mismatch:** Explain why matching hardware alone cannot schedule the Pod.
3. **Untolerated taint:** Identify the missing placement permission.
4. **No advertised GPU:** Distinguish device-plugin registration from free-capacity exhaustion.
5. **Scheduled but CUDA fails:** Show why this is beyond the scheduler.

### Cleanup [LAB-MUTATE]

```bash
kubectl -n gpu-lab delete pod gpu-scheduling-ch04 --ignore-not-found
kubectl -n gpu-lab get pods
```

Delete `gpu-lab` only if this lab created it exclusively and no other resources remain. Confirm no leftover GPU allocation.

## 7. Cloud checks

**EKS:** Record node group/Karpenter provisioner, GPU instance type, AMI/driver strategy and plugin registration.  
**AKS:** Record GPU VM SKU, node-pool labels/taints, driver strategy and resource advertising.  
**GKE:** Record accelerator node pool, GPU driver mode, allocatable GPU and scheduler constraints.

These are evidence categories, not claims that every cluster uses the same configuration.

## 8. Acceptance report

```text
Date/time and Kubernetes context:
Namespace and TrueFoundry deployment:
Affected Pod and owning controller:
GPU node inventory:
GPU capacity / allocatable / existing allocations:
Pod GPU limit and effective request:
Selector / affinity / taint / toleration evidence:
Scheduler event and PodScheduled condition:
Lowest Proven Healthy Layer:
First Failed Transition:
Blast radius:
Current actionable owner:
Proposed approved remediation:
Rollback plan:
End-to-end inference validation:
Optional lab result and cleanup:
```

**Pass:** Identify the actual first failed scheduling or post-scheduling transition with evidence, avoid all shared-node mutation, and clean up optional disposable resources.

**Next:** Chapter 4.3 — NVIDIA Drivers, Device Plugin and GPU Operator.
