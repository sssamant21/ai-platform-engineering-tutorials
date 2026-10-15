# Chapter 10.5 — Cloud IAM, Supply Chain and Runtime Security

**Part 10 — Security | Canonical enterprise tutorial**

## 1. Objectives, prerequisites and safety

Harden TrueFoundry application and model-serving infrastructure from cloud identity through source/build/model provenance to Kubernetes runtime and GPU nodes. Build a verifiable chain of trust for deployment artifacts, workload permissions, admission controls and incident recovery.

**Safety:** [SAFE-READ] read-only discovery; [LAB-MUTATE] disposable staging; [CHANGE-CONTROL] production security changes require approval, change window, rollback and incident coordination. Cloud IAM, admission controllers, artifact-signing support and TrueFoundry features vary by provider and version: inventory the actual installation first. Never use production secrets, customer prompts, real private model weights or destructive security tests in the lab.

## 2. End-to-end chain of trust

```text
Developer → source repository → reviewed commit / protected branch
          → CI identity → build environment → SBOM / vulnerability scan
          → signed image digest / model artifact checksum + provenance
          → registry/object store with scoped IAM
          → admission/GitOps/TrueFoundry deployment policy
          → Kubernetes ServiceAccount + cloud workload identity
          → Pod security context / GPU node / container runtime
          → authenticated inference gateway → approved data dependencies
```

Distinct controls protect each boundary. A signed image does not prove its code is safe; a private registry does not prove provenance; a secure Pod does not imply a safe GPU driver or model artifact. Model artifacts can include executable formats or unsafe deserialization pathways. Treat models and container images as independently versioned supply-chain inputs.

## 3. Cloud IAM: principals, trust and authorization

Inventory cloud accounts/subscriptions, Kubernetes cluster/node roles, TrueFoundry agent roles, CI/CD roles, model-serving ServiceAccounts, object stores, container registries, KMS/Key Vault and telemetry roles. Document each role's trusted principal, conditions, allowed actions/resources, permission boundary, audit trail, credential lifetime and owner.

**AWS EKS:** distinguish node instance role, IRSA, EKS Pod Identity and CI OIDC federation. Restrict `sts:AssumeRoleWithWebIdentity` subject/audience for IRSA; EKS Pod Identity uses its own agent/association and role trust. Grant narrow S3 prefixes, ECR, KMS and Secrets Manager permissions; review resource policies, VPC endpoint policies and SCPs as independent constraints.

**Azure AKS:** use supported Microsoft Entra Workload Identity with exact federated subject/audience, managed identity and least-privilege Azure RBAC. Separate control-plane roles, kubelet identity, ACR pull permissions, Blob data-plane roles, Key Vault access and subscription-level RBAC. Private endpoints are networking, not authorization.

**Cross-account/cross-subscription:** explicitly validate resource policies, trust, external IDs/conditions where applicable, DNS/private routes and key permissions. A token exchange success does not imply permission to retrieve a specific artifact.

## 4. Safe discovery [SAFE-READ]

```bash
kubectl config current-context
kubectl get namespaces --show-labels
kubectl get serviceaccounts -A
kubectl get rolebinding,clusterrolebinding -A
kubectl get deployments,daemonsets -A -o wide
kubectl get pods -A -o custom-columns='NS:.metadata.namespace,POD:.metadata.name,SA:.spec.serviceAccountName,IMAGE:.spec.containers[*].image'
kubectl get validatingwebhookconfigurations,mutatingwebhookconfigurations
kubectl get clusterpolicies,policies -A 2>/dev/null || true
kubectl get constrainttemplates,constraints -A 2>/dev/null || true
kubectl get events -A --sort-by=.lastTimestamp | tail -40
```

Optional policy resources depend on installed CRDs. Do not assume a webhook is an image-signature verifier. Review admission configuration and fail-open/fail-closed behavior, exemption rules, audit/deny mode, failurePolicy and namespace selectors. Do not export secrets or credential-bearing environment variables.

## 5. Build and artifact supply chain

**Source:** protected branches, mandatory reviews, CODEOWNERS, signed/verified commits when policy requires, pinned dependencies, least-privilege CI permissions and no unreviewed workflow changes.

**Build:** isolated runners, minimal base images, pinned dependencies, deterministic/reproducible build where feasible, SBOM, SCA and vulnerability scans with documented severity/exception criteria. Never publish build secrets into layers or cache.

**Container image:** immutable digest `registry.example/model@sha256:...`, provenance attestation, signature verification using organization-approved tooling, trusted registry and digest pinning. Kubernetes deployment by digest prevents tag drift but does not independently verify signer identity.

**Model artifacts:** registry/storage IAM, version pinning and cryptographic hash, license and provenance checks, safe loading/deserialization, allowlisted download sources, malware and model-specific security review. Prefer formats that avoid arbitrary code execution when appropriate; avoid untrusted pickled model loading in privileged runtimes.

**Promotion:** staging → canary → production with policy gates and recorded artifact digests. Rollback must restore both image and model digest plus deployment configuration.

## 6. Kubernetes admission and Pod security

Use Kubernetes Pod Security Admission (baseline/restricted as compatible), or supported Kyverno/Gatekeeper policies, to control privileged mode, capabilities, host namespaces, hostPath, root filesystem, seccomp, serviceAccount token mounts, image registries and immutable digests.

Example non-GPU workload hardening baseline:
```yaml
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop: ["ALL"]
  seccompProfile:
    type: RuntimeDefault
```

The snippet is an illustrative **container-level** securityContext; `seccompProfile` may be set at Pod or container scope. Verify UID, writable cache/tmp volumes, filesystem ownership, model runtime and device access. GPU serving images and device plugins may require different settings; test and document any narrowly scoped exception. Do not weaken cluster-wide policies for a single GPU workload.

## 7. GPU runtime and node security

Inventory NVIDIA GPU Operator/device plugin, driver and CUDA compatibility, container toolkit, node OS/kernel patching, image provenance and scheduling labels/taints. Restrict privileged DaemonSets and host mounts to approved platform components; protect GPU nodes with workload isolation, quotas, admission and network controls. Review GPU metrics exposure and device health without assuming GPU isolation equals strong tenant isolation. For sensitive multi-tenant use cases, evaluate dedicated nodes or supported partitioning/MIG with threat model and vendor guidance.

## 8. Deployment and runtime enforcement

- Restrict registry pulls and artifact egress to approved destinations.
- Separate workload identity from node identity; disable unneeded ServiceAccount token automount.
- Apply resource requests/limits and namespace quotas; protect against abusive inference concurrency.
- Use NetworkPolicy and cloud firewalls for lateral movement and exfiltration paths.
- Validate TLS/authentication at inference gateway and model-serving application.
- Preserve logs for policy denials, unexpected exec, privileged Pod creation, new role bindings and cloud API access.
- Keep signed, versioned GitOps manifests as the source of truth; manual emergency changes require reconciliation.

## 9. Observability, evidence and release gates

Track IAM policy/trust changes, denied cloud actions, role assumption errors, CI identity usage, registry artifact digest, SBOM/scan status, signature/admission decisions, Pod Security violations, privileged workload creation, GPU driver/operator drift and unexpected egress. Correlate Git commit, build ID, image digest, model digest, Kubernetes rollout and TrueFoundry deployment identity. Avoid secrets, full prompts and model weights in telemetry.

Release gates: approved source revision, immutable artifact identity, risk-based scan result, valid provenance/signature where policy requires, least-privilege cloud identity, staging runtime compatibility, denied-action tests, production canary SLO and rollback rehearsal.

## 10. Twelve production troubleshooting runbooks

### RB-01 — Cloud IAM AssumeRole/identity exchange denied
Inspect actual workload identity mechanism, trust conditions, audience, issuer and role association. Restore minimum required trust; do not grant wildcard principals.

### RB-02 — Object storage 403 during model download
Separate identity acquisition, object/prefix permissions, encryption key, bucket policy, endpoint policy and private DNS/firewall.

### RB-03 — Registry pull unauthorized
Differentiate node image-pull identity from runtime model-fetch identity. Check ECR/ACR permissions, repository policy, tag/digest and network.

### RB-04 — Admission rejects image
Inspect webhook/constraint verdict, registry allowlist, digest/signature/provenance and exemptions. Correct artifact or policy through approved review, not global bypass.

### RB-05 — Signature/provenance verification fails
Confirm signer identity, trust root, attestation binding to exact digest and key rotation. Quarantine unknown artifact.

### RB-06 — Critical vulnerability found in running image
Assess exploitability and exposure, patch/rebuild, stage rollout, preserve digest evidence and follow vulnerability exception policy.

### RB-07 — Model artifact integrity mismatch
Stop deployment of suspicious model, compare pinned checksum/provenance and restore reviewed artifact. Investigate storage and CI access.

### RB-08 — Pod Security Admission denies GPU workload
Identify exact restricted setting and whether device plugin/runtime requires exception. Use namespace/workload-scoped approved exception only after risk review.

### RB-09 — GPU runtime fails after hardening
Check driver/toolkit, device plugin, node labels, resource limits, seccomp/capabilities and CUDA compatibility. Restore last-known-good staging settings without removing broad controls.

### RB-10 — Unexpected privileged DaemonSet
Inspect owner, image digest, host mounts, audit events and GitOps source; contain unauthorized workload under incident process.

### RB-11 — CI identity overprivileged or compromised
Revoke/rotate CI credentials, freeze promotion, review artifact provenance and audit, narrow role trust and redeploy verified builds.

### RB-12 — GitOps reintroduces unsafe image or IAM config
Identify desired-state commit and reconciliation owner; revert approved configuration, validate effective runtime/cloud permissions and negative tests.

## 11. Production change and rollback

Create change record with risk, owners, cloud IAM diff, admission policy diff, pinned image/model digests, GPU compatibility, expected allowed/denied actions, canary thresholds and rollback commit. Security incidents may require immediate revocation/quarantine even if service is interrupted; do not roll back to known-compromised artifacts or credentials.

Complete the [companion lab](../labs/05-cloud-iam-supply-chain-runtime-security-check.md) with **30 failure scenarios and 10 acceptance gates**. Formal review and live execution remain separate.

**Next:** Chapter 10.6 — Security Observability, Audit and Incident Response.
