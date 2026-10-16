# Chapter 004 — Hands-On Lab Environment and Safety Controls

**Part 01 — AI/LLM SRE Fundamentals | Chapter 004 of 100**

> Publication means the written material exists, not that any environment was provisioned, tested or accepted. Commands must be reviewed against your actual cloud, Kubernetes version and organizational policies.

## Learning outcomes

Design a reproducible, cost-bounded, isolated LLM inference SRE laboratory; verify Kubernetes and GPU prerequisites without making changes; choose safe execution tiers; enforce IAM, network, data and budget controls; and document cleanup, evidence and stop conditions.

## 1. Lab topology and execution tiers

```text
Developer workstation / approved CI runner
   ├── git + kubectl + cloud CLI + helm (version pinned)
   ├── read-only context (production discovery, if authorized)
   └── dedicated staging context
          ├── namespace llmops-sre-lab
          ├── service account + scoped RBAC + quota
          ├── gateway / synthetic client (no real prompts)
          ├── optional CPU mock inference service
          ├── optional approved GPU pool + inference runtime
          ├── telemetry: metrics/logs/traces/DCGM if available
          └── object store: approved synthetic model artifacts
```

**Tier A — [SAFE-READ]:** inspect approved cluster, quotas, nodes, Pods and dashboards; no creation or changes.

**Tier B — [LAB-MUTATE]:** create a dedicated isolated staging namespace and CPU-only mock workload after explicit environment authorization; use a time limit and cleanup.

**Tier C — [GPU-LAB]:** schedule a GPU workload only after confirming node availability, quota, budget, image/model licensing and compatibility. GPU allocation can incur significant costs even when idle.

**Tier D — [CHANGE-CONTROL]:** production, shared GPU pools, network/security changes, node replacement, chaos injection and failover require change approval. These are not prerequisites for this chapter.

## 2. Required approvals and preflight

Record: owner, business purpose, UTC window, cloud account/subscription/project, cluster and context, allowed namespaces, allowed resource types, approved registry, approved models, maximum spend, TTL, reviewer, escalation and rollback owner.

Never assume a Kubernetes context name proves environment identity. Confirm cluster API server and cloud account via approved mechanisms, without publishing sensitive values.

```bash
kubectl version --client
kubectl config current-context
kubectl cluster-info
kubectl auth can-i get pods --all-namespaces
kubectl get nodes -o wide
kubectl get namespaces
kubectl get resourcequota,limitrange -A
kubectl get events -A --sort-by=.lastTimestamp
```

The `auth can-i` check is advisory: policy admission, cloud IAM and other constraints may still block an operation.

## 3. Workstation and toolchain baseline

Pin and record versions of Git, kubectl, Helm, cloud CLI, container runtime, Python, load generator and model runtime. For Windows, PowerShell and WSL2 are supported workflows but command examples in this chapter use Bash; run them in an appropriate shell.

```bash
git --version
kubectl version --client
helm version --short
python3 --version
docker version
```

A Docker client version alone does not prove a running daemon. Use `docker info` in an approved local environment when container-based labs are planned. Never store kubeconfig, tokens or cloud credentials in the repository.

## 4. GPU compatibility and scheduling checklist

Before provisioning a GPU workload, verify:
1. Node pool, GPU SKU, cloud quota and availability zone.
2. NVIDIA driver, container runtime, device plugin/GPU Operator and compatible CUDA/runtime matrix.
3. `nvidia.com/gpu` allocatable and Pod resource requests.
4. Taints, tolerations, node selectors, affinity and topology constraints.
5. Model weights, precision, maximum context, KV-cache and runtime overhead fit GPU memory.
6. GPU utilization/VRAM/temperature/XID/ECC telemetry if available.
7. Rollout surge capacity, warmup, cost ceiling and abort thresholds.

```bash
kubectl get nodes -L topology.kubernetes.io/zone -o wide
kubectl describe nodes
kubectl get daemonsets -A
kubectl get resourcequota -A
```

Do not deploy privileged GPU diagnostic containers or install a device plugin without authorization. Node descriptions may contain sensitive operational details: sanitize evidence.

## 5. Isolation and security controls

A namespace is not a security boundary by itself. Use scoped RBAC, admission policies, restricted Pod security, workload identity, network policies with a compatible CNI, image allowlists, signed/pinned artifacts, secrets management, egress controls and per-namespace quotas. Validate tenant isolation from both network and application layers.

Do not use public model downloads unless licensing, network egress, provenance, scanning and cost have been approved. Do not use PHI, PII, secrets or production prompts. Use synthetic non-sensitive requests and redact logs/traces.

### Example namespace and quota (review-only until approved)

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: llmops-sre-lab
  labels:
    app.kubernetes.io/part-of: llmops-sre
    purpose: isolated-training
    pod-security.kubernetes.io/enforce: restricted
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: llmops-sre-budget
  namespace: llmops-sre-lab
spec:
  hard:
    requests.cpu: "2"
    requests.memory: 4Gi
    limits.cpu: "4"
    limits.memory: 8Gi
    pods: "6"
    services: "2"
```

This quota is a CPU-only example; it does not set GPU quota or cloud spend limits. Namespace labels and quota can conflict with existing admission policy and must be reviewed before application.

## 6. Cost and time-to-live controls

Set a hard lab window, idle shutdown policy, quota, GPU instance budget, model artifact storage/egress budget and explicit cleanup owner. Cloud billing alerts are not always real-time hard stops; implement cloud quotas or external guardrails where available.

```text
Approved max runtime (hours):
GPU hourly price estimate:
Number of GPU nodes / GPUs:
Storage + network egress estimate:
Estimated max cost:
Cloud quota:
Budget alert owner:
TTL / cleanup deadline:
Abort trigger:
```

For example only, 2 GPU instances × $3/hour × 4 hours = $24 in compute before storage/egress. This is illustrative arithmetic, not a live price quote.

## 7. Evidence and reproducibility

Record cluster/context identifier in a secure internal location, approved environment, UTC timestamps, Git commit SHA, Kubernetes/tool versions, image digest, model checksum, runtime version, sanitized outputs, expected/actual behavior, errors, cost and reviewer. Never copy full kubeconfig, environment variables, credentials or sensitive Pod logs into Git.

Evidence should distinguish **planned**, **executed**, **failed**, **blocked** and **accepted**. A successful GitHub commit is not proof a lab was executed.

## 8. Safe cleanup and rollback

Before any creation, identify the exact resources to be created and how they will be deleted. Use owner labels and avoid broad wildcard deletion. Never run `kubectl delete namespace` against an unverified context; namespace deletion can cascade to all its resources.

Proposed cleanup procedure (requires approved staging context):
1. Stop load generator and inference traffic.
2. Verify namespace ownership and list resources.
3. Delete only approved lab resources using explicit names/manifests.
4. Verify Pods, Services, PVCs, load balancers, cloud disks and GPU nodes are removed or scaled down as intended.
5. Check lingering cloud charges and reconcile evidence.

## 9. Sixteen common setup failures

| ID | Symptom | Diagnosis |
|---|---|---|
| F01 | Wrong context | verify account/cluster identity |
| F02 | Expired cloud SSO | refresh via approved workflow |
| F03 | RBAC forbidden | inspect role and scope |
| F04 | Admission denied | inspect policy and Pod security |
| F05 | Namespace quota exceeded | check ResourceQuota |
| F06 | CPU/memory Pending | inspect requests and allocatable |
| F07 | GPU Pending | check GPU plugin, quota and taints |
| F08 | No CUDA runtime | verify driver/toolkit compatibility |
| F09 | CUDA OOM | size model/KV cache and concurrency |
| F10 | ImagePullBackOff | registry auth/digest/network |
| F11 | Model download 403 | workload identity and artifact ACL |
| F12 | DNS/service failure | CNI, CoreDNS, EndpointSlice |
| F13 | No metrics | scrape/exporter permissions |
| F14 | Telemetry contains secrets | redact, rotate and incident review |
| F15 | Unexpected GPU spend | stop workloads and reconcile billing |
| F16 | Cleanup leaves disks/LBs | inventory external resources |

## 10. Twelve operational runbooks

1. Verify identity, account and Kubernetes context.
2. Validate tool versions and local runtime.
3. Confirm staging authorization and approvals.
4. Check RBAC, admission and quotas.
5. Verify network and artifact egress.
6. Check GPU scheduling prerequisites.
7. Investigate CUDA/driver mismatch.
8. Recover model/image pull errors.
9. Establish synthetic-only test traffic.
10. Verify telemetry and privacy controls.
11. Respond to budget or TTL overrun.
12. Clean up and verify cloud resources.

## 11. Six game days

1. Wrong-context prevention (tabletop).
2. Unauthorized namespace creation blocked (tabletop).
3. GPU Pending due to quota/taint (tabletop).
4. CUDA mismatch and recovery decision (tabletop).
5. Sensitive log exposure and redaction (tabletop).
6. GPU budget overrun and cleanup (tabletop).

## 12. Twelve acceptance gates

G01 approval and owner; G02 context verification; G03 tool version baseline; G04 least-privilege RBAC; G05 admission/network controls; G06 synthetic data; G07 GPU compatibility checklist; G08 quota and budget; G09 telemetry/privacy; G10 cleanup/rollback; G11 failure tabletop; G12 reviewer signoff.

All gates are **pending** until real evidence is collected. See the [Chapter 004 companion lab](../labs/004-hands-on-lab-environment-safety-controls-check.md).

**Next: Chapter 005 — Fundamentals Acceptance Lab and Incident Walkthrough.**
