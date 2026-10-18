# Chapter 096 — Enterprise LLM Inference Reference Architecture Acceptance Lab

**Status:** Published. Offline tests, real deployment and independent review pending.

## Setup
Python 3.11+ and pytest:
```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install pytest
```
PowerShell: `.venv\Scripts\Activate.ps1`.

## 1. Architecture policy evaluator
Save as `architecture_acceptance.py`:
```python
from dataclasses import dataclass
import json

@dataclass(frozen=True)
class Architecture:
    zones: dict[str, int]
    peak_demand: int
    isolated_environments: bool
    workload_identity: bool
    tenant_authorization: bool
    signed_artifacts: bool
    approved_model_routing: bool
    retrieval_acl_verified: bool
    tool_idempotency: bool
    observability_enabled: bool
    quality_release_gates: bool
    token_budget_enforced: bool
    dr_fencing_ready: bool
    owner_approved: bool

def evaluate(a):
    if not isinstance(a, Architecture):
        raise TypeError("Architecture required")
    if not isinstance(a.zones, dict) or len(a.zones) < 3:
        return {"accepted": False, "reason": "insufficient_zones"}
    if any(not isinstance(k, str) or not k or type(v) is not int or v < 0
           for k, v in a.zones.items()):
        return {"accepted": False, "reason": "invalid_zone_capacity"}
    if type(a.peak_demand) is not int or a.peak_demand < 0:
        return {"accepted": False, "reason": "invalid_demand"}
    n1 = sum(a.zones.values()) - max(a.zones.values())
    gates = {
        "isolated_environments": a.isolated_environments,
        "workload_identity": a.workload_identity,
        "tenant_authorization": a.tenant_authorization,
        "signed_artifacts": a.signed_artifacts,
        "approved_model_routing": a.approved_model_routing,
        "retrieval_acl_verified": a.retrieval_acl_verified,
        "tool_idempotency": a.tool_idempotency,
        "observability_enabled": a.observability_enabled,
        "quality_release_gates": a.quality_release_gates,
        "token_budget_enforced": a.token_budget_enforced,
        "dr_fencing_ready": a.dr_fencing_ready,
        "owner_approved": a.owner_approved,
    }
    failed = sorted(k for k, v in gates.items() if type(v) is not bool or not v)
    return {"accepted": not failed and n1 >= a.peak_demand,
            "n_minus_one_capacity": n1, "headroom": n1 - a.peak_demand,
            "failed_gates": failed}

def example():
    return Architecture(
        {"zone-a": 120, "zone-b": 120, "zone-c": 120}, 200,
        True, True, True, True, True, True, True,
        True, True, True, True, True,
    )

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```

**Limitations:** The capacity figures are synthetic safe throughput values, not GPU measurements. Security and SLO flags are asserted, not independently verified. Production sign-off requires actual evidence.

## 2. Pytest suite
Save as `test_architecture_acceptance.py`:
```python
from dataclasses import replace
import pytest
from architecture_acceptance import Architecture, evaluate, example

def test_reference_architecture():
    r = evaluate(example())
    assert r["accepted"] and r["n_minus_one_capacity"] == 240

def test_n1_capacity_breach():
    assert not evaluate(replace(example(), peak_demand=260))["accepted"]

def test_n1_capacity_boundary():
    assert evaluate(replace(example(), peak_demand=240))["accepted"]

def test_two_zones_rejected():
    assert not evaluate(replace(example(), zones={"a": 120, "b": 120}))["accepted"]

def test_asymmetric_zones():
    r = evaluate(replace(example(), zones={"a": 300, "b": 100, "c": 100}))
    assert not r["accepted"] and r["n_minus_one_capacity"] == 200

@pytest.mark.parametrize("field", [
    "isolated_environments", "workload_identity", "tenant_authorization",
    "signed_artifacts", "approved_model_routing", "retrieval_acl_verified",
    "tool_idempotency", "observability_enabled", "quality_release_gates",
    "token_budget_enforced", "dr_fencing_ready", "owner_approved",
])
def test_critical_architecture_gate(field):
    r = evaluate(replace(example(), **{field: False}))
    assert not r["accepted"] and field in r["failed_gates"]

def test_negative_capacity():
    assert evaluate(replace(example(), zones={"a": -1, "b": 120, "c": 120}))["reason"] == "invalid_zone_capacity"

def test_invalid_demand():
    assert evaluate(replace(example(), peak_demand=-1))["reason"] == "invalid_demand"

def test_invalid_type():
    with pytest.raises(TypeError):
        evaluate({})
```

Run:
```bash
python architecture_acceptance.py
python -m pytest -q
```
Expected: **20 pytest cases** (12 parameterized plus 8 standalone). Execution has not been performed.

## 3. Hands-on architecture review
1. Draw trust boundaries and identify all identity/policy enforcement points.
2. Create separate dev/staging/prod IAM and GitOps ownership matrices.
3. Pin model/tokenizer/runtime digests and validate signed provenance.
4. Verify tenant retrieval ACLs and tool side-effect authorization.
5. Measure GPU throughput by token-length class and calculate N-1 headroom.
6. Define user-facing SLOs, RPO/RTO and backup restore procedures.
7. Document cost budgets, quota policy and retry/fallback constraints.
8. Create a risk register, ADRs, rollback plan and independent approval checklist.

## 4. 100-point assessment
| Domain | Points |
|---|---:|
| Architecture and trust boundaries | 20 |
| Capacity and resilience | 20 |
| Security and provenance | 20 |
| Observability and FinOps | 20 |
| IaC/GitOps and evidence | 20 |
| **Total** | **100** |

Educational pass >=90/100 with no critical security or integrity failures. Live implementation and review remain pending.
