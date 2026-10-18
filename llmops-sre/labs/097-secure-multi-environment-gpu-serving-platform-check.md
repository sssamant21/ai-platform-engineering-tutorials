# Chapter 097 — Executable Secure Multi-Environment GPU Serving Platform Lab

**Status:** Published. Offline tests, real provisioning and independent security review pending.

## 1. Setup
Python 3.11+ and pytest:
```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install pytest
```
PowerShell: `.venv\Scripts\Activate.ps1`.

## 2. Offline environment and promotion evaluator
Save as `gpu_platform.py`:
```python
from dataclasses import dataclass
import json

ENVIRONMENTS = ("dev", "staging", "prod")

@dataclass(frozen=True)
class Environment:
    name: str
    identity: str
    registry_scope: str
    zone_gpu_capacity: dict[str, int]
    peak_gpu_demand: int
    network_policy_enforced: bool
    workload_identity: bool
    pod_security_enforced: bool
    signed_model_verified: bool
    immutable_image_verified: bool
    tenant_acl_verified: bool
    budget_guard_active: bool

def validate_platform(environments):
    if not isinstance(environments, tuple) or len(environments) != 3:
        return {"accepted": False, "reason": "three_environments_required"}
    if any(not isinstance(e, Environment) for e in environments):
        return {"accepted": False, "reason": "invalid_environment"}
    if tuple(e.name for e in environments) != ENVIRONMENTS:
        return {"accepted": False, "reason": "environment_order"}
    if len({e.identity for e in environments}) != 3 or any(not e.identity for e in environments):
        return {"accepted": False, "reason": "shared_identity"}
    if len({e.registry_scope for e in environments}) != 3 or any(not e.registry_scope for e in environments):
        return {"accepted": False, "reason": "shared_registry_scope"}
    for e in environments:
        if not isinstance(e.zone_gpu_capacity, dict) or len(e.zone_gpu_capacity) < 3:
            return {"accepted": False, "reason": e.name + "_zone_count"}
        if any(not isinstance(k, str) or not k or type(v) is not int or v < 0
               for k, v in e.zone_gpu_capacity.items()):
            return {"accepted": False, "reason": e.name + "_invalid_capacity"}
        if type(e.peak_gpu_demand) is not int or e.peak_gpu_demand < 0:
            return {"accepted": False, "reason": e.name + "_invalid_demand"}
        n1 = sum(e.zone_gpu_capacity.values()) - max(e.zone_gpu_capacity.values())
        if e.name == "prod" and n1 < e.peak_gpu_demand:
            return {"accepted": False, "reason": "prod_n_minus_one_shortfall"}
        gates = (e.network_policy_enforced, e.workload_identity,
                 e.pod_security_enforced, e.signed_model_verified,
                 e.immutable_image_verified, e.tenant_acl_verified,
                 e.budget_guard_active)
        if any(type(x) is not bool or not x for x in gates):
            return {"accepted": False, "reason": e.name + "_security_gate"}
    return {"accepted": True, "reason": "all_gates_passed"}

def example():
    return tuple(
        Environment(name, name + "-identity", name + "-registry",
                    {"a": 4, "b": 4, "c": 4}, 6,
                    True, True, True, True, True, True, True)
        for name in ENVIRONMENTS
    )

if __name__ == "__main__":
    print(json.dumps(validate_platform(example()), indent=2))
```

**Limitations:** The identities, scopes, capacities and security gates are synthetic assertions. This evaluator does not authenticate to cloud accounts, test Kubernetes NetworkPolicy enforcement, inspect actual GPU quotas or verify signatures. Production acceptance requires live tests and independent evidence.

## 3. Pytest suite
Save as `test_gpu_platform.py`:
```python
from dataclasses import replace
import pytest
from gpu_platform import Environment, validate_platform, example

def modify(index, **changes):
    envs = list(example())
    envs[index] = replace(envs[index], **changes)
    return tuple(envs)

def test_valid_platform():
    assert validate_platform(example())["accepted"]

def test_shared_identity():
    assert not validate_platform(modify(2, identity="dev-identity"))["accepted"]

def test_shared_registry():
    assert not validate_platform(modify(1, registry_scope="prod-registry"))["accepted"]

def test_missing_environment():
    assert not validate_platform(example()[:2])["accepted"]

def test_wrong_order():
    a = example()
    assert not validate_platform((a[1], a[0], a[2]))["accepted"]

def test_prod_n1_shortfall():
    assert validate_platform(modify(2, peak_gpu_demand=9))["reason"] == "prod_n_minus_one_shortfall"

def test_prod_n1_boundary():
    assert validate_platform(modify(2, peak_gpu_demand=8))["accepted"]

@pytest.mark.parametrize("field", [
    "network_policy_enforced", "workload_identity",
    "pod_security_enforced", "signed_model_verified",
    "immutable_image_verified", "tenant_acl_verified",
    "budget_guard_active",
])
def test_security_gate(field):
    assert not validate_platform(modify(2, **{field: False}))["accepted"]

def test_negative_gpu_capacity():
    assert not validate_platform(modify(2, zone_gpu_capacity={"a": -1, "b": 4, "c": 4}))["accepted"]

def test_invalid_demand():
    assert not validate_platform(modify(2, peak_gpu_demand=-1))["accepted"]

def test_insufficient_zones():
    assert not validate_platform(modify(2, zone_gpu_capacity={"a": 4, "b": 4}))["accepted"]

def test_invalid_environment_type():
    assert not validate_platform(({}, {}, {}))["accepted"]
```

Run:
```bash
python gpu_platform.py
python -m pytest -q
```
Expected: **18 pytest cases** (7 parameterized and 11 standalone). Execution has not been performed.

## 4. Staging implementation checklist
1. Provision separate dev/staging/prod cloud identities and resource boundaries.
2. Apply tested RBAC, Pod Security Admission and CNI-supported network policies.
3. Deploy GPU device plugin and inspect actual allocatable GPUs per zone.
4. Verify workload identity can fetch only approved signed model artifacts.
5. Run negative tests for cross-environment and cross-tenant access.
6. Confirm staging/prod GPU quotas, capacity headroom and model warmup.
7. Enforce protected GitOps promotions, approvals and rollback.
8. Collect audit, SLO, cost and security evidence before production sign-off.

## 5. 100-point assessment
| Area | Points |
|---|---:|
| Environment isolation | 20 |
| GPU scheduling and N-1 capacity | 20 |
| Security and provenance | 20 |
| GitOps promotion and rollback | 20 |
| Tests, runbooks and audit evidence | 20 |
| **Total** | **100** |

Educational pass >=90/100 with no critical isolation or provenance bypass. Live deployment and independent review pending.
