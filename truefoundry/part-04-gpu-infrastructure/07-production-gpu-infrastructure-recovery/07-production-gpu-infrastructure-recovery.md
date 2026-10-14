# 4.7 — Production GPU Infrastructure Recovery Lab

**Part:** 04 — GPU Infrastructure · **Capstone:** 7 of 7  
**Audience:** TrueFoundry platform engineers, GPU infrastructure owners, Kubernetes SREs, ML serving teams  
**Scope:** Integrates Chapters 4.1–4.6: GPU architecture, scheduling, NVIDIA drivers and device plugin, CUDA/runtime compatibility, pool isolation, observability and incident recovery.  
**Safety classification:** All default production actions are **[SAFE-READ]**. Fault injection, Pod creation, node drains, driver/plugin changes and failover are **[LAB-MUTATE]** or **[CHANGE-CONTROL]** only in approved environments.

> **Completion means the capstone documentation is published, not that any live production recovery exercise, vendor validation or formal SRE review has been performed.** This chapter defines the acceptance procedure and evidence to collect.

## 1. Outcomes and success criteria

By the end of this capstone, an operator can:
1. Establish the current cluster/TrueFoundry workload identity and business impact.
2. Distinguish GPU node provisioning, device registration, Pod scheduling, container GPU exposure, CUDA initialization and model readiness.
3. Isolate the **first failed transition** using evidence rather than symptoms.
4. Recover an approved disposable GPU workload or present a reviewed production change/rollback plan.
5. Validate GPU telemetry and user-visible inference service-level objectives.
6. Produce a reproducible incident timeline, ownership map and recovery acceptance report.

**Pass requires:** correct failure classification, evidence for each relevant layer, no unapproved production mutation, documented rollback, successful lab smoke test when a GPU lab is available, and no residual lab resources. A real production change is *not* required to pass the read-only assessment.

## 2. Reference deployment and dependencies

```text
TrueFoundry workspace/deployment
  → Kubernetes controller/Pod desired state
  → GPU pool selector + tolerations + resource limit
  → Scheduler → GPU node (Ready, schedulable, free allocatable resource)
  → kubelet + NVIDIA device plugin
  → host driver + NVIDIA container toolkit/runtime
  → container GPU visibility
  → CUDA-enabled PyTorch / vLLM
  → model readiness → inference endpoint → SLOs
                          ↑
                DCGM / Prometheus / logs
```

The path is diagnostic, not a universal installation order. GPU Operator may manage several components, or a cloud-managed node image may own drivers/toolkit. **Do not install duplicate GPU stack components.**

## 3. Lab topology, prerequisites and safeguards

**Preferred environment:** isolated nonproduction GPU node pool with a working device plugin, approved immutable CUDA/PyTorch image, non-sensitive sample model, observability access and cost approval.

**Read-only alternative:** if no disposable GPU is available, run Sections 4–9 on an authorized environment, classify the supplied fault scenarios and produce a dry-run recovery plan. Mark runtime execution **Not executed** rather than claiming a pass.

**Preflight [SAFE-READ]:**

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get namespaces
kubectl get nodes -L node-pool,gpu,gpu_type
```

Record environment, namespace, TrueFoundry workspace/deployment, cluster owner, escalation contacts, maintenance window, cloud GPU quota and billing owner. Never create debug Pods with privileged mode or host mounts to bypass controls.

## 4. Establish baseline [SAFE-READ]

### 4.1 Hardware and node resource advertisement

```bash
kubectl describe node <gpu-node>
kubectl get node <gpu-node> -o jsonpath='{.status.capacity.nvidia\.com/gpu}{" capacity\n"}{.status.allocatable.nvidia\.com/gpu}{" allocatable\n"}'
kubectl get node <gpu-node> -o jsonpath='{.spec.unschedulable}{" unschedulable\n"}{.spec.taints}{" taints\n"}'
```

Collect node Ready condition, GPU model, driver version (if authorized), GPU capacity/allocatable and **Allocated resources** from `kubectl describe node`. Allocatable is **not** remaining GPU capacity; assigned Pod GPU requests consume it regardless of instantaneous GPU utilization.

### 4.2 Device plugin, GPU Operator and monitoring

```bash
kubectl get daemonsets -A
kubectl get pods -A -o wide
kubectl get services -A
kubectl -n <gpu-system-namespace> describe pod <device-plugin-pod>
kubectl -n <gpu-system-namespace> logs <device-plugin-pod> --tail=150
```

Discover actual component names first. Where GPU Operator is installed, inspect its configured operands and custom resources; otherwise identify standalone plugin or managed node-image ownership. Capture DCGM exporter availability and metric freshness if present.

### 4.3 Workload and inference baseline

```bash
kubectl -n <namespace> get deployments,statefulsets,pods -o wide
kubectl -n <namespace> describe pod <gpu-pod>
kubectl -n <namespace> get events --sort-by=.lastTimestamp
kubectl -n <namespace> logs <gpu-pod> --tail=150
```

Record Pod image digest, GPU limit/request, node, scheduling events, container state, CUDA/vLLM startup evidence, model readiness, inference latency/error baseline and recent change history. Avoid sensitive data in logs and tickets.

## 5. Fault-injection scenarios (tabletop by default)

Run each scenario as a **read-only evidence exercise**. Optional active injection requires a separate approved, disposable environment and explicit rollback. Never create these faults in shared production.

| ID | Injected or observed symptom | First evidence | Expected first failed transition | Owner |
|---|---|---|---|---|
| F1 | GPU Pod Pending: `Insufficient nvidia.com/gpu` | Node allocations, Pod events | GPU capacity → scheduling | GPU capacity / scheduling |
| F2 | Pod selector mismatch or untolerated taint | Node labels/taints, Pod spec | Placement contract → scheduling | Workload / pool policy |
| F3 | GPU host healthy, `nvidia.com/gpu` absent | Driver evidence, plugin logs, Node API | Device plugin → kubelet registration | GPU platform |
| F4 | GPU resource allocated, container GPU unavailable | Pod events, runtime/toolkit state | Allocation → container device exposure | Runtime / node image |
| F5 | `torch.cuda.is_available() == False` | PyTorch build, driver/runtime versions | Container visibility → framework init | ML image / GPU runtime |
| F6 | vLLM: `No CUDA runtime is found` | CUDA checks, vLLM logs | Multi-layer; determine empirically | Based on first failed layer |
| F7 | Model CUDA works, `CUDA out of memory` | VRAM, concurrency, model parameters | CUDA init → model allocation | Model serving |
| F8 | Xid errors, device reset, intermittent inference | Driver logs, DCGM, affected GPU identity | Hardware/driver health | GPU hardware / cloud |
| F9 | DCGM exporter down, inference healthy | Scrape status, exporter logs, inference | Observability pipeline | Monitoring |
| F10 | GPU node scales slowly; requests Pending | Autoscaler events, cloud quota/capacity | Provisioning → node readiness | Node pool / cloud |
| F11 | GPU utilization low but P99 high | Queue, CPU, batch, request tracing | Serving throughput/latency | Inference platform |
| F12 | Unauthorized tenant lands on GPU pool | Taints, selectors, admission, quota | Pool policy / tenancy | Security / platform |

**Evaluation:** each diagnosis must state which layers are proven healthy, which transition fails first, what evidence is still missing, the likely blast radius, the current actionable owner and the smallest approved remediation. For F6, do **not** assert a root cause from the message alone.

## 6. Reusable incident triage decision tree

```text
User-visible inference failure or GPU workload not Ready
  ├─ Pod unscheduled?
  │   ├─ GPU nodes absent → cloud GPU pool provisioning / quotas
  │   ├─ GPU not advertised → driver / plugin registration
  │   ├─ GPU fully allocated → capacity / fragmentation
  │   └─ selector/taint/affinity mismatch → placement policy
  └─ Pod scheduled?
      ├─ container creation fails → runtime/device allocation
      ├─ GPU not visible in container → toolkit/runtime
      ├─ PyTorch CUDA unavailable → image/driver/CUDA compatibility
      ├─ vLLM initialization fails → serving/runtime/model details
      ├─ model OOM → VRAM, model/concurrency configuration
      └─ model healthy, SLO fails → load, batching, queue, CPU/GPU metrics
```

At every branch, compare timestamped node, Pod, application and monitoring evidence. A telemetry gap is not equivalent to a GPU failure.

## 7. Controlled recovery exercise [LAB-MUTATE]

Use the companion lab for a **disposable** GPU Pod smoke test. The default recovery exercise is a tabletop: simulate F1–F12 from provided/observed evidence and write the change plan.

### Recovery plan template

```text
Incident / severity / impact:
First failed transition and supporting evidence:
Affected cluster / node pool / workloads:
Authoritative change source (TrueFoundry / GitOps / IaC / node image):
Proposed minimal change:
GPU quota / capacity / tenant impact:
Maintenance window / approver:
Prechange snapshots and version inventory:
Canary or disposable validation:
Rollout and stop conditions:
Rollback action and required artifacts:
Expected RTO / RPO (only if established by service policy):
End-to-end inference acceptance:
Observability and SLO verification:
```

**Examples of approved remediations:** provision supported GPU capacity, correct workload placement in the authoritative source, restore device-plugin registration via a supported operator/node-image workflow, deploy a compatible immutable PyTorch/vLLM image, or reduce serving concurrency when VRAM pressure is proven. The action must match the observed failure.

**Prohibited shortcuts:** removing taints from shared nodes, force-deleting another tenant's GPU Pods, rebooting nodes or restarting GPU drivers without change approval, and assuming driver upgrades fix all CUDA failures.

## 8. Recovery and rollback gates

| Gate | Required evidence | Fail condition |
|---|---|---|
| G1 — Infrastructure | GPU node Ready; hardware/driver evidence acceptable | Unhealthy GPU or unapproved node |
| G2 — Registration | Expected GPU resource advertised | `nvidia.com/gpu` missing |
| G3 — Placement | PodScheduled true; policy/tenant match | Pending or unauthorized placement |
| G4 — Runtime | Container receives GPU device | Allocation/runtime failure |
| G5 — Framework | CUDA-enabled PyTorch operation succeeds | CUDA init/execution fails |
| G6 — Serving | vLLM/model Ready; test inference succeeds | Startup/inference error |
| G7 — Observability | GPU metrics fresh or gap explicitly accounted for | Silent/unowned telemetry gap |
| G8 — Reliability | Error/latency baseline restored | SLO regression |
| G9 — Cleanup | Disposable Pods deleted, capacity released | Leftover lab workloads |

A failed gate halts promotion. Roll back via the authoritative image, node pool or operator configuration and revalidate earlier gates. Rollback of a driver/node image may require node replacement or downtime.

## 9. Monitoring and alert acceptance

Confirm live DCGM metrics and semantics before defining rules. Useful signals include `DCGM_FI_DEV_GPU_UTIL`, framebuffer memory, temperature and supported Xid/ECC metrics; not all collectors/hardware expose every series. Correlate GPU health with Kubernetes GPU allocation, Pending Pods, vLLM errors, request throughput and P99 latency.

**Alerts should identify:** expected GPU resource disappearance, prolonged GPU Pod scheduling failures, exporter scrape failures, CUDA initialization regression, confirmed GPU hardware events and inference SLO burn. Validate hardware-specific thresholds with vendor guidance; do not invent universal temperature/Xid cutoffs.

## 10. FinOps and operational readiness

Record GPU node-hours, GPU allocations, actual utilization, VRAM, warm capacity, provisioning latency and cost per successful inference request from billing and telemetry. A low utilization reading is not proof a GPU can accept another Pod; GPU requests are scheduling commitments.

Check cloud GPU quota, zone availability, node-image support, operator ownership, workload tenancy, emergency capacity and documented rollback before declaring the GPU platform production-ready.

## 11. Incident report and evidence bundle

```text
Date/time, timezone, cluster context:
TrueFoundry workspace, deployment, namespace:
Incident severity, start/end, customer impact:
GPU SKU / node / driver / runtime:
Device plugin / GPU Operator ownership:
GPU capacity / allocatable / assigned:
Node labels / taints / workload selectors:
Pod events / scheduling outcome:
Host and container GPU visibility:
PyTorch version / torch.version.cuda / CUDA test:
vLLM logs and inference results:
DCGM / Prometheus / Grafana evidence:
First failed transition / lowest proven healthy layer:
Root cause confidence / remaining unknowns:
Actionable owner and escalation:
Approved remediation and rollback:
Recovery gates G1–G9:
Preventive actions / owner / due date:
Lab cleanup / change ticket:
```

## 12. Final Part 04 acceptance

- [ ] GPU node, driver, plugin and toolkit ownership documented.
- [ ] Node resource advertisement and allocated requests distinguished from utilization.
- [ ] GPU placement and tenant isolation verified.
- [ ] CUDA compatibility evidence recorded for immutable workload image.
- [ ] F1–F12 classified with the first failed transition and owner.
- [ ] Approved recovery and rollback plan covers each relevant layer.
- [ ] GPU metrics and inference SLOs correlated.
- [ ] Companion lab completed in a disposable cluster **or** marked as a read-only tabletop with unexecuted checks explicit.
- [ ] No unauthorized production mutation or leftover disposable resources.
- [ ] Formal vendor validation and SRE review tracked separately from publication.

**Companion lab:** [4.7 — GPU Infrastructure Recovery Acceptance Lab](../labs/07-production-gpu-infrastructure-recovery-check.md)

**Part 04 complete for chapter publication:** 7/7 canonical chapters and labs. **Next:** Part 05 — Application Deployment (follow the repository's existing Part 05 layout).

## References

- [Kubernetes — Schedule GPUs](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/)
- [Kubernetes — Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
- [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/)
- [NVIDIA Kubernetes Device Plugin](https://github.com/NVIDIA/k8s-device-plugin)
- [NVIDIA CUDA Compatibility](https://docs.nvidia.com/deploy/cuda-compatibility/)
- [NVIDIA DCGM Exporter](https://github.com/NVIDIA/dcgm-exporter)
- [NVIDIA Xid Errors](https://docs.nvidia.com/deploy/xid-errors/)
- [vLLM Documentation](https://docs.vllm.ai/)
- [TrueFoundry Documentation](https://docs.truefoundry.com/)
