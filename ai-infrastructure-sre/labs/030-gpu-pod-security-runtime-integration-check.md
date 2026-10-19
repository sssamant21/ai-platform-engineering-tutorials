# Lab 030 — GPU Pod Security and Runtime Integration Gate

**Python 3.11+ / pytest / offline synthetic evidence policy.** This lab makes no Kubernetes API calls and never modifies a GPU, host driver, runtime or namespace.

## Goal

Reject unsafe GPU application PodSpecs and incomplete runtime evidence. Test least-privilege admission, approved immutable images, runtime/CDI compatibility, device-plugin readiness, GPU identity, cloud workload identity, telemetry, checkpoint, N-1 and rollback.

## File: `gpu_pod_security_policy.py`

```python
from dataclasses import dataclass
import json
import math
import re

DIGEST = re.compile(r"^sha256:[a-fA-F0-9]{64}$")
RESOURCE = re.compile(r"^nvidia\.com/(?:gpu|mig-[A-Za-z0-9.-]+)$")

@dataclass(frozen=True)
class Workload:
    namespace: str
    image_digest: str
    gpu_resource: str
    gpu_count: int
    privileged: bool
    allow_privilege_escalation: bool
    host_pid: bool
    host_network: bool
    host_path_mount: bool
    manually_mounted_gpu_devices: bool
    run_as_non_root: bool
    read_only_root_fs: bool
    seccomp_runtime_default: bool
    all_capabilities_dropped: bool
    automount_service_account_token: bool
    psa_restricted_verified: bool
    rbac_least_privilege_verified: bool
    admission_policy_verified: bool
    image_provenance_verified: bool
    workload_identity_verified: bool
    tenant_isolation_verified: bool
    device_plugin_ready: bool
    toolkit_cdi_compatible: bool
    driver_cuda_compatible: bool
    gpu_uuid_verified: bool
    cuda_canary_passed: bool
    telemetry_age_s: float
    max_telemetry_age_s: float
    checkpoint_verified: bool
    n_minus_one_verified: bool
    change_approved: bool
    rollback_tested: bool
    slo_verified: bool

def evaluate(w):
    if not isinstance(w, Workload):
        raise TypeError("Workload required")
    if not isinstance(w.namespace, str) or not w.namespace.strip():
        return {"accepted": False, "reason": "missing_namespace"}
    if not isinstance(w.image_digest, str) or not DIGEST.fullmatch(w.image_digest):
        return {"accepted": False, "reason": "invalid_image_digest"}
    if not isinstance(w.gpu_resource, str) or not RESOURCE.fullmatch(w.gpu_resource):
        return {"accepted": False, "reason": "invalid_gpu_resource"}
    if type(w.gpu_count) is not int or w.gpu_count < 1:
        return {"accepted": False, "reason": "invalid_gpu_count"}
    if (any(type(v) not in (int, float) or not math.isfinite(v)
            for v in (w.telemetry_age_s, w.max_telemetry_age_s))
            or w.telemetry_age_s < 0 or w.max_telemetry_age_s <= 0):
        return {"accepted": False, "reason": "invalid_telemetry"}
    gates = {
        "no_privileged": w.privileged is False,
        "no_escalation": w.allow_privilege_escalation is False,
        "no_host_pid": w.host_pid is False,
        "no_host_network": w.host_network is False,
        "no_host_path": w.host_path_mount is False,
        "no_manual_devices": w.manually_mounted_gpu_devices is False,
        "non_root": w.run_as_non_root is True,
        "read_only_fs": w.read_only_root_fs is True,
        "seccomp": w.seccomp_runtime_default is True,
        "capabilities": w.all_capabilities_dropped is True,
        "no_sa_token": w.automount_service_account_token is False,
        "psa": w.psa_restricted_verified is True,
        "rbac": w.rbac_least_privilege_verified is True,
        "admission": w.admission_policy_verified is True,
        "provenance": w.image_provenance_verified is True,
        "workload_identity": w.workload_identity_verified is True,
        "tenant_isolation": w.tenant_isolation_verified is True,
        "device_plugin": w.device_plugin_ready is True,
        "runtime_cdi": w.toolkit_cdi_compatible is True,
        "driver_cuda": w.driver_cuda_compatible is True,
        "gpu_uuid": w.gpu_uuid_verified is True,
        "cuda_canary": w.cuda_canary_passed is True,
        "telemetry": w.telemetry_age_s <= w.max_telemetry_age_s,
        "checkpoint": w.checkpoint_verified is True,
        "n_minus_one": w.n_minus_one_verified is True,
        "change": w.change_approved is True,
        "rollback": w.rollback_tested is True,
        "slo": w.slo_verified is True,
    }
    failed = sorted(k for k, ok in gates.items() if not ok)
    return {"accepted": not failed, "failed_gates": failed,
            "namespace": w.namespace, "gpu_resource": w.gpu_resource}

def example():
    return Workload(
        "synthetic-gpu-staging", "sha256:" + "a" * 64, "nvidia.com/gpu", 1,
        *([False] * 6), *([True] * 4), False,
        *([True] * 11), 10.0, 120.0, *([True] * 5)
    )

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```

## File: `test_gpu_pod_security_policy.py`

```python
from dataclasses import replace
import pytest
from gpu_pod_security_policy import evaluate, example

def test_baseline():
    assert evaluate(example())["accepted"]

def test_type():
    with pytest.raises(TypeError):
        evaluate({})

def test_missing_namespace():
    assert evaluate(replace(example(), namespace=" "))["reason"] == "missing_namespace"

def test_invalid_digest():
    assert evaluate(replace(example(), image_digest="latest"))["reason"] == "invalid_image_digest"

def test_invalid_resource():
    assert evaluate(replace(example(), gpu_resource="gpu"))["reason"] == "invalid_gpu_resource"

def test_fractional_gpu():
    assert evaluate(replace(example(), gpu_count=0.5))["reason"] == "invalid_gpu_count"

def test_boolean_gpu():
    assert evaluate(replace(example(), gpu_count=True))["reason"] == "invalid_gpu_count"

def test_nan_telemetry():
    assert evaluate(replace(example(), telemetry_age_s=float("nan")))["reason"] == "invalid_telemetry"

def test_stale_telemetry():
    assert "telemetry" in evaluate(replace(example(), telemetry_age_s=121))["failed_gates"]

def test_nonboolean_approval():
    assert "change" in evaluate(replace(example(), change_approved="yes"))["failed_gates"]

@pytest.mark.parametrize("field,gate", [
    ("privileged", "no_privileged"),
    ("allow_privilege_escalation", "no_escalation"),
    ("host_pid", "no_host_pid"),
    ("host_network", "no_host_network"),
    ("host_path_mount", "no_host_path"),
    ("manually_mounted_gpu_devices", "no_manual_devices"),
    ("automount_service_account_token", "no_sa_token"),
])
def test_forbidden_true(field, gate):
    assert gate in evaluate(replace(example(), **{field: True}))["failed_gates"]

@pytest.mark.parametrize("field,gate", [
    ("run_as_non_root", "non_root"),
    ("read_only_root_fs", "read_only_fs"),
    ("seccomp_runtime_default", "seccomp"),
    ("all_capabilities_dropped", "capabilities"),
    ("psa_restricted_verified", "psa"),
    ("rbac_least_privilege_verified", "rbac"),
    ("admission_policy_verified", "admission"),
    ("image_provenance_verified", "provenance"),
    ("workload_identity_verified", "workload_identity"),
    ("tenant_isolation_verified", "tenant_isolation"),
    ("device_plugin_ready", "device_plugin"),
    ("toolkit_cdi_compatible", "runtime_cdi"),
    ("driver_cuda_compatible", "driver_cuda"),
    ("gpu_uuid_verified", "gpu_uuid"),
    ("cuda_canary_passed", "cuda_canary"),
    ("checkpoint_verified", "checkpoint"),
    ("n_minus_one_verified", "n_minus_one"),
    ("change_approved", "change"),
    ("rollback_tested", "rollback"),
    ("slo_verified", "slo"),
])
def test_required_true(field, gate):
    assert gate in evaluate(replace(example(), **{field: False}))["failed_gates"]
```

Run:

```bash
python -m pip install pytest
python gpu_pod_security_policy.py
python -m pytest -q
```

**Expected 37 cases:** 10 standalone, 7 forbidden-true parameterized and 20 required-true parameterized. Runtime execution has not been performed as part of publication.

## Ten synthetic incident drills

1. Privileged GPU tenant pod.
2. HostPath/manual GPU device mount.
3. Runtime CDI mismatch.
4. Missing GPU plugin registration.
5. Driver/CUDA incompatibility.
6. Missing approved image provenance.
7. Missing workload identity evidence.
8. Unapproved cross-tenant GPU sharing.
9. Stale GPU telemetry or failed CUDA canary.
10. Missing checkpoint, N-1, change approval or rollback.

## Staging acceptance

Inventory actual Pod Security Admission policy, RBAC, GPU Operator operands, device plugin, toolkit/CDI, driver, runtime and image digest. Run authorized server-side dry-run against approved staging namespace; verify forbidden privileged/hostPath PodSpecs are rejected. Deploy a reviewed non-root GPU canary, confirm GPU UUID/CUDA initialization, workload identity, secrets redaction, SLO, N-1 and rollback. The dry-run does not prove GPU device allocation or scheduling.

**Limitation:** This evaluator trusts declared evidence. It does not parse PodSpec YAML, verify signed images, call Kubernetes, prove device permissions, or enforce runtime isolation.

## 100-point rubric

Trust boundaries 20; Pod Security/RBAC 20; runtime/CDI 20; incident runbooks 20; tests/staging evidence 20. Pass >=90 with no critical privilege, tenant isolation or correctness blocker.
