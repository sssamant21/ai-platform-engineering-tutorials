# 5.2 — Container Images, Registries and Supply Chain

**Part:** 05 — Application Deployment  
**Audience:** TrueFoundry application teams, platform engineers, DevSecOps, Kubernetes SREs  
**Safety:** Discovery is **[SAFE-READ]**. Builds, pushes, registry changes, secret changes and deployments are **[LAB-MUTATE]** or **[CHANGE-CONTROL]**.

## 1. Objectives

Build and publish a reproducible, least-privilege container image; understand image tags versus immutable digests; integrate approved private registries with TrueFoundry and Kubernetes; verify provenance, vulnerability scanning and signing; and diagnose image-pull failures without leaking credentials.

## 2. Supply-chain architecture

```text
Source control (protected branch, reviewed commit)
   ↓
CI build (pinned builder, locked dependencies)
   ↓
SBOM + vulnerability and secret scans
   ↓
Image signing / provenance attestation (where supported)
   ↓
Approved registry (ECR / ACR / Artifact Registry / other)
   ↓
TrueFoundry deployment desired state (image@sha256:digest)
   ↓
Kubernetes Pod spec + ServiceAccount / imagePullSecrets
   ↓
Kubelet image pull → container startup → readiness
```

The registry credential and deployment identity are different concerns. TrueFoundry's registry integration and Kubernetes node/pod image-pull mechanism depend on installed version and cluster configuration. Inspect the actual rendered Pod and cloud identity path.

## 3. Docker/OCI image anatomy

An OCI image consists of a manifest (or multi-platform index), configuration and layers addressed by content digests. A tag such as `v1.2` can be moved; a digest identifies content. **Pin production deployments to an immutable image digest** and retain a human-readable version/commit annotation for traceability.

The digest of a multi-platform index can differ from a platform-specific manifest digest. Confirm the node's CPU architecture and image platform selection. An image can pull successfully but fail with `exec format error` if the architecture is wrong.

### Minimal example application

```python
# app.py — illustrative sample; pin versions in your build lockfile
from flask import Flask, jsonify
app = Flask(__name__)

@app.get("/health")
def health():
    return jsonify(status="ok")

@app.get("/ready")
def ready():
    return jsonify(status="ready")

@app.get("/")
def index():
    return jsonify(service="hello-platform")
```

Use a version-pinned dependency lockfile; do not install unpinned latest dependencies in production.

### Example Dockerfile

```dockerfile
# Replace with an approved base image pinned to a verified digest.
FROM python:3.12-slim@sha256:<approved-base-image-digest>
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
RUN useradd --uid 10001 --create-home appuser && chown -R appuser:appuser /app
USER 10001
EXPOSE 8080
CMD ["python", "-m", "flask", "--app", "app", "run", "--host", "0.0.0.0", "--port", "8080"]
```

**Lab-only server:** Flask's development server is not a production-serving choice. Use an approved production WSGI server, e.g. a pinned Gunicorn dependency and appropriate command, in real deployments. Verify filesystem permissions, non-root behavior and readiness semantics. The placeholder digest must be replaced with a real digest before building.

### Build context hygiene

```dockerignore
.git
.env
*.pem
*.key
__pycache__/
.pytest_cache/
node_modules/
```

A `.dockerignore` reduces accidental inclusion but is not a substitute for secret scanning. Never put tokens in Dockerfile `ARG`/`ENV`, image layers or build logs. Use approved build-secret mechanisms.

## 4. Local build, test and digest [LAB-MUTATE]

In a disposable workstation/CI runner:

```bash
docker build --pull -t hello-platform:lab .
docker image inspect hello-platform:lab --format '{{.Os}}/{{.Architecture}}'
docker run --rm --name hello-platform-test -p 127.0.0.1:18080:8080 hello-platform:lab
```

In another terminal:

```bash
curl -fsS http://127.0.0.1:18080/health
curl -fsS http://127.0.0.1:18080/ready
```

A locally built image may have an image ID but no registry `RepoDigests` until pushed/pulled. Obtain and record the registry digest **after push**, using approved tooling. Test the exact immutable digest used for deployment.

## 5. Private registry identity and permissions

| Registry | Common supported identity strategy | What to verify |
|---|---|---|
| Amazon ECR | EKS node IAM role or approved image-pull credential mechanism | ECR auth and repository pull permissions, regional endpoint |
| Azure ACR | AKS kubelet managed identity or approved pull secret | `AcrPull`/equivalent scoped permissions, registry networking |
| Google Artifact Registry | GKE node/workload configuration and supported image-pull identity | Artifact Registry Reader permissions, repository region |
| Other OCI registry | Approved Kubernetes `imagePullSecrets` or platform integration | Secret namespace, credentials expiry, registry CA |

**Important:** a workload's cloud service account/IRSA/Workload Identity does not automatically authenticate the kubelet's image pull. Determine which identity actually pulls images for the cluster and node type.

### Read-only checks

```bash
kubectl config current-context
kubectl -n <namespace> get serviceaccount <serviceaccount> -o yaml
kubectl -n <namespace> get pod <pod> -o jsonpath='{.spec.imagePullSecrets}{"\n"}'
kubectl -n <namespace> get pod <pod> -o jsonpath='{.spec.containers[*].image}{"\n"}'
kubectl -n <namespace> describe pod <pod>
```

Do not print Secret data. If `imagePullSecrets` is empty, that is not necessarily wrong: node/cloud-managed pull identity may be in use.

## 6. TrueFoundry integration

1. Confirm TrueFoundry workspace and Kubernetes cluster association.
2. Choose the supported application image deployment workflow for the installed TrueFoundry version.
3. Configure registry access using the organization's approved TrueFoundry integration or cloud-managed Kubernetes pull identity.
4. Supply `registry/repository@sha256:<verified-digest>`, not an untrusted floating tag.
5. Confirm deployment identity, resource policy, security context and internal exposure.
6. Inspect the rendered Pod image, image pull events and successful readiness.
7. Record image digest, Git commit, build run, SBOM, scan/signature status and TrueFoundry revision.

Do not assume a specific TrueFoundry CLI flag or API schema; version-specific workflows must be verified against installed documentation.

## 7. Scanning, SBOM, signatures and provenance

**Recommended controls:**
- Generate a Software Bill of Materials (SBOM) in SPDX or CycloneDX format with approved tooling.
- Scan OS and application dependencies against current vulnerability feeds.
- Define risk-based gating for critical exploitable findings; account for false positives and documented exceptions.
- Sign image digests and verify signatures in CI/admission where supported.
- Record build provenance/attestations tying artifact to reviewed source and build process.
- Use minimal bases, regular patching, pinned dependencies and reproducible builds where feasible.
- Protect CI tokens and restrict registry push/delete permissions.

A passing vulnerability scan is not proof of a secure image. Signatures establish provenance/integrity only when trust roots and policy are configured correctly.

### Generic CI/CD stages

```yaml
stages:
  - lint-and-test
  - build
  - scan-and-sbom
  - sign-and-attest
  - publish
  - deploy-nonprod
  - validate
  - promote
```

This is a conceptual pipeline, not executable GitLab/GitHub syntax. Pin tool versions and use least-privilege short-lived CI credentials.

## 8. Kubernetes image-pull diagnostics [SAFE-READ]

```bash
kubectl -n <namespace> get pods -o wide
kubectl -n <namespace> describe pod <pod>
kubectl -n <namespace> get events --sort-by=.lastTimestamp
kubectl -n <namespace> get deployment <deployment> -o yaml
```

Focus on **Events** for registry URL, auth failures, TLS trust, DNS, timeout, manifest mismatch and rate limiting. Sanitize tokens and internal URLs when sharing externally.

| Symptom | Evidence | First failed transition | Next action |
|---|---|---|---|
| `ErrImagePull` / `ImagePullBackOff` | Pod events | Registry → node image pull | Read detailed error |
| 401 / unauthorized | Registry auth, node/pull identity | Identity → registry | Verify pull permissions |
| 403 / denied | Repository policy, image path | Authorization | Check scoped repository access |
| `manifest unknown` | Tag/digest and repository | Image reference → manifest | Verify artifact exists |
| `no matching manifest` | Node architecture, manifest index | Platform resolution | Publish correct architecture |
| `x509: certificate signed by unknown authority` | Registry CA, runtime trust | TLS trust | Use approved CA distribution |
| DNS lookup failure | Node DNS, egress, endpoint | Network → registry | Diagnose DNS and routing |
| Timeout / connection refused | Firewall, proxy, private endpoint | Network connectivity | Check egress/private link |
| Pull rate limit | Registry quota and retry events | Registry availability | Use approved caching/mirror |
| Container starts then crashes | Pod state, logs, image entrypoint | Image pull → process startup | Debug app, not registry |
| `exec format error` | Image architecture | Image → executable platform | Rebuild for correct platform |

Do not recreate image-pull secrets blindly; determine whether node-managed pull credentials are the actual mechanism.

## 9. Production incident runbook

1. Identify cluster, namespace, TrueFoundry deployment, image digest and first failed Pod.
2. Capture Pod events and node placement; distinguish pull failure from scheduling and application startup.
3. Confirm the image manifest exists in the expected repository/region and supports the node architecture.
4. Identify the **actual image-pull identity** and permissions; never disclose credentials.
5. Check registry endpoint, DNS, TLS, private connectivity and rate limits.
6. Compare current digest, Git commit and CI build with last known-good artifact.
7. Record the **Lowest Proven Healthy Layer** and **First Failed Transition**.
8. Choose the smallest authorized remediation: permission fix, registry network repair, corrected digest or last-known-good image rollback.
9. Validate new Pod Ready, health endpoints, app requests and error/latency metrics.

## 10. Rollback, disaster readiness and retention

Preserve known-good digests in registry retention policy; tags alone are insufficient rollback references. Record image scan and provenance artifacts with each release. A registry outage may block new Pod scheduling even while existing Pods continue serving. Validate regional availability, pull-through caching/mirroring policies and image retention according to RTO/RPO requirements.

**Rollback:** update the authoritative TrueFoundry/GitOps deployment source to the last known-good digest, confirm reconciliation, and verify Kubernetes image digest and application readiness. A manual `kubectl rollout undo` may be overwritten by platform reconciliation.

## 11. Acceptance checklist

- [ ] Immutable image digest and supported architecture verified.
- [ ] Base image and dependencies pinned; non-root execution assessed.
- [ ] No secrets in image, Dockerfile, logs or CI artifacts.
- [ ] Registry pull identity and permissions documented.
- [ ] SBOM, scan, signature and provenance policies evaluated.
- [ ] TrueFoundry desired image matches rendered Kubernetes Pod.
- [ ] Pull failure troubleshooting and rollback demonstrated.
- [ ] Application readiness and request validation complete.

**Hands-on lab:** [5.2 — Container Image and Registry Validation](../labs/02-container-images-registries-supply-chain-check.md)  
**Next:** Chapter 5.3 — Application Configuration, Environment Variables and Secrets.

## References

- [TrueFoundry Documentation](https://docs.truefoundry.com/)
- [OCI Image Specification](https://github.com/opencontainers/image-spec)
- [Kubernetes Images](https://kubernetes.io/docs/concepts/containers/images/)
- [Kubernetes Pull an Image from a Private Registry](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/)
- [Kubernetes Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Sigstore Cosign](https://docs.sigstore.dev/cosign/)
- [SLSA Supply Chain Security](https://slsa.dev/)
- [CycloneDX](https://cyclonedx.org/)
