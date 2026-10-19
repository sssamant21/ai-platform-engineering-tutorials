# Chapter 034 — Kubernetes GPU Platform Capstone

**AI Infrastructure Engineering & SRE | Part 04 — Kubernetes GPU Administration | Chapter 034/100**

**Publication:** Documentation and offline lab published. Live GPU staging, runtime execution, security review, and production acceptance are pending.

## Mission and learning outcomes

Build, validate and recover a **secure, observable, resilient Kubernetes GPU platform** integrating Chapters 026–033: NVIDIA device discovery, GPU Operator, scheduling, tenant quotas, runtime security, autoscaling, workload rollouts, and troubleshooting. Produce a repeatable evidence package and a scored production-readiness decision. This capstone targets Kubernetes/EKS and AKS where applicable; Karpenter/EC2NodeClass applies specifically to supported AWS EKS environments.

## 1. Enterprise reference architecture

```text
CI/GitOps: signed immutable image + IaC + approved changes
     |
Kubernetes API: RBAC -> PSA/admission -> ResourceQuota
     |
scheduler: GPU extended resource + topology + taints/affinity
     |
GPU node pool: cloud quota + N-1 + disruption budget
     |                        |
kubelet/device manager      Karpenter or supported cloud autoscaler
     |
NVIDIA GPU Operator -> driver/toolkit/device plugin/MIG/DCGM
     |
CRI/containerd + CDI -> CUDA/NCCL runtime -> inference/training
     |                                    |
GPU UUID, Xid/ECC, DCGM                app traces/SLO/checkpoints
     |                                    |
Prometheus/Grafana/OpenTelemetry -> alerting -> incident runbooks
     |
Evidence, RCA/COA, cost allocation and go/no-go decision
```

**Critical trust boundaries:** Operator driver/toolkit operands can be privileged platform infrastructure; tenant GPU pods should be least privilege. Namespace quota does not guarantee physical GPU availability or isolation. Time-slicing is not equivalent to MIG hardware partitioning. A Ready node, an allocatable GPU, a successful CUDA canary and a healthy model are **four separate checks**.

## 2. Capstone environment and prerequisites

- Dedicated, approved staging GPU node pool with known GPU SKU, OS/kernel, driver, container runtime, device plugin and Operator versions.
- Kubernetes access with least-privilege read permissions and authorized, disposable staging namespace write permissions.
- Approved immutable CUDA and application images; signed/provenance-checked artifacts where supported.
- Separate durable checkpoint/object storage, test datasets, no production secrets or PHI.
- Prometheus/DCGM metrics, node/pod events, GPU UUID/Xid/ECC, application p95/p99 and checkpoint telemetry.
- N-1 capacity or an explicitly documented nonproduction exception. No intentional hardware fault injection on shared production GPU nodes.
- Cloud-specific: EKS node image/Karpenter/EC2NodeClass or managed node group; AKS GPU node pool and driver ownership.

## 3. Architecture decisions and ownership

| Decision | Evidence | Owner |
|---|---|---|
| GPU Operator vs managed driver | installed driver/toolkit, compatibility matrix | Platform |
| Full GPU vs MIG vs time-slicing | GPU SKU, resource keys, isolation review | GPU platform/security |
| Tenant quotas and RBAC | namespace ResourceQuota, PSA, policies | Platform/security |
| Scheduling and topology | affinity, taints, GPU allocatable, zones | Platform |
| Scale-out and scale-in | node pool constraints, quota, checkpoint | Cloud/platform |
| Inference release strategy | surge GPU capacity, readiness, SLO | App/SRE |
| Training recovery | durable restore, rank coordination, RPO/RTO | ML/SRE |
| Telemetry and escalation | DCGM, logs, traces, alert owners | Observability/on-call |

Record an ADR for each tradeoff, its rollback path and a measurable acceptance threshold.

## 4. Safe read-only baseline

```bash
kubectl config current-context
kubectl version
kubectl get nodes -o wide
kubectl get nodes -o json | jq -r '.items[] | [.metadata.name, (.status.capacity["nvidia.com/gpu"] // "none"), (.status.allocatable["nvidia.com/gpu"] // "none")] | @tsv'
kubectl get resourcequota,limitrange -A
kubectl get pods -A -o wide
kubectl get pdb -A
kubectl get runtimeclasses
kubectl get events -A --sort-by=.lastTimestamp
kubectl get daemonsets -A | grep -Ei 'nvidia|gpu'
kubectl get clusterpolicies.nvidia.com
```

Some CRDs are optional. If `clusterpolicies.nvidia.com` is absent, determine whether the deployment intentionally uses a managed driver/device plugin. For EKS with Karpenter, additionally inspect `kubectl get nodepools,ec2nodeclasses` only when CRDs are installed. On authorized GPU hosts use `nvidia-smi -L`, driver version, Xid/ECC kernel logs and toolkit/CDI inspection.

**Never** paste full secret-bearing manifests or unrestricted environment variables into incident tickets.

## 5. Staging implementation workstreams

### Workstream A — GPU discovery and Operator

Verify physical GPU UUIDs, GPU health, driver and CUDA compatibility, Operator ClusterPolicy reconciliation (if installed), toolkit and device-plugin registration. Compare node allocatable resource keys with actual GPU/MIG strategy. Establish who owns driver lifecycle on EKS/AKS; avoid double-management.

### Workstream B — Secure tenancy

Create a disposable namespace through approved IaC with restricted Pod Security Admission, least-privilege RBAC, ResourceQuota for `requests.nvidia.com/gpu` or actual MIG resource key, CPU/memory controls, network policy and service account hardening. Verify a quota-exceeding request is rejected; verify a within-quota request can still be Pending due to unavailable physical GPU capacity.

### Workstream C — GPU scheduling and scaling

Verify taints/tolerations, affinity, node-pool restrictions, GPU resource requests, zone/topology and N-1 capacity. On EKS with Karpenter, validate installed NodePool and EC2NodeClass schemas, cloud service quotas, subnet IPs, EC2 capacity, GPU bootstrap readiness and interruption behavior. On AKS use supported node-pool autoscaling controls. Record Pending-to-CUDA-ready latency.

### Workstream D — Runtime security and CUDA/NCCL

Deploy only an approved, pinned image with non-root execution, no privilege escalation, dropped capabilities, RuntimeDefault seccomp and no host device mounts. Test CDI/device allocation and a CUDA smoke test; validate NCCL only when multi-GPU or multi-node communication is in scope. Verify actual Pod Security behavior through staging admission.

### Workstream E — Rollout, disruption and recovery

Canary a GPU inference Deployment with realistic startup/readiness probes and measured GPU surge capacity. Exercise approved drain with PDB and workload checkpoint, then verify rollback and restore. For distributed training, coordinate rank recovery; do not assume Deployment rolling updates are safe for collectives. Demonstrate RPO/RTO from a real disposable checkpoint.

### Workstream F — Observability and FinOps

Publish DCGM GPU health/utilization/memory metrics, node readiness, plugin health, pod Pending age, application p95/p99, checkpoint age and cost per GPU-hour. Confirm Xid/ECC alerts, dashboard ownership, SLO error-budget policy, idle GPU attribution and resource quotas. Redact tenant/process secrets.

## 6. Illustrative secure canary template

**Review before use.** This is not deployable until the image digest, service endpoint, namespace, resource key, topology and security policy are customized and validated.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-capstone-canary
  namespace: approved-gpu-staging
spec:
  restartPolicy: Never
  automountServiceAccountToken: false
  securityContext:
    runAsNonRoot: true
    runAsUser: 10001
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: cuda
      image: registry.example.invalid/cuda-canary@sha256:REPLACE_WITH_APPROVED_DIGEST
      command: ["sh", "-c", "nvidia-smi -L"]
      securityContext:
        privileged: false
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
      resources:
        requests:
          cpu: 250m
          memory: 512Mi
          nvidia.com/gpu: 1
        limits:
          memory: 1Gi
          nvidia.com/gpu: 1
```

Run a **server-side dry-run** in the approved staging namespace before applying. The CUDA image must actually support a non-root UID and read-only filesystem. GPU canary success does not replace application SLO or tenant isolation testing.

## 7. Fifteen capstone acceptance scenarios

1. Driver healthy, GPU resources missing: device-plugin registration.
2. Node Ready, GPU Operator ClusterPolicy Degraded.
3. CDI/containerd mismatch yields CreateContainerError.
4. MIG profile mismatch strands pods.
5. Time-slicing exceeds isolation requirements.
6. Quota admits request, no physical GPU capacity.
7. Tenant bypasses node-pool intent with unauthorized toleration.
8. GPU pod fails restricted Pod Security Admission.
9. Karpenter/managed node pool scales out, driver never becomes ready.
10. N-1 loss of largest GPU node or AZ leaves insufficient capacity.
11. GPU inference canary causes p99 latency regression.
12. PDB blocks planned drain.
13. Spot interruption races checkpoint completion.
14. GPU Xid/ECC causes CUDA/NCCL failure.
15. Telemetry, rollback or cost guardrail is absent.

For each: classify detection, scope, evidence, containment, mitigation, rollback, application validation and RCA/COA. Use synthetic injection or approved disposable staging resources.

## 8. Ten incident game days

| # | Scenario | Required recovery proof |
|---|---|---|
| 1 | Device plugin registration loss | GPU allocatable and CUDA restored |
| 2 | Operator/driver version regression | compatible driver and node canary |
| 3 | CDI/runtime failure | container GPU allocation restored |
| 4 | MIG partition drift | resource keys and isolation restored |
| 5 | GPU tenant quota breach | admission rejects excess request |
| 6 | GPU node pool capacity exhaustion | queue/capacity policy verified |
| 7 | Spot interruption | checkpoint resume and RPO |
| 8 | Inference rollout regression | p99/error rate restored |
| 9 | DCGM telemetry outage | alerts and freshness restored |
| 10 | N-1 node/zone failure | recovery, SLO and rollback |

## 9. Eight integrated production runbooks

**RB-034-A — Baseline inventory:** capture cluster, driver, GPU UUID, Operator/plugin, runtime, tenant and telemetry versions.

**RB-034-B — GPU missing:** trace hardware -> driver -> plugin -> allocatable -> scheduler.

**RB-034-C — CUDA/CDI:** inspect device allocation, toolkit/CRI, CUDA version and runtime logs.

**RB-034-D — Tenant incident:** enforce RBAC/PSA/quota, inspect sharing/isolation and contain exposure.

**RB-034-E — Scaling failure:** inspect GPU supply, Karpenter/managed node pool, cloud quotas, zones and startup.

**RB-034-F — Safe rollout/drain:** verify PDB, GPU surge, checkpoint, N-1, change approval and rollback.

**RB-034-G — Training recovery:** restore durable checkpoint, restart ranks and verify NCCL progress/RPO/RTO.

**RB-034-H — Incident closure:** prove SLO, GPU health, observability, FinOps and owner acceptance.

## 10. Automated lab and evidence pack

[Lab 034 — Kubernetes GPU Platform Capstone Gate](../labs/034-kubernetes-gpu-platform-capstone-check.md) supplies an offline Python 3.11+ validator, negative tests and a staging evidence checklist. It fails closed on security, GPU readiness, quota, N-1, checkpoint, SLO, telemetry and rollback evidence. **A passing synthetic gate does not establish production readiness.**

Evidence folder convention:

```text
evidence/034/
  architecture-and-adr.md
  cluster-gpu-inventory-redacted.md
  operator-runtime-compatibility.md
  tenant-security-and-quotas.md
  gpu-canary-and-slo.md
  checkpoint-rpo-rto.md
  n-minus-one-and-disruption.md
  dcgm-alerts-and-finops.md
  incident-game-days.md
  acceptance-signoff.md
```

Store only approved, redacted evidence in the repository; keep sensitive cluster inventory and secrets in restricted systems.

## 11. Final 100-point production-readiness assessment

| Domain | Points |
|---|---:|
| GPU discovery, Operator, driver and CDI | 20 |
| Scheduling, quotas and tenant security | 20 |
| Autoscaling, N-1 and disruption recovery | 20 |
| Workload correctness, SLO, checkpoints and telemetry | 20 |
| Automated tests, incident game days, runbooks and signoff | 20 |
| **Total** | **100** |

**Pass threshold:** >=90 points **and** no critical unresolved GPU correctness, tenant security, data loss, capacity, telemetry or rollback blocker. Scoring below 90, missing evidence, or an unverified critical control is **no-go**. Live staging acceptance and independent security review are mandatory for a real go-live decision.

## References

- [Kubernetes Device Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)
- [Kubernetes Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
- [Kubernetes Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/)
- [Kubernetes Disruptions](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)
- [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/)
- [NVIDIA Kubernetes Device Plugin](https://github.com/NVIDIA/k8s-device-plugin)
- [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/)
- [Karpenter](https://karpenter.sh/docs/)

**Part 04 complete by publication. Next: Chapter 035 — AWS GPU EC2 Architecture and Quotas (Part 05 — Cloud GPU Infrastructure).**
