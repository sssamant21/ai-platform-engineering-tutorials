# Lab 4.5 — Dedicated GPU Node Pools and Resource Isolation Check

**Canonical:** [Chapter 4.5](../05-dedicated-gpu-node-pools-isolation/05-dedicated-gpu-node-pools-isolation.md)  
**Default:** [SAFE-READ]. Optional scheduling validation: [LAB-MUTATE] only in an approved disposable GPU-enabled cluster.

## Objectives

Inventory GPU pools, validate selectors and taints, audit GPU allocation, identify placement exclusions, and evaluate tenancy, capacity and cost tradeoffs without changing production nodes.

## 0. Safety preflight

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get namespaces
```

Confirm cluster, environment, TrueFoundry deployment, namespace and current time. Never edit shared node labels/taints, autoscaler settings, ResourceQuotas or production workloads for this lab.

## 1. Discover GPU pools [SAFE-READ]

```bash
kubectl get nodes -L node-pool,gpu,gpu_type
kubectl get nodes --show-labels
kubectl describe node <gpu-node>
kubectl get node <gpu-node> -o jsonpath='{.status.capacity.nvidia\.com/gpu}{" capacity\n"}{.status.allocatable.nvidia\.com/gpu}{" allocatable\n"}{.spec.taints}{" taints\n"}'
```

Record actual pool labels, GPU SKU, Ready state, taints, capacity, allocatable and allocated resource requests. Labels used in examples are not guaranteed to exist.

## 2. Inspect workload placement [SAFE-READ]

```bash
kubectl -n <namespace> get pods -o wide
kubectl -n <namespace> describe pod <gpu-pod>
kubectl -n <namespace> get pod <gpu-pod> -o yaml
kubectl -n <namespace> get events --sort-by=.lastTimestamp
```

Capture `nodeSelector`, required node affinity, tolerations, GPU limits, assigned node and scheduler events. Explain why a toleration alone does not select a node.

## 3. Resource isolation and quota [SAFE-READ]

```bash
kubectl -n <namespace> get resourcequota
kubectl -n <namespace> describe resourcequota
kubectl -n <namespace> get limitrange
kubectl -n <namespace> get serviceaccounts
```

Check whether GPU quota and workload policies are enforced by the namespace/admission configuration. **Do not** assume namespaces alone prevent GPU tenant interference.

## 4. GPU allocation and fragmentation [SAFE-READ]

```bash
kubectl describe node <gpu-node>
kubectl -n <namespace> get pods --field-selector spec.nodeName=<gpu-node> -o wide
```

Use the node's **Allocated resources** summary for cluster-wide scheduled resource commitments. A namespaced Pod query does not include other namespaces.

Complete this worksheet:

| GPU pool | Nodes | Total allocatable GPUs | Assigned GPU requests | Approximate unassigned GPUs | Pending GPU Pods | Main blocker |
|---|---|---|---|---|---|---|
| pool-a | | | | | | |
| pool-b | | | | | | |

The approximate unassigned count is not a guarantee of schedulability; account for CPU, memory, topology, selectors and taints.

## 5. Candidate-node exclusion ledger [SAFE-READ]

| Node | Ready | Label match | Taints tolerated | GPUs fit | Other resources fit | Eligible |
|---|---|---|---|---|---|---|
| node-a | | | | | | |
| node-b | | | | | | |

Identify the **first failed placement constraint** for each candidate and the owner of that constraint.

## 6. Optional disposable scheduling smoke test [LAB-MUTATE]

Only on an approved disposable GPU cluster. Replace `<approved-cuda-image@sha256:digest>` with a real organization-approved immutable CUDA image containing `nvidia-smi`. Adapt node selectors and tolerations to the actual disposable pool. Do not alter nodes.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pool-isolation-ch04
  namespace: gpu-lab
spec:
  restartPolicy: Never
  nodeSelector:
    node-pool: gpu-reserved
    gpu: "true"
    gpu_type: A10G
  tolerations:
    - key: dedicated
      operator: Equal
      value: gpu
      effect: NoSchedule
  containers:
    - name: gpu-check
      image: <approved-cuda-image@sha256:digest>
      command: ["/bin/sh", "-c"]
      args: ["nvidia-smi"]
      resources:
        limits:
          nvidia.com/gpu: 1
```

Create the namespace only if absent and authorized:

```bash
kubectl create namespace gpu-lab
```

Then:

```bash
kubectl apply -f gpu-pool-isolation-ch04.yaml
kubectl -n gpu-lab get pod gpu-pool-isolation-ch04 -o wide
kubectl -n gpu-lab describe pod gpu-pool-isolation-ch04
kubectl -n gpu-lab logs gpu-pool-isolation-ch04
```

If no matching GPU pool exists, adjust only the disposable manifest to match the approved pool or skip the exercise. A Pending Pod can be a valid diagnostic result; preserve events.

### Cleanup [LAB-MUTATE]

```bash
kubectl -n gpu-lab delete pod gpu-pool-isolation-ch04 --ignore-not-found
kubectl -n gpu-lab get pods
```

Delete `gpu-lab` only if created solely for this lab and confirmed empty.

## 7. Non-destructive failure scenarios

1. **Toleration without selector:** explain why the Pod can still land on another eligible node.
2. **Selector without toleration:** explain the scheduler's untolerated-taint rejection.
3. **Insufficient GPUs despite low utilization:** distinguish committed GPU requests from device utilization.
4. **Autoscaler cannot provision:** identify cloud quota, GPU SKU supply and impossible placement constraints.
5. **MIG or time-slicing proposal:** verify GPU model support and document isolation/latency tradeoffs before enabling.
6. **Tenant contention:** propose admission, quota and workload policy controls without treating taints as a security boundary.

## 8. FinOps worksheet

```text
GPU pool / cloud provider:
GPU SKU and count:
Node-hours / billing period:
Estimated GPU node spend (from billing, not guesses):
Allocated GPU-hours:
Measured utilization / VRAM:
Warm spare capacity:
P95/P99 inference latency and error rate:
GPU Pod Pending time / provisioning lead time:
Fragmentation / unused capacity:
Proposed rightsizing or sharing:
Isolation and reliability risks:
Rollback trigger:
```

Use actual billing and telemetry for calculations. Do not invent cloud rates or infer spend from GPU count alone.

## 9. Acceptance evidence

```text
Date/time and Kubernetes context:
TrueFoundry deployment / namespace:
GPU pool labels / taints:
Pool GPU capacity and allocated requests:
Quota and admission evidence:
Workload selector / toleration:
Candidate-node exclusion ledger:
First failed placement constraint:
Tenant-isolation findings:
Autoscaler/provisioning evidence:
FinOps and SLO considerations:
Approved remediation / rollback:
Optional smoke-test outcome and cleanup:
```

**Pass:** demonstrate a defensible pool eligibility analysis, distinguish resource allocation from utilization, document tenant-isolation limitations, and clean up optional lab resources.

**Next:** Chapter 4.6 — GPU Validation, Monitoring and Troubleshooting.
