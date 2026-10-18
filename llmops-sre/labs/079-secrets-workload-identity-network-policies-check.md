# Chapter 079 — Executable Secrets, Workload Identity and Network Policies Lab

**Status:** Published; execution and independent security review pending. The default lab is offline and does not contact a cluster or secret manager.

## 1. Setup
Python 3.11+ and pytest:
```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install pytest
```
PowerShell: `.venv\Scripts\Activate.ps1`.

## 2. Deterministic identity, secret and network evaluator
Save as `identity_network_lab.py`:
```python
from dataclasses import dataclass
import json

@dataclass(frozen=True)
class Workload:
    namespace: str
    service_account: str
    issuer: str
    audience: str
    token_valid: bool

@dataclass(frozen=True)
class SecretRequest:
    path: str
    action: str

TRUSTED_ISSUER = "https://issuer.example.invalid/cluster"
EXPECTED_AUDIENCE = "secret-manager"
ALLOWED_IDENTITY = ("llm-inference", "inference-reader")
ALLOWED_SECRET_PATH = "llm/staging/inference-token"

def authorize_secret(workload, request):
    if workload.issuer != TRUSTED_ISSUER or workload.audience != EXPECTED_AUDIENCE:
        return {"allow": False, "reason": "federation_trust_denied"}
    if workload.token_valid is not True:
        return {"allow": False, "reason": "invalid_token"}
    if (workload.namespace, workload.service_account) != ALLOWED_IDENTITY:
        return {"allow": False, "reason": "workload_denied"}
    if request.action != "read" or request.path != ALLOWED_SECRET_PATH:
        return {"allow": False, "reason": "secret_scope_denied"}
    return {"allow": True, "reason": "policy_allow"}

ALLOWED_FLOWS = {
    ("llm-gateway", "gateway", "llm-inference", "inference", 8000, "TCP"),
    ("llm-inference", "inference", "kube-system", "dns", 53, "UDP"),
    ("llm-inference", "inference", "kube-system", "dns", 53, "TCP"),
    ("llm-inference", "inference", "llm-data", "vector-db", 5432, "TCP"),
}

def allow_flow(source_ns, source_app, dest_ns, dest_app, port, protocol):
    if type(port) is not int or not 1 <= port <= 65535:
        return False
    return (source_ns, source_app, dest_ns, dest_app, port, protocol) in ALLOWED_FLOWS

if __name__ == "__main__":
    w = Workload("llm-inference", "inference-reader",
                 TRUSTED_ISSUER, EXPECTED_AUDIENCE, True)
    print(json.dumps(authorize_secret(w, SecretRequest(ALLOWED_SECRET_PATH, "read")), indent=2))
    print("gateway_to_inference", allow_flow("llm-gateway", "gateway", "llm-inference", "inference", 8000, "TCP"))
```

**Limitations:** This evaluator accepts a `token_valid` boolean and does not verify cryptographic JWTs, network packets, CNI policies or cloud IAM. It is a policy-unit-test teaching aid only. `ALLOWED_FLOWS` is an application-layer model, not a Kubernetes NetworkPolicy interpreter.

## 3. Automated tests
Save as `test_identity_network_lab.py`:
```python
from dataclasses import replace
import pytest
from identity_network_lab import (
    Workload, SecretRequest, authorize_secret, allow_flow,
    TRUSTED_ISSUER, EXPECTED_AUDIENCE, ALLOWED_SECRET_PATH
)

W = Workload("llm-inference", "inference-reader", TRUSTED_ISSUER, EXPECTED_AUDIENCE, True)
R = SecretRequest(ALLOWED_SECRET_PATH, "read")

def test_authorized_secret_read():
    assert authorize_secret(W, R)["allow"]

def test_wrong_issuer():
    assert not authorize_secret(replace(W, issuer="untrusted"), R)["allow"]

def test_wrong_audience():
    assert not authorize_secret(replace(W, audience="other"), R)["allow"]

def test_expired_or_invalid_token():
    assert not authorize_secret(replace(W, token_valid=False), R)["allow"]

def test_wrong_namespace():
    assert not authorize_secret(replace(W, namespace="default"), R)["allow"]

def test_wrong_service_account():
    assert not authorize_secret(replace(W, service_account="default"), R)["allow"]

def test_secret_scope_denied():
    assert not authorize_secret(W, SecretRequest("llm/prod/root", "read"))["allow"]

def test_write_denied():
    assert not authorize_secret(W, SecretRequest(ALLOWED_SECRET_PATH, "write"))["allow"]

@pytest.mark.parametrize("flow", [
    ("llm-gateway", "gateway", "llm-inference", "inference", 8000, "TCP"),
    ("llm-inference", "inference", "kube-system", "dns", 53, "UDP"),
    ("llm-inference", "inference", "llm-data", "vector-db", 5432, "TCP"),
])
def test_allowed_flows(flow):
    assert allow_flow(*flow)

@pytest.mark.parametrize("flow", [
    ("default", "unknown", "llm-inference", "inference", 8000, "TCP"),
    ("llm-inference", "inference", "llm-data", "vector-db", 5432, "UDP"),
    ("llm-inference", "inference", "internet", "unknown", 443, "TCP"),
    ("llm-inference", "inference", "kube-system", "dns", 0, "UDP"),
])
def test_denied_flows(flow):
    assert not allow_flow(*flow)
```

Run:
```bash
python identity_network_lab.py
python -m pytest -q
```
Expected: **15 pytest cases** (8 standalone plus 7 parameterized). Execution not yet performed.

## 4. Optional staging-only Kubernetes exercise
**Do not run against production.** Require a disposable cluster/namespace, enforcement-capable CNI, change approval and a tested rollback.

Save as `default-deny.yaml`:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: llm-inference
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
```
Only after creating and verifying a disposable `llm-inference` namespace:
```bash
kubectl apply --dry-run=server -f default-deny.yaml
kubectl apply -f default-deny.yaml
kubectl -n llm-inference get networkpolicy default-deny
# Run explicit authorized connectivity probes from selected test pods.
# Rollback under the test change plan:
kubectl delete -f default-deny.yaml
```
A default-deny policy may break DNS and image/model artifact downloads; add minimum required allow policies **before** running service workloads. Verify actual enforcement using denied/allowed connectivity probes, not merely `kubectl get`.

## 5. Failure drills and acceptance
1. Reject wrong issuer, audience, namespace, service account and token-validity assertion.
2. Reject secret write and non-allowlisted path.
3. Verify only explicitly allowlisted network flows are modeled as allowed.
4. In staging, test DNS, gateway ingress, vector DB and unapproved egress.
5. Simulate token expiration and planned secret rotation without logging values.
6. Confirm rollback and audit evidence for NetworkPolicy changes.

| Domain | Points |
|---|---:|
| Workload identity trust and scope | 20 |
| Secret read/rotation/revocation design | 20 |
| Network segmentation and enforcement | 20 |
| Automated regression tests | 20 |
| Runbooks, audit and security review | 20 |
| **Total** | **100** |

Educational acceptance >=90/100 with no critical trust or egress failures. Production acceptance requires real federation, CNI enforcement, secret-manager integration and independent security review.
