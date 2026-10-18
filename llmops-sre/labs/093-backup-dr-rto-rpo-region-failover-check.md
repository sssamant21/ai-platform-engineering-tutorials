# Chapter 093 — Executable Backup, DR, RTO/RPO and Region Failover Lab

**Status:** Published. Offline execution, cross-region staging restore and independent review pending.

## 1. Setup
Python 3.11+ and pytest:
```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install pytest
```
PowerShell: `.venv\Scripts\Activate.ps1`.

## 2. Synthetic DR gate evaluator
Save as `dr_failover.py`:
```python
from dataclasses import dataclass
import json

@dataclass(frozen=True)
class Dependency:
    name: str
    lag_seconds: int
    rpo_target_seconds: int
    restored: bool
    integrity_verified: bool

def check_dependency(d):
    if not isinstance(d, Dependency):
        raise TypeError("Dependency required")
    if not d.name or not isinstance(d.name, str):
        return False
    if any(type(v) is not int for v in (d.lag_seconds, d.rpo_target_seconds)):
        return False
    if d.lag_seconds < 0 or d.rpo_target_seconds < 0:
        return False
    return (d.restored is True and d.integrity_verified is True
            and d.lag_seconds <= d.rpo_target_seconds)

@dataclass(frozen=True)
class Failover:
    dependencies: tuple[Dependency, ...]
    primary_writers_fenced: bool
    side_effect_workers_fenced: bool
    identity_verified: bool
    tenant_isolation_verified: bool
    residency_approved: bool
    signed_model_verified: bool
    gpu_capacity_verified: bool
    inference_probe_passed: bool
    slo_verified: bool
    measured_rto_seconds: int
    rto_target_seconds: int
    owner_approved: bool

def can_failover(f):
    if not isinstance(f, Failover):
        raise TypeError("Failover required")
    if not isinstance(f.dependencies, tuple) or not f.dependencies:
        return False
    if any(not check_dependency(d) for d in f.dependencies):
        return False
    if type(f.measured_rto_seconds) is not int or type(f.rto_target_seconds) is not int:
        return False
    if f.measured_rto_seconds < 0 or f.rto_target_seconds <= 0:
        return False
    gates = (
        f.primary_writers_fenced, f.side_effect_workers_fenced,
        f.identity_verified, f.tenant_isolation_verified,
        f.residency_approved, f.signed_model_verified,
        f.gpu_capacity_verified, f.inference_probe_passed,
        f.slo_verified, f.owner_approved,
    )
    return all(type(x) is bool and x for x in gates) and f.measured_rto_seconds <= f.rto_target_seconds

def example():
    return Failover(
        (Dependency("postgresql", 10, 30, True, True),
         Dependency("vector-index", 25, 60, True, True)),
        True, True, True, True, True, True, True, True, True,
        180, 300, True,
    )

if __name__ == "__main__":
    print(json.dumps({"failover_allowed": can_failover(example()),
                      "dependency_count": len(example().dependencies)}, indent=2))
```

**Limitations:** This is a deterministic **policy-gate simulator**, not a backup or restore tool. Lag, integrity, fencing, approvals and SLOs are operator-supplied assertions, not independently measured. A true restore drill must prove them from authoritative system evidence.

## 3. Automated tests
Save as `test_dr_failover.py`:
```python
from dataclasses import replace
import pytest
from dr_failover import Dependency, Failover, check_dependency, can_failover, example

def test_valid_failover():
    assert can_failover(example())

def test_rpo_boundary():
    assert check_dependency(Dependency("db", 30, 30, True, True))

def test_rpo_breach():
    assert not check_dependency(Dependency("db", 31, 30, True, True))

def test_unrestored_dependency():
    assert not check_dependency(Dependency("db", 0, 30, False, True))

def test_corrupt_dependency():
    assert not check_dependency(Dependency("db", 0, 30, True, False))

def test_empty_dependencies():
    assert not can_failover(replace(example(), dependencies=()))

def test_stale_index():
    f = example()
    stale = replace(f.dependencies[1], lag_seconds=100)
    assert not can_failover(replace(f, dependencies=(f.dependencies[0], stale)))

@pytest.mark.parametrize("field", [
    "primary_writers_fenced", "side_effect_workers_fenced",
    "identity_verified", "tenant_isolation_verified",
    "residency_approved", "signed_model_verified",
    "gpu_capacity_verified", "inference_probe_passed",
    "slo_verified", "owner_approved",
])
def test_critical_gate(field):
    assert not can_failover(replace(example(), **{field: False}))

def test_rto_breach():
    assert not can_failover(replace(example(), measured_rto_seconds=301))

def test_rto_boundary():
    assert can_failover(replace(example(), measured_rto_seconds=300))

def test_negative_lag():
    assert not check_dependency(Dependency("db", -1, 30, True, True))

def test_invalid_rto():
    assert not can_failover(replace(example(), rto_target_seconds=0))

def test_invalid_failover_type():
    with pytest.raises(TypeError):
        can_failover({})
```

Run:
```bash
python dr_failover.py
python -m pytest -q
```

Expected: **22 pytest cases** (10 parameterized and 12 standalone). Execution has not been performed.

## 4. Staging exercises
1. Record backup and replica checkpoints for each durable dependency.
2. Restore PostgreSQL/MongoDB to isolated staging and verify integrity.
3. Validate vector index freshness and embedding compatibility.
4. Fence primary and tool workers before simulated traffic promotion.
5. Verify signed model artifacts, KMS, identity, tenant isolation and residency.
6. Run synthetic inference, retrieval and SSE probes.
7. Measure actual RPO/RTO and compare with approved objectives.
8. Practice safe failback, reverse replication and budget reconciliation.
9. Produce UTC timeline, cost impact, reviewer approvals and RCA/COA.

## 5. 100-point assessment
| Area | Points |
|---|---:|
| Dependency backup and restore inventory | 20 |
| RPO/RTO measurement | 20 |
| Fencing, security and residency | 20 |
| Regional traffic recovery and SLO | 20 |
| Tests, evidence and operational review | 20 |
| **Total** | **100** |

Educational pass >=90/100 and zero critical dual-writer, tenant-isolation, residency or unverified restore failures. Live drills and independent review remain pending.
