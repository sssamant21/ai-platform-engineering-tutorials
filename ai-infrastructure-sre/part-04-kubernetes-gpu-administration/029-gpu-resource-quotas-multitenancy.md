# Chapter 029 — GPU Resource Quotas and Multitenancy

**Part 04 — Kubernetes GPU Administration · Chapter 029/100**

**Status:** Published documentation; executable lab validation, real cluster tests, security review and production acceptance are pending.

## Objectives

Design GPU tenant boundaries using namespaces, ResourceQuota, LimitRange, RBAC, Pod Security Admission, scheduling isolation, GPU Operator/device-plugin policy, MIG, time-slicing, capacity reservations and cost attribution. Troubleshoot quota admission errors, starvation, noisy neighbors and tenant isolation violations. Implement a fail-closed synthetic policy gate and demonstrate staging acceptance.

## 1. Reference architecture

```text
Platform team: GPU inventory / driver / operator / DCGM
                |
       Admission: RBAC + PSA + policy
                |
      Namespace tenant-a | tenant-b
      ResourceQuota      | ResourceQuota
      LimitRange         | LimitRange
      NetworkPolicy      | NetworkPolicy
                |
      scheduler + PriorityClass + taints/affinity
                |
      GPU node pools (full GPU / MIG / shared GPU)
                |
      kubelet + NVIDIA device plugin + CDI/runtime
                |
      workload identity, telemetry, SLO, chargeback
```

**Boundary warning:** a namespace and ResourceQuota govern API admission and accounting; they do **not** provide physical GPU memory isolation. MIG provides hardware partitioning on supported GPUs but is not equivalent to a complete security boundary for all threat models. Time-slicing and MPS share resources and require additional trust and contention controls. Security claims require vendor/platform-specific verification.

## 2. Kubernetes extended resource quotas

GPUs are exposed as vendor-defined integer extended resources such as `nvidia.com/gpu`. With MIG mixed strategy, profile-specific keys may be exposed, e.g. `nvidia.com/mig-1g.10gb` on supported hardware. Actual keys depend on GPU generation, MIG profiles and device-plugin configuration.

ResourceQuota example, for a dedicated staging namespace:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-gpu-quota
  namespace: tenant-a-staging
spec:
  hard:
    requests.nvidia.com/gpu: "4"
    requests.cpu: "16"
    requests.memory: 64Gi
    limits.memory: 96Gi
    pods: "20"
```

For extended resources, the quota key is `requests.<resource-name>`; confirm support for the cluster version and exposed resource. ResourceQuota is an **admission-time** control: it does not guarantee the namespace can obtain the quota on a particular node or zone. ResourceQuota is not a fair-share scheduler and does not reserve GPU capacity. Limits for GPUs are integer quantities; if both request and limit are set, they must match. Quota configuration must reflect the actual device-plugin sharing model.

Inspect:

```bash
kubectl config current-context
kubectl get resourcequotas -A
kubectl describe resourcequota tenant-gpu-quota -n tenant-a-staging
kubectl get limitranges -A
kubectl get pods -n tenant-a-staging -o wide
kubectl get events -n tenant-a-staging --sort-by=.lastTimestamp
```

**LimitRange:** useful for CPU/memory defaults and maximums, but do not assume it automatically creates meaningful GPU defaults for all vendor extended resources. Prefer explicit GPU requests/limits and validating admission policies.

## 3. Scheduling and capacity isolation

A tenant GPU request passes several independent gates: namespace quota; pod security and policy; node selector/affinity; taints/tolerations; node allocatable GPU count; CPU/memory/PVC/network constraints; device-plugin Allocate; CUDA runtime; application health.

Use separate node pools for strong workload placement boundaries where appropriate. Assign verified labels (e.g. `accelerator-pool=tenant-a`), protected taints, and admission enforcement so tenants cannot simply add tolerations for other pools. Avoid granting tenants node label, taint, ClusterRole or GPU Operator configuration permissions.

**N-1 capacity:** plan for loss of the largest relevant node/failure domain, not just average usage. Quota sums may exceed available physical capacity as a deliberate oversubscription policy, but do not represent guaranteed availability. For guaranteed service levels, use separate reservations/admission or dedicated capacity with documented ownership.

## 4. MIG, time-slicing and MPS

| Mode | Scheduling/accounting | Isolation / risk |
|---|---|---|
| Full GPU | typically integer `nvidia.com/gpu` | device-level allocation; shared host still matters |
| MIG | profile-specific or strategy-dependent keys | hardware partitions on supported devices; validate supported isolation guarantees |
| Time-slicing | multiple advertised logical shares may map to same GPU | shared memory/compute contention; not MIG-style isolation |
| MPS | supported cooperative process sharing | workload trust and fairness need explicit review |

Changing MIG or sharing configuration can alter node allocatable resources and interrupt running workloads. Treat as a coordinated change with checkpoints, drain, rollback and tenant approval.

## 5. Identity and security boundaries

Use least-privilege namespace-scoped RBAC, restricted Pod Security Admission, trusted image digests, service account controls, network policies and secrets isolation. Privileged driver/toolkit/device-plugin DaemonSets are platform infrastructure, **not** a permission model for tenant pods. Avoid exposing host device files, hostPath mounts, host PID/network, arbitrary runtime classes or broad node permissions to tenants.

Where GPU metrics include process identity, model names or request information, apply access controls and retention. Separate per-tenant dashboards and audit quota overrides.

## 6. Fairness, priorities and noisy neighbors

Quota constrains admitted requests; it does not prevent a tenant from monopolizing GPU compute within shared allocations. Monitor GPU SM utilization, HBM usage, memory pressure, job queue age, inference p99, training step time, preemptions, GPU UUID and tenant ownership. Apply scheduling policy, workload concurrency limits, priority classes and dedicated capacity based on measured SLOs. Preemption may disrupt uncheckpointed training jobs; define checkpoint and grace-period expectations.

## 7. Capacity and cost model

For each tenant, track requested GPU-hours, allocated GPU-hours, actual busy time, MIG profile-hours, queued hours, GPU SKU cost, node idle cost and shared overhead. Do not equate GPU utilization percentage with chargeback allocation unless the billing model explicitly does so. Chargeback for shared GPU requires a documented allocation policy, not merely node-level DCGM counters.

Example planning equation:

```text
effective tenant admission <= min(
  namespace quota remaining,
  eligible node allocatable remaining,
  pool reservation remaining,
  failure-domain / N-1 budget
)
```

This is a policy-level bound, not a Kubernetes scheduler formula.

## 8. Twelve troubleshooting cases

| Symptom | Evidence | Safe action |
|---|---|---|
| Pod creation Forbidden | ResourceQuota status | inspect requested resource key and hard/used |
| Pod Pending despite quota | scheduler events | check node allocatable and affinity |
| wrong MIG quota key | plugin node resources | correct resource name through change |
| quota used stays high | terminating pods / API state | inspect lifecycle and controllers |
| noisy neighbor on shared GPU | per-tenant latency/usage | isolate workload or reduce sharing |
| GPU OOM in tenant | GPU memory/MIG profile | right-size, do not confuse quota with VRAM |
| cross-tenant placement | labels/taints/admission | quarantine and fix policy |
| unauthorized privileged pod | PSA/RBAC/audit | contain and revoke access |
| PriorityClass starvation | queue, preemption | review fairness and checkpoint policy |
| tenant SLO breach | DCGM/app traces | correlate contention and input load |
| quota overcommitted | pool inventory, N-1 | adjust reservations |
| metrics missing | exporter and permissions | fail closed on acceptance |

## 9. Ten incident simulations

1. Tenant exceeds GPU ResourceQuota.
2. Quota allows admission but no eligible GPU node exists.
3. MIG profile key changes after Operator update.
4. Shared GPU tenant causes inference p99 regression.
5. Privileged tenant pod attempts host GPU access.
6. Tenant modifies toleration to access restricted pool.
7. Preemption kills training before checkpoint.
8. One zone loses GPU capacity and N-1 budget fails.
9. Tenant metrics expose another tenant's process identifiers.
10. GPU chargeback counts logical shares as physical GPUs.

Each exercise records UTC evidence, admission/scheduler events, tenant, GPU UUID, blast radius, rollback, SLO, RCA/COA and owner.

## 10. Eight operational runbooks

**RB-029-A — Quota admission:** distinguish API quota rejection from scheduler Pending.

**RB-029-B — GPU capacity:** reconcile allocatable, requests, node eligibility, reservations and N-1.

**RB-029-C — MIG/share mode:** inspect GPU SKU, strategy, profile keys and tenant isolation.

**RB-029-D — Tenant security:** audit RBAC, PSA, admission, runtime privileges and network boundaries.

**RB-029-E — Noisy neighbor:** correlate shared GPU occupancy, application SLO and workload concurrency.

**RB-029-F — Preemption/fairness:** protect checkpoints, inspect PriorityClass and queue behavior.

**RB-029-G — Safe change:** canary quota/pool policy with rollback, monitoring and owner approval.

**RB-029-H — RCA/FinOps:** document incidents, quota exceptions, chargeback and preventive controls.

## 11. Hands-on staging labs

1. Inventory namespaces, ResourceQuota, LimitRange, GPU resource keys, GPU Operator/plugin versions and node pools.
2. Create **approved disposable staging namespaces** with reviewed RBAC, PSA, NetworkPolicy and quota.
3. Test one permitted integer GPU request and a second request that exceeds quota; verify the rejection reason.
4. Test a request within quota but with an impossible node affinity; observe Pending versus admission rejection.
5. Verify full-GPU or MIG profile-specific requests against actual node allocatable resource names.
6. Exercise an approved shared-GPU contention simulation; compare tenant SLO and telemetry.
7. Confirm tenants cannot schedule onto protected node pools by adding arbitrary tolerations.
8. Demonstrate N-1 capacity, checkpoint, rollback, monitoring and owner signoff.

The [offline companion lab](../labs/029-gpu-resource-quotas-multitenancy-check.md) validates synthetic tenant admission policy with Python and pytest. **It does not prove Kubernetes RBAC, GPU isolation or real resource accounting.**

## 12. 100-point assessment

| Category | Points |
|---|---:|
| ResourceQuota and GPU accounting | 20 |
| MIG, sharing and scheduling isolation | 20 |
| Security and tenant boundaries | 20 |
| Incident response, fairness and FinOps | 20 |
| Automated tests and staging evidence | 20 |
| **Total** | **100** |

Pass >=90 with no critical tenant isolation, data security, capacity, checkpoint or SLO blocker.

## References

- [Kubernetes Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
- [Kubernetes Resource Management](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Kubernetes RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Kubernetes Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/)
- [NVIDIA Kubernetes Device Plugin](https://github.com/NVIDIA/k8s-device-plugin)
- [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/)
- [NVIDIA MIG User Guide](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/)

**Next: Chapter 030 — GPU Pod Security and Runtime Integration.**
