# Chapter 10.5 — Hands-on Lab: Cloud IAM, Supply Chain and Runtime Security

**Part 10 | Isolated staging | Formal execution pending**

## Mission and safety

Validate the chain of trust from source and CI identity through cloud IAM, image/model artifacts, Kubernetes admission, runtime and GPU nodes. Use approved staging, synthetic model data and disposable cloud resources. **[SAFE-READ]** inspect; **[LAB-MUTATE]** lab-only resources; **[CHANGE-CONTROL]** production approval. Never publish cloud credentials, signing private keys, customer data or untrusted executable model files.

## Lab 0 — Preflight [SAFE-READ]

```bash
kubectl config current-context
kubectl get namespaces --show-labels
kubectl get sa -A
kubectl get rolebinding,clusterrolebinding -A
kubectl get deployments,daemonsets -A -o wide
kubectl get pods -A -o custom-columns='NS:.metadata.namespace,POD:.metadata.name,SA:.spec.serviceAccountName,IMAGE:.spec.containers[*].image'
kubectl get validatingwebhookconfigurations,mutatingwebhookconfigurations
kubectl get events -A --sort-by=.lastTimestamp | tail -30
```

Record EKS IRSA/Pod Identity or AKS Workload Identity, CI federation, registry, storage, KMS/Key Vault, admission engine, GPU driver/operator versions, GitOps owner and abort criteria. If a component is absent, record a tabletop instead of installing cluster-wide infrastructure without approval.

## Lab 1 — IAM and trust [SAFE-READ]

Build an identity matrix for CI, TrueFoundry agent, model-serving Pod, node image pull and telemetry. Record issuer, trust subject/audience, allowed resources/actions, cloud network path, audit and owner. Verify that workload identity cannot read unapproved staging storage.

## Lab 2 — Supply-chain verification [SAFE-READ]

Trace protected source commit → CI run → SBOM/scan → image digest/signature/provenance → model checksum/version → GitOps manifest → running Pod image ID. Note which controls are enforced and which are only advisory. Never claim a signature validates code safety.

## Lab 3 — Admission and runtime [LAB-MUTATE]

In a disposable namespace, deploy a reviewed minimal workload by immutable digest and verify securityContext. Where an admission policy already exists, validate its denial of an unauthorized image or privileged Pod with a harmless rejected manifest. Do not modify cluster-wide admission policy. Confirm approved GPU workload compatibility using existing staging nodes and synthetic inference.

## Lab 4 — Cloud object access [LAB-MUTATE]

Using disposable object storage and a scoped staging identity, verify allowed object access and denied unrelated object access. Review IAM/KMS/endpoint logs without exposing values. Distinguish node pull from in-Pod model fetch.

## Lab 5 — Controlled recovery/tabletop

Rehearse bad image digest, revoked CI identity, artifact checksum mismatch, IAM deny, admission failure, GPU driver mismatch and GitOps reversion. Capture incident command, containment, last-known-good image/model digest, security sign-off and rollback. Never deploy intentionally malicious images/models or disable security controls to force a test.

## Lab 6 — Cleanup and evidence

Revert lab GitOps changes; remove disposable identities, storage and workloads; verify admission policy remains intact, no elevated IAM bindings persist and staging inference baseline recovers.

## Thirty failure scenarios

| # | Scenario | Evidence focus |
|---|---|---|
| 01 | CI OIDC issuer mismatch | Federation trust |
| 02 | CI branch/environment trust too broad | Role conditions |
| 03 | EKS IRSA subject mismatch | OIDC trust |
| 04 | EKS Pod Identity association absent | Agent/association |
| 05 | AKS federated credential mismatch | Entra subject/audience |
| 06 | Node role mistaken for Pod role | Identity path |
| 07 | S3/Blob object 403 | Data-plane IAM |
| 08 | KMS/Key Vault decrypt denied | Key policy |
| 09 | Private endpoint route blocked | DNS/network |
| 10 | Registry pull denied | Node pull identity |
| 11 | Image tag drift | Immutable digest |
| 12 | Image signature missing | Admission decision |
| 13 | Signature signer untrusted | Trust root |
| 14 | Attestation digest mismatch | Provenance binding |
| 15 | SBOM missing | Build evidence |
| 16 | Critical vulnerability unreviewed | Risk gate |
| 17 | CI build secret exposed | Build hygiene |
| 18 | Model checksum mismatch | Artifact integrity |
| 19 | Untrusted model serialization | Safe loading policy |
| 20 | Model storage permission broad | IAM scope |
| 21 | Admission webhook unavailable | Failure policy |
| 22 | Pod privileged unexpectedly | Security context |
| 23 | HostPath or hostNetwork used | Runtime isolation |
| 24 | ServiceAccount token auto-mounted | Credential surface |
| 25 | GPU driver/CUDA incompatible | Node/runtime |
| 26 | GPU device plugin unhealthy | Device availability |
| 27 | Unreviewed privileged DaemonSet | Node access |
| 28 | Namespace cross-tenant access | RBAC/network |
| 29 | CI credential compromised | Revocation/audit |
| 30 | GitOps restores unsafe artifact | Desired-state rollback |

Each scenario requires expected/actual result, timestamp, sanitized evidence, risk owner, containment, least-privilege fix, rollback and retest. Mark unexecuted tests honestly.

## Ten acceptance gates

| Gate | Required evidence | Status |
|---|---|---|
| G1 | Approved scope, architecture and IAM inventory | Pending |
| G2 | CI and workload identity trust constrained | Pending |
| G3 | Allowed/denied cloud access verified | Pending |
| G4 | Image/model digest and provenance traceable | Pending |
| G5 | SBOM, scan and risk policy reviewed | Pending |
| G6 | Admission enforcement and exceptions validated | Pending |
| G7 | Pod security and GPU runtime compatibility tested | Pending |
| G8 | Runtime/audit/drift alert coverage verified | Pending |
| G9 | Incident rollback and cleanup demonstrated | Pending |
| G10 | Security/SRE review and sign-off recorded | Pending |

**Pass:** all applicable gates supported by timestamped evidence and reviewer approval; pending/blocked tests are not passes.

## Evidence worksheet

UTC timestamp, cluster/context, identity type, cloud trust reference, resource/action, commit/build ID, image digest, model digest, policy verdict, GPU versions, sanitized audit ID, rollback commit and reviewer.

**Next:** Chapter 10.6 — Security Observability, Audit and Incident Response.
