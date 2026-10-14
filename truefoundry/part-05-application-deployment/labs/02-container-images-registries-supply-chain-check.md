# Lab 5.2 — Container Images, Registries and Supply Chain Check

**Canonical:** [Chapter 5.2](../02-container-images-registries-supply-chain/02-container-images-registries-supply-chain.md)  
**Mode:** Local disposable build and nonproduction deployment **[LAB-MUTATE]**; production Kubernetes inspection **[SAFE-READ]**.

## Objectives

Validate a reproducible OCI image, inspect its digest and architecture, document registry pull identity, diagnose pull errors, and verify TrueFoundry/Kubernetes use of an approved immutable artifact.

## 0. Safety and preflight

Use an approved development workstation or CI runner, test registry and nonproduction TrueFoundry workspace. Do not push to production registries or alter production image-pull secrets. Confirm Docker/compatible builder, registry permissions, image scanning policy and cleanup ownership.

```bash
docker version
kubectl config current-context
kubectl get namespaces
```

## 1. Prepare a sample HTTP application [LAB-MUTATE]

Create a disposable project directory with `app.py`, `requirements.txt`, `Dockerfile` and `.dockerignore`. Use a pinned, reviewed Flask version in `requirements.txt` and a real approved base-image digest.

```python
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

```dockerfile
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

This Flask server is **for disposable lab use only**, not production serving. Replace digest placeholders before execution and lock dependencies per organization policy.

## 2. Build and test locally [LAB-MUTATE]

```bash
docker build --pull -t hello-platform:lab .
docker image inspect hello-platform:lab --format '{{.Os}}/{{.Architecture}}'
docker run --rm -p 127.0.0.1:18080:8080 hello-platform:lab
```

In a second terminal:

```bash
curl -fsS http://127.0.0.1:18080/health
curl -fsS http://127.0.0.1:18080/ready
```

Stop the local container after verification. Record architecture, image ID and health response.

## 3. Registry publication and digest [LAB-MUTATE]

Only if test registry access is approved:

```bash
docker tag hello-platform:lab <approved-registry>/<team>/hello-platform:lab-<commit>
docker push <approved-registry>/<team>/hello-platform:lab-<commit>
docker image inspect <approved-registry>/<team>/hello-platform:lab-<commit> --format '{{json .RepoDigests}}'
```

Record the **registry-confirmed digest**. If `RepoDigests` is unavailable, query the registry through approved tooling; do not mistake a local image ID for the published manifest digest.

## 4. Scan, SBOM and provenance worksheet

Use your organization's approved scanning/signing tooling and pinned versions. Record outputs; do not fabricate results.

| Control | Tool / policy | Result | Evidence |
|---|---|---|---|
| Dependency lockfile | | | |
| OS/dependency vulnerability scan | | | |
| Secret scan | | | |
| SBOM (SPDX/CycloneDX) | | | |
| Image signature verification | | | |
| Build provenance | | | |
| Base-image and architecture check | | | |

If tooling is unavailable, mark **Not executed** with the responsible owner.

## 5. Inspect image pull and TrueFoundry deployment [SAFE-READ]

```bash
kubectl -n <namespace> get pods -o wide
kubectl -n <namespace> describe pod <pod>
kubectl -n <namespace> get pod <pod> -o jsonpath='{.spec.containers[*].image}{"\n"}'
kubectl -n <namespace> get pod <pod> -o jsonpath='{.spec.imagePullSecrets}{"\n"}'
kubectl -n <namespace> get serviceaccount <serviceaccount> -o yaml
kubectl -n <namespace> get events --sort-by=.lastTimestamp
```

Identify the actual pull identity. Empty `imagePullSecrets` may be valid when cloud node identity pulls images. Do not print Secret contents.

If authorized to deploy in a disposable TrueFoundry workspace, select the verified immutable image digest in the installed deployment workflow and validate that the rendered Pod uses it. Record the TrueFoundry revision and Pod Ready condition.

## 6. Non-destructive failure scenarios

| Scenario | Evidence to gather | Expected diagnosis |
|---|---|---|
| Wrong image digest | Pod events and registry manifest | Manifest not found |
| Expired or missing pull credentials | Events and actual pull identity | Authentication/authorization |
| Wrong image architecture | Node architecture and manifest | Platform mismatch |
| Private registry unreachable | DNS, network and TLS errors | Connectivity |
| Wrong registry CA | x509 event | Trust configuration |
| Image pulls, process crashes | Container logs and exit status | Application startup, not image pull |

For each, record first failed transition, actionable owner, approved remediation and rollback.

## 7. Cleanup [LAB-MUTATE]

Remove only disposable local containers/images and test registry tags created for this lab, following registry retention rules. Delete any nonproduction TrueFoundry deployment through its supported workflow, not an unmanaged Kubernetes patch. Verify no residual Pods or Services owned by the lab.

```bash
docker ps -a
kubectl -n <namespace> get pods,services
```

Do not delete shared namespaces, credentials or registry repositories.

## 8. Acceptance evidence

```text
Date/time / local and Kubernetes context:
TrueFoundry workspace / revision:
Git commit / CI build:
Base image digest / application image digest:
Image OS/architecture:
Local health results:
Registry and pull identity:
SBOM / scan / signature / provenance:
Pod image reference and pull events:
Application readiness:
Failure scenario diagnosis:
Known-good rollback digest:
Cleanup evidence:
Unexecuted checks:
```

**Pass:** immutable artifact, architecture and registry pull path are evidenced; approved image health and nonproduction deployment checks succeed where access is available; no credentials are exposed; disposable resources are cleaned up. Do not mark unexecuted controls as passed.

**Next:** Chapter 5.3 — Application Configuration, Environment Variables and Secrets.
