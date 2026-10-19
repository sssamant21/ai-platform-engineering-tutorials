# Chapter 030 — GPU Pod Security and Runtime Integration

**AI Infrastructure Engineering & SRE · Part 04 — Kubernetes GPU Administration · Chapter 030/100**

**Status:** Published documentation. Offline test execution, live GPU/Kubernetes staging validation, independent security review, and production acceptance are pending.

## Learning objectives

Explain the trust boundaries between GPU workloads, Kubernetes admission, kubelet, CRI/containerd, NVIDIA Container Toolkit, CDI, NVIDIA device plugin, GPU Operator, and host driver. Enforce least privilege without breaking CUDA device allocation. Diagnose security-policy and runtime integration failures, and validate staged changes with automated policy checks, incident exercises, rollback and a 100-point assessment.

## 1. Architecture and threat model

```text
Tenant / CI / GitOps
   | authenticated Kubernetes API
   v
RBAC -> admission policies -> Pod Security Admission -> quota
   |
scheduler -> kubelet -> CRI/containerd
                       |     |
                       |  NVIDIA Container Toolkit / CDI
                       |     |
                       |  device-plugin Allocate
                       |     |
                       +--> OCI container -> CUDA application
                                              |
                                    /dev/nvidia* and driver libraries
                                              |
                                 host NVIDIA kernel driver / GPU
Operator and privileged driver/toolkit/plugin operands: separate platform trust boundary
```

GPU device files, kernel drivers and host-mounted runtime components are security-sensitive. Kubernetes namespaces, taints and ResourceQuota are administrative boundaries, **not hardware isolation**. A GPU Operator driver/toolkit DaemonSet may legitimately need elevated host access, but an ordinary inference or training pod should not inherit those privileges. MIG can provide supported hardware partitioning; time-slicing and MPS do not offer equivalent isolation guarantees. Confirm the actual GPU SKU and threat model.

## 2. Runtime integration: device plugin, CDI and toolkit

1. The NVIDIA device plugin registers vendor extended resources with kubelet.
2. Kubernetes schedules a pod requesting `nvidia.com/gpu` or an appropriate MIG profile key.
3. Kubelet requests allocation of device resources.
4. Depending on plugin/toolkit/CRI configuration, device exposure uses environment variables, device specs, mounts, or Container Device Interface (CDI) device names.
5. NVIDIA Container Toolkit and the runtime integrate host driver capabilities into the OCI container.
6. The application initializes CUDA and validates expected GPU UUID, driver compatibility, memory headroom and workload correctness.

**Failure domains:** missing allocatable resources (plugin), Pending pod (scheduler), CreateContainerError (runtime/CDI), CUDA initialization error (driver/toolkit/framework), and application SLO regression (workload). Treat these separately.

CDI support and configuration depend on installed Kubernetes, containerd/CRI-O, toolkit and device-plugin versions. A `RuntimeClass` may be needed for some legacy setups but should not be added blindly to CDI-native deployments. Never enable privileged pods or mount `/dev/nvidia*` manually as a generic fix.

## 3. Least-privilege GPU workload manifest

The following is a **staging-only example**. Replace the illustrative image tag with an approved, scanned immutable digest; verify driver/CUDA compatibility and ensure the image works under the restricted security context.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-security-canary
  namespace: gpu-security-staging
  labels:
    app: gpu-security-canary
spec:
  restartPolicy: Never
  automountServiceAccountToken: false
  securityContext:
    runAsNonRoot: true
    runAsUser: 10001
    runAsGroup: 10001
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: cuda-check
      image: nvcr.io/nvidia/cuda:12.4.1-base-ubuntu22.04
      command: ["sh", "-c"]
      args: ["nvidia-smi -L"]
      securityContext:
        allowPrivilegeEscalation: false
        privileged: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
      resources:
        requests:
          cpu: "250m"
          memory: 512Mi
          nvidia.com/gpu: 1
        limits:
          cpu: "1"
          memory: 1Gi
          nvidia.com/gpu: 1
```

A container image or NVIDIA utility may require writable temporary directories or additional runtime permissions. Validate this canary on a dedicated staging node; add narrowly scoped ephemeral volumes only when necessary. The pod must not use hostPID, hostNetwork, hostPath, privileged mode, host device mounts or blanket capabilities. `runAsNonRoot` alone does not prove the image works as an unprivileged UID.

## 4. Namespace and admission hardening

Use Pod Security Admission (PSA) restricted policy for ordinary GPU application namespaces where compatible. Use namespace labels such as `pod-security.kubernetes.io/enforce: restricted` only after confirming supported policy versions and the workload's requirements. GPU Operator operands belong in separately governed platform namespaces with documented privileged exemptions; do **not** weaken all tenant namespaces to accommodate the Operator.

Layer RBAC, service account token minimization, immutable/scanned images, signed artifact verification where supported, network policy, secret manager/workload identity, ResourceQuota, node pool controls, and admission rules that block unapproved runtime classes, host mounts, privileged pods, cross-tenant selectors and unauthorized GPU sharing modes. NetworkPolicy behavior depends on the CNI and does not protect device files.

## 5. Read-only discovery and security audit

```bash
kubectl config current-context
kubectl get ns --show-labels
kubectl auth can-i create pods -n APPROVED_NAMESPACE
kubectl get pods -n APPROVED_NAMESPACE -o wide
kubectl get daemonsets -A | grep -Ei 'nvidia|gpu-operator'
kubectl get runtimeclasses
kubectl describe pod APPROVED_POD -n APPROVED_NAMESPACE
kubectl describe node APPROVED_GPU_NODE
kubectl get events -n APPROVED_NAMESPACE --sort-by=.lastTimestamp
```

Inspect the **actual** operator/plugin namespace, runtime class, CRI, CDI strategy, driver and toolkit version. Authorized host checks:

```bash
nvidia-smi -L
nvidia-smi --query-gpu=uuid,name,driver_version --format=csv
journalctl -u kubelet -b --no-pager | tail -n 100
journalctl -k -b --no-pager | grep -Ei 'NVRM|Xid|ECC|AER' | tail -n 100
```

When available, inspect `nvidia-ctk cdi list` and `nvidia-ctk --version` on the host; syntax and supported commands vary by toolkit release. Avoid printing secrets, full environment variables or unrestricted process command lines into incident tickets.

## 6. Supply chain, IAM and tenant boundaries

Use pinned image digests and approved registries, signed images and SBOM attestations where supported, minimal service accounts and cloud workload identity (EKS IRSA/Pod Identity or AKS Workload Identity where configured). Do not inject long-lived AWS/Azure credentials into GPU containers. Restrict registry pulls, image provenance, GPU Operator Helm chart upgrades and node image changes through change management.

Multi-tenant GPU sharing requires explicit approval. A namespace with `nvidia.com/gpu` quota does not guarantee data isolation. If multiple tenants share a physical GPU via time-slicing or MPS, validate isolation, fairness, denial-of-service risks, model-data handling and memory clearing according to vendor guarantees.

## 7. Runtime failure matrix

| Symptom | Likely boundary | First evidence |
|---|---|---|
| Pod forbidden | PSA/RBAC/admission | API response and audit |
| Pod Pending | quota/scheduler/plugin | scheduler events |
| CreateContainerError | CRI/CDI/toolkit | kubelet/runtime logs |
| `nvidia-smi` unavailable | image/toolkit/device visibility | image contents, allocation |
| CUDA driver mismatch | host driver/runtime | version matrix |
| Permission denied on GPU | UID/device permissions/LSM | container and host logs |
| GPU UUID differs | allocation/MIG mapping | plugin and NVML inventory |
| Device works but SLO fails | application/contended GPU | traces and DCGM |
| GPU Operator privileged failure | platform operand PSA/SCC | operator logs and policy |
| unexpected host access | privilege or mount drift | PodSpec, admission and audit |

## 8. Twelve troubleshooting scenarios

1. Restricted PSA rejects an application pod.
2. Tenant requests `privileged: true` to work around CUDA failure.
3. NVIDIA device plugin registers zero resources after runtime change.
4. CDI device reference is not resolved by containerd.
5. CUDA runtime cannot find compatible driver libraries.
6. Non-root process cannot access an allocated device.
7. Read-only root filesystem breaks a CUDA-dependent image.
8. Tenant attempts hostPath or `/dev/nvidia*` mounts.
9. GPU Operator driver DaemonSet is blocked by policy.
10. Image digest is unapproved or missing signature evidence.
11. Workload identity or registry authentication fails.
12. Shared GPU mode violates tenant isolation requirements.

For each: preserve pod UID, GPU UUID, runtime/toolkit/driver versions, admission decision, logs, blast radius, checkpoint and rollback.

## 9. Ten production incident simulations

1. PSA policy hardening blocks inference rollout.
2. Containerd/CDI upgrade causes GPU CreateContainerError.
3. Privileged application image is detected in a tenant namespace.
4. GPU Operator operands lose required host permissions.
5. RuntimeClass migration hides NVIDIA devices.
6. Driver/library mismatch breaks CUDA initialization.
7. Unapproved image digest is promoted to a GPU node pool.
8. GPU time-slicing introduces a tenant isolation violation.
9. Cloud workload identity failure prevents checkpoint access.
10. GPU pod logs expose sensitive environment variables.

Each simulation requires detection, UTC timeline, containment, tenant impact, recovery, correctness, SLO, RCA/COA and signoff.

## 10. Eight operational runbooks

**RB-030-A — Admission:** distinguish PSA/RBAC/webhook rejection from scheduling failure; preserve audit evidence.

**RB-030-B — Device injection:** reconcile kubelet Allocate, CDI, runtime, toolkit and GPU UUID mapping.

**RB-030-C — Non-root permissions:** test minimum UID/device access; avoid privileged fallback.

**RB-030-D — Operator privilege:** isolate platform operand permissions from tenant workloads.

**RB-030-E — Supply chain:** verify image digest, provenance, registry, SBOM and rollback image.

**RB-030-F — Tenant isolation:** investigate sharing mode, node-pool access, secrets and audit scope.

**RB-030-G — Safe upgrade/rollback:** canary containerd/toolkit/driver separately with N-1 and checkpoint protection.

**RB-030-H — RCA and acceptance:** verify admission, CUDA canary, workload correctness, security, telemetry and SLO.

## 11. Hands-on lab sequence

1. Inventory PSA, RBAC, GPU Operator, device plugin, toolkit, runtime, CDI, node taints and resource keys.
2. Review a pinned-image, non-root, restricted GPU PodSpec and run authorized server-side dry-run.
3. Submit an approved disposable GPU canary to a staging namespace; confirm scheduling and CUDA UUID.
4. Test deliberate PSA rejection of a privileged test PodSpec using dry-run (do not create a privileged pod).
5. Simulate a CDI/runtime mismatch in the **offline** lab, not on shared hosts.
6. Confirm missing cloud credentials do not leak into logs and workload identity is scoped.
7. Exercise a controlled rollback of staging configuration and verify checkpoint/SLO.
8. Record evidence, review ownership and acceptance signoff.

## 12. Executable companion lab

[Lab 030 — GPU Pod Security and Runtime Integration Gate](../labs/030-gpu-pod-security-runtime-integration-check.md) contains a Python 3.11+ offline policy evaluator and 30+ pytest cases. It validates declared evidence, not actual GPU runtime security or Kubernetes admission behavior.

## 13. 100-point acceptance

| Area | Points |
|---|---:|
| Trust boundaries and GPU runtime | 20 |
| Pod Security/RBAC and workload identity | 20 |
| CDI, toolkit and device-plugin troubleshooting | 20 |
| Incident response, rollback and runbooks | 20 |
| Automated tests and staging evidence | 20 |
| **Total** | **100** |

Pass >=90 with no critical tenant isolation, secrets, privilege escalation, GPU correctness or recovery blocker.

## References

- [Kubernetes Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [Kubernetes Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/)
- [Kubernetes Security Context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)
- [Kubernetes Device Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)
- [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/)
- [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/)
- [NVIDIA Kubernetes Device Plugin](https://github.com/NVIDIA/k8s-device-plugin)

**Next: Chapter 031 — GPU Node Autoscaling and Karpenter.**
