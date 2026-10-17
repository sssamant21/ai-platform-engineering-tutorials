# Chapter 049 — GPU Resource Quotas and Namespace Isolation

**Part 07 — Kubernetes GPU Scheduling and Autoscaling | Chapter 049/100**

> **Status:** Written tutorial published. Live lab execution, independent technical/security review and operational acceptance pending. GPU resource names, admission policies, CNI behavior and quota mechanisms vary by cluster and device-plugin configuration. Do not modify production quotas or tenant isolation without change control.

## Learning objectives

Design GPU quotas and tenant isolation across namespaces; distinguish ResourceQuota, LimitRange, RBAC, Pod Security, NetworkPolicy, workload identity, admission and GPU sharing; enforce fair GPU allocation, protect latency-critical inference, diagnose quota/admission failures, and define cost and recovery runbooks.

## 1. Architecture and trust boundaries

```text
Tenant / CI identity → Kubernetes API authentication
                         │
                      RBAC authorization
                         │
            Admission / Pod Security / policy
                         │
              Namespace ResourceQuota
                         │
             LimitRange / default controls
                         │
            Scheduler: GPU node placement
                         │
              Device plugin GPU allocation
                         │
            CUDA / VRAM / shared GPU hardware
                         │
      NetworkPolicy + IAM + storage boundaries
                         │
        Tenant SLO, GPU-hours and cost reporting
```

**ResourceQuota controls requested resources**, not actual GPU memory or inference tokens. **Namespaces are organizational and policy scopes, not hardware isolation boundaries.**

## 2. GPU quota mechanics

GPU device plugins commonly expose `nvidia.com/gpu` or MIG-specific extended resources. ResourceQuota can constrain extended-resource requests, for example `requests.nvidia.com/gpu`. Verify advertised resource names and admission behavior; do not assume all devices expose the same resource keys.

```bash
kubectl describe node APPROVED_GPU_NODE
kubectl -n APPROVED_NS get resourcequota,limitrange
kubectl -n APPROVED_NS describe resourcequota APPROVED_QUOTA
```

Quota exhaustion may reject Pod creation with `Forbidden` before the Pod reaches Pending scheduling.

## 3. Example ResourceQuota [DESIGN ONLY]

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: gpu-budget
  namespace: approved-staging
spec:
  hard:
    requests.nvidia.com/gpu: "2"
    pods: "12"
    requests.cpu: "16"
    requests.memory: 64Gi
```

Replace namespace and GPU resource name with verified values. This is an illustrative staging design, not an instruction to overwrite existing quotas. The `requests.nvidia.com/gpu` quota works with appropriate GPU resource requests; GPU extended-resource request/limit constraints must be validated on the installed Kubernetes version.

## 4. LimitRange and resource defaults

LimitRange helps enforce CPU/memory defaults and min/max resource bounds. Extended GPU resource behavior differs from ordinary CPU/memory defaults; do not assume LimitRange can cap VRAM or substitute for ResourceQuota. Validate admission using server-side dry-run against an approved staging namespace where policy permits.

## 5. Namespace isolation layers

| Layer | Protects | Does not guarantee |
|---|---|---|
| RBAC | Kubernetes API permissions | runtime data isolation |
| ResourceQuota | aggregate resource requests | GPU VRAM isolation |
| Pod Security | privileged container restrictions | network segmentation |
| NetworkPolicy | supported L3/L4 network flows | L7 model authorization |
| workload identity | cloud API credentials | local GPU isolation |
| storage IAM | model/object access | GPU fairness |
| admission policies | validated Pod attributes | physical device separation |
| MIG | supported hardware partitioning | full tenant security alone |

Defense in depth is essential; no single namespace mechanism provides full multi-tenant isolation.

## 6. RBAC and service accounts

Use dedicated namespace-scoped service accounts for model serving. Restrict create/update on GPU workloads, quota edits, Secrets, node labels, privileged DaemonSets and impersonation. Separate tenant developer roles from platform-admin rights.

## 7. Pod Security

Enforce the organization-approved Pod Security Admission profile or equivalent policy. GPU workloads may need approved runtime integration but should not require arbitrary hostPath mounts, privileged containers or host networking. Driver/operator DaemonSets belong in controlled platform namespaces.

## 8. NetworkPolicy and DNS

Apply namespace default-deny where supported by the CNI, then allow approved gateway ingress, DNS, model artifact endpoints, metrics and necessary control-plane communication. NetworkPolicy enforcement is CNI-dependent. Denying DNS or model storage egress can make inference appear as a model startup failure.

## 9. Workload identity and model artifacts

Use least-privilege IRSA/Pod Identity on EKS, managed/workload identity on AKS and Workload Identity Federation on GKE as appropriate. Bind identity to specific service accounts, restrict object paths and KMS decrypt permissions, and pin model/tokenizer digests. Never mount long-lived cloud credentials into Pods.

## 10. GPU sharing risks

A Pod with one `nvidia.com/gpu` request may have exclusive device allocation under a standard plugin setup, but time-slicing and other sharing modes change the model. Kubernetes GPU quotas count advertised resources, not guaranteed VRAM. MIG isolation depends on hardware, partition configuration and supported profiles. Require explicit sharing policy and noisy-neighbor testing.

## 11. Multi-tenant inference controls

Beyond Kubernetes resource quotas, enforce per-tenant requests/second, active sequences, input/output token budgets, queue caps, max context length, streaming deadlines and authentication. Protect GPU goodput and prevent one tenant from exhausting KV cache or monopolizing model runtime concurrency.

## 12. Priority and fairness

Use PriorityClasses carefully for production inference versus batch jobs, combined with quotas and admission. Preemption can disrupt long-lived streams and does not create new GPU hardware. Fair scheduling across tenants may require gateway-level token/concurrency quotas, not merely Pod quotas.

## 13. Hierarchical governance

Namespaces are independent ResourceQuota scopes. Organizations may use additional policy engines, fleet admission or quota orchestration for hierarchical team budgets. Do not assume Kubernetes ResourceQuota automatically enforces organization-wide parent quotas across namespaces.

## 14. FinOps and chargeback

Track GPU requested versus allocated, node idle time, MIG/time-sliced allocation, GPU-hours, cost per valid completion and tenant request/token consumption. GPU-hour attribution is approximate when devices are shared; document the allocation method and overhead.

## 15. Safe-read diagnostics

```bash
date -u
kubectl config current-context
kubectl get namespaces
kubectl get resourcequota -A
kubectl get limitrange -A
kubectl get networkpolicy -A
kubectl get serviceaccounts -A
kubectl get events -A --sort-by=.lastTimestamp | tail -100
kubectl -n APPROVED_NS describe resourcequota APPROVED_QUOTA
kubectl -n APPROVED_NS get pods -o wide
```

RBAC errors are evidence of access restrictions, not permission to escalate.

## 16. Troubleshooting decision tree

```text
GPU workload unavailable?
 ├─ API Forbidden → quota/RBAC/admission
 ├─ Pod Pending → GPU allocatable/taints/affinity
 ├─ image/model startup → IAM/KMS/NetworkPolicy
 ├─ CUDA OOM → VRAM/KV/sharing
 ├─ tenant 429 → gateway token/concurrency quota
 ├─ cross-tenant access → authn/authz/storage policy
 ├─ GPU cost spike → orphaned Pods/nodes/chargeback
 └─ rollback drift → quota/policy/GitOps reconciliation
```

## 17. Twenty-four production failure scenarios

| ID | Failure | Evidence |
|---|---|---|
| F01 | no GPU quota configured | quota inventory |
| F02 | quota uses wrong GPU resource key | admission |
| F03 | quota exhausted | Forbidden |
| F04 | CPU/memory quota blocks GPU Pod | admission |
| F05 | LimitRange defaults conflict | policy |
| F06 | RBAC grants quota modification | role audit |
| F07 | tenant can create privileged Pods | PSA |
| F08 | service account overprivileged | RBAC |
| F09 | NetworkPolicy blocks DNS | CNI flows |
| F10 | NetworkPolicy blocks model storage | startup |
| F11 | workload identity denied | IAM audit |
| F12 | storage IAM cross-tenant access | object audit |
| F13 | untrusted image admitted | supply chain |
| F14 | GPU device shared unexpectedly | plugin config |
| F15 | time-slicing noisy neighbor | latency/VRAM |
| F16 | MIG profile unavailable | allocatable |
| F17 | tenant monopolizes KV cache | runtime metrics |
| F18 | token quota absent | gateway |
| F19 | PriorityClass causes eviction | events |
| F20 | cross-namespace service exposed | auth audit |
| F21 | telemetry leaks tenant/prompt data | privacy review |
| F22 | chargeback misattributes shared GPU | FinOps |
| F23 | rollback leaves quota/policy drift | GitOps diff |
| F24 | orphaned GPU workload burns budget | cost audit |

## 18. Sixteen operational runbooks

1. Inventory namespaces, GPU quotas and ownership.
2. Validate extended-resource quota keys.
3. Recover ResourceQuota admission denial.
4. Resolve CPU/memory and LimitRange conflicts.
5. Audit RBAC and service account privilege.
6. Restore Pod Security and admission controls.
7. Recover NetworkPolicy DNS and model egress.
8. Restore workload identity and KMS permissions.
9. Recover tenant-specific artifact authorization.
10. Audit GPU sharing and MIG allocation.
11. Contain time-slicing/noisy-neighbor incidents.
12. Recover runtime token/KV-cache fairness.
13. Tune priority/preemption and streaming drain.
14. Restore sanitized telemetry and privacy controls.
15. Reconcile tenant GPU cost and orphaned workloads.
16. Execute complete quota/policy rollback and RCA.

## 19. Eight tabletop game days

1. ResourceQuota rejects critical inference rollout.
2. Tenant obtains permission to modify its GPU quota.
3. NetworkPolicy default deny blocks model storage.
4. Workload identity allows cross-tenant artifact access.
5. Time-sliced tenant causes VRAM contention.
6. Long-context requests monopolize KV cache.
7. Priority/preemption interrupts active streams.
8. Rollback restores app but not tenant quotas/policies.

Tabletop by default; live injection requires approved isolated staging.

## 20. Twenty acceptance gates

G01 scope/versions; G02 namespace ownership; G03 GPU resource key; G04 ResourceQuota; G05 CPU/memory/LimitRange; G06 RBAC; G07 Pod Security; G08 NetworkPolicy; G09 workload identity; G10 storage IAM; G11 image integrity; G12 GPU sharing/MIG; G13 tenant token quotas; G14 fairness/priority; G15 SLO/terminal validity; G16 privacy/telemetry; G17 FinOps/chargeback; G18 full rollback; G19 game-day evidence; G20 independent review.

See [companion lab](../labs/049-gpu-resource-quotas-namespace-isolation-check.md). All gates remain pending until independently evidenced.

**Next: Chapter 050 — MIG, GPU Sharing and Multi-Tenant Risks.**
