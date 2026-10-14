# Lab 4.7 — Production GPU Infrastructure Recovery Acceptance

**Canonical:** [Chapter 4.7](../07-production-gpu-infrastructure-recovery/07-production-gpu-infrastructure-recovery.md)  
**Type:** Part 04 final SRE capstone  
**Default:** [SAFE-READ] tabletop and evidence gathering. **Optional:** [LAB-MUTATE] only in an approved disposable GPU-enabled cluster.

## 1. Objectives and scoring

Demonstrate incident triage, first-failure isolation, safe remediation planning, recovery gates, rollback and cleanup. Pass when all required evidence is complete and every unexecuted check is clearly marked. A read-only tabletop is not a live recovery test.

**Assessment:** 20 points each for GPU node/registration evidence, scheduling and isolation, CUDA/framework evidence, incident classification, and recovery/rollback acceptance. Suggested passing score: **80/100**, with **zero unapproved production mutations** required.

## 2. Safety gate

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get namespaces
```

Record environment, operator, date/time and approval. If context is production, **do not** create test Pods, change taints/labels, drain nodes, restart GPU components, alter autoscaling or run fault injection. Use authorized read-only inspection only.

## 3. Node and GPU inventory [SAFE-READ]

```bash
kubectl get nodes -L node-pool,gpu,gpu_type
kubectl describe node <gpu-node>
kubectl get node <gpu-node> -o jsonpath='{.status.capacity.nvidia\.com/gpu}{" capacity\n"}{.status.allocatable.nvidia\.com/gpu}{" allocatable\n"}'
```

Record GPU model, node Ready status, capacity, allocatable, allocated requests, taints, pool labels and GPU stack ownership. **Do not** confuse allocatable with currently unassigned GPUs.

## 4. Device plugin, operator and exporter [SAFE-READ]

```bash
kubectl get daemonsets -A
kubectl get pods -A -o wide
kubectl get services -A
kubectl -n <gpu-namespace> describe pod <device-plugin-pod>
kubectl -n <gpu-namespace> logs <device-plugin-pod> --tail=150
```

Discover real namespaces and component names before substituting placeholders. If GPU Operator is absent, mark **Not installed** rather than a failure. If DCGM exporter is absent or inaccessible, record the telemetry gap and owner.

## 5. Workload and scheduling [SAFE-READ]

```bash
kubectl -n <namespace> get deployments,statefulsets,pods -o wide
kubectl -n <namespace> describe pod <gpu-pod>
kubectl -n <namespace> get events --sort-by=.lastTimestamp
kubectl -n <namespace> logs <gpu-pod> --tail=150
```

Collect PodScheduled condition, `nvidia.com/gpu` limit, selectors, affinity, tolerations, assigned node, container state, image digest and vLLM/CUDA logs. Use node's **Allocated resources** to account for Pods across namespaces.

## 6. Optional disposable CUDA smoke test [LAB-MUTATE]

**Prerequisites:** approved disposable GPU cluster, device plugin working, GPU quota available, approved immutable PyTorch image with CUDA support. Replace the image placeholder and adapt placement to the disposable pool; do not edit shared nodes.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-recovery-ch04
  namespace: gpu-lab
spec:
  restartPolicy: Never
  containers:
    - name: pytorch-check
      image: <approved-pytorch-cuda-image@sha256:digest>
      command: ["python", "-c"]
      args:
        - |
          import torch
          print("torch", torch.__version__)
          print("build_cuda", torch.version.cuda)
          assert torch.cuda.is_available(), "CUDA not available"
          print("CUDA_EXEC_PASS", torch.ones(4,device="cuda").sum().item())
      resources:
        requests:
          cpu: "100m"
          memory: "512Mi"
        limits:
          memory: "2Gi"
          nvidia.com/gpu: 1
```

If `gpu-lab` is absent and namespace creation is approved:

```bash
kubectl create namespace gpu-lab
```

Then:

```bash
kubectl apply -f gpu-recovery-ch04.yaml
kubectl -n gpu-lab get pod gpu-recovery-ch04 -o wide
kubectl -n gpu-lab describe pod gpu-recovery-ch04
kubectl -n gpu-lab logs gpu-recovery-ch04
```

**Expected:** Pod schedules to an authorized GPU node and prints `CUDA_EXEC_PASS 4.0`. If it remains Pending or fails, collect evidence; do not modify production pool policy to force success.

Cleanup:

```bash
kubectl -n gpu-lab delete pod gpu-recovery-ch04 --ignore-not-found
kubectl -n gpu-lab get pods
```

Delete the namespace only if it was created exclusively for this lab and is empty. Confirm no residual GPU allocations.

## 7. Twelve fault-classification exercises

| Case | Symptom | Evidence to provide | First failed transition |
|---|---|---|---|
| F1 | Insufficient GPUs | GPU allocatable, assigned requests, Pod events | Capacity → scheduling |
| F2 | Taint or selector mismatch | Pod placement rules, node labels/taints | Policy → scheduling |
| F3 | No advertised GPU | Host driver status, plugin logs | Plugin → registration |
| F4 | GPU allocated, container GPU absent | Pod events, runtime evidence | Allocation → container |
| F5 | PyTorch CUDA false | Image build, driver/runtime evidence | Container → framework |
| F6 | vLLM device inference failure | Pod, CUDA checks, vLLM logs | **Determine from evidence** |
| F7 | CUDA out of memory | VRAM, concurrency, model size | CUDA → model memory |
| F8 | Xid/device reset | Xid code, GPU identity, node events | Hardware/driver |
| F9 | Exporter down, inference healthy | Scrape health and inference metrics | Telemetry pipeline |
| F10 | GPU autoscaling blocked | Cloud quota, autoscaler events | Provisioning |
| F11 | Low utilization, high P99 | Queueing, CPU, batching, tracing | Serving |
| F12 | Unapproved tenant on GPU pool | Taints, admission, quotas, Pod placement | Isolation policy |

For each: document the **lowest proven healthy layer**, first failed transition, blast radius, actionable owner, safe remediation and rollback.

## 8. Recovery gate worksheet

| Gate | Result (Pass/Fail/Not run) | Evidence |
|---|---|---|
| G1 GPU node and driver healthy | | |
| G2 GPU resource advertised | | |
| G3 Pod scheduled to approved pool | | |
| G4 GPU exposed in container | | |
| G5 CUDA operation succeeds | | |
| G6 vLLM/model readiness and inference | | |
| G7 GPU metrics fresh or gap owned | | |
| G8 Inference SLO baseline restored | | |
| G9 Lab cleanup complete | | |

**Do not** mark G6 or G8 Pass without actual inference evidence. Mark Not run when no model endpoint is available.

## 9. Production change and rollback tabletop

```text
Incident ID / environment / time:
TrueFoundry workspace / deployment:
Customer impact and SLO:
First failed transition and evidence:
Affected GPU nodes / GPU SKU / workloads:
Proposed minimal supported change:
Authoritative source (TrueFoundry / GitOps / IaC / node image):
Change approver / maintenance window:
Canary plan / stop conditions:
Driver/runtime/model compatibility:
Capacity and tenant blast radius:
Rollback artifact / procedure:
RTO/RPO if established:
Post-change G1–G9 acceptance:
Escalation owner:
```

**Failure rule:** any unapproved shared-node mutation, undocumented tenant disruption, fabricated evidence or uncleaned disposable GPU workload fails the exercise regardless of numeric score.

## 10. Final acceptance record

```text
Candidate / date / cluster:
Read-only tabletop or executed disposable lab:
Evidence links / incident timeline:
GPU registration and scheduling score (0–40):
CUDA/runtime validation score (0–20):
Fault isolation score (0–20):
Recovery/rollback score (0–20):
Total score:
Unexecuted checks:
Safety and cleanup confirmed:
Reviewer and next actions:
```

**Completion:** Part 04 documentation is published when the chapter and lab are present in GitHub. The actual capstone execution, formal vendor validation and production SRE review are separate milestones.

**Next:** Part 05 — Application Deployment, following the existing curriculum master layout.
