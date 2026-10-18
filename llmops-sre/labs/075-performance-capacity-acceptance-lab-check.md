# Chapter 075 — Executable Performance and Capacity Acceptance Lab

**Status:** Published; live execution and technical/security review pending. Offline synthetic acceptance gates only.

## 1. Prerequisites
Python 3.11+ and pytest. In a disposable lab directory:
```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install pytest
```
Windows PowerShell activation: `.venv\Scripts\Activate.ps1`.

## 2. Acceptance engine
Save as `acceptance.py`:
```python
from dataclasses import dataclass, asdict
from decimal import Decimal
import json
import math

@dataclass(frozen=True)
class Evidence:
    offered: int
    completed: int
    good: int
    ttft_p95_s: float
    tpot_p95_s: float
    kv_peak_fraction: float
    oom_events: int
    critical_safety_failures: int
    quality_pass: bool
    peak_rps: float
    safe_rps_per_replica: float
    ready_replicas: int
    failure_domain_replicas: int
    rollout_max_unavailable: int
    gpu_cost_usd: float
    good_output_tokens: int
    max_cost_per_million_good_tokens: float

def evaluate(e):
    for key in ("offered", "completed", "good", "oom_events",
                "critical_safety_failures", "ready_replicas",
                "failure_domain_replicas", "rollout_max_unavailable",
                "good_output_tokens"):
        x = getattr(e, key)
        if type(x) is not int or x < 0:
            raise ValueError(f"invalid {key}")
    if e.offered == 0 or not (0 <= e.good <= e.completed <= e.offered):
        raise ValueError("invalid request accounting")
    if e.failure_domain_replicas > e.ready_replicas:
        raise ValueError("invalid failure domain")
    if e.rollout_max_unavailable > e.ready_replicas:
        raise ValueError("invalid rollout")
    for key in ("ttft_p95_s", "tpot_p95_s", "kv_peak_fraction",
                "peak_rps", "safe_rps_per_replica", "gpu_cost_usd",
                "max_cost_per_million_good_tokens"):
        x = getattr(e, key)
        if not isinstance(x, (int, float)) or not math.isfinite(x) or x < 0:
            raise ValueError(f"invalid {key}")
    if e.safe_rps_per_replica <= 0 or e.kv_peak_fraction > 1:
        raise ValueError("invalid capacity or KV fraction")
    if type(e.quality_pass) is not bool:
        raise ValueError("invalid quality flag")
    price = Decimal(str(e.gpu_cost_usd))
    cost = price * Decimal(1_000_000) / e.good_output_tokens if e.good_output_tokens else None
    gates = {
        "good_ratio": e.good / e.offered >= .99,
        "ttft": e.ttft_p95_s <= 2.0,
        "tpot": e.tpot_p95_s <= .15,
        "kv_headroom": e.kv_peak_fraction <= .90,
        "no_oom": e.oom_events == 0,
        "safety": e.critical_safety_failures == 0,
        "quality": e.quality_pass,
        "n_minus_domain": (e.ready_replicas - e.failure_domain_replicas)
                          * e.safe_rps_per_replica >= e.peak_rps,
        "rollout": (e.ready_replicas - e.rollout_max_unavailable)
                   * e.safe_rps_per_replica >= e.peak_rps,
        "unit_cost": cost is not None and cost <= Decimal(str(e.max_cost_per_million_good_tokens)),
    }
    return {"approved": all(gates.values()), "gates": gates,
            "failed_gates": [k for k, v in gates.items() if not v],
            "cost_per_million_good_tokens_usd": str(cost) if cost is not None else None,
            "good_ratio": e.good / e.offered}

def sample():
    return Evidence(
        offered=1000, completed=998, good=995,
        ttft_p95_s=1.2, tpot_p95_s=.09, kv_peak_fraction=.75,
        oom_events=0, critical_safety_failures=0, quality_pass=True,
        peak_rps=90, safe_rps_per_replica=15, ready_replicas=7,
        failure_domain_replicas=1, rollout_max_unavailable=1,
        gpu_cost_usd=6.0, good_output_tokens=100000,
        max_cost_per_million_good_tokens=75.0)

if __name__ == "__main__":
    result = evaluate(sample())
    print(json.dumps(result, indent=2))
```

## 3. Automated tests
Save as `test_acceptance.py`:
```python
from dataclasses import replace
import pytest
from acceptance import evaluate, sample

def test_healthy_approved():
    assert evaluate(sample())["approved"]

def test_good_ratio():
    assert not evaluate(replace(sample(), good=980))["gates"]["good_ratio"]

def test_ttft():
    assert not evaluate(replace(sample(), ttft_p95_s=3))["gates"]["ttft"]

def test_tpot():
    assert not evaluate(replace(sample(), tpot_p95_s=.2))["gates"]["tpot"]

def test_kv():
    assert not evaluate(replace(sample(), kv_peak_fraction=.95))["gates"]["kv_headroom"]

def test_oom():
    assert not evaluate(replace(sample(), oom_events=1))["gates"]["no_oom"]

def test_safety():
    assert not evaluate(replace(sample(), critical_safety_failures=1))["gates"]["safety"]

def test_quality():
    assert not evaluate(replace(sample(), quality_pass=False))["gates"]["quality"]

def test_n_minus_domain():
    assert not evaluate(replace(sample(), failure_domain_replicas=2))["gates"]["n_minus_domain"]

def test_rollout():
    assert not evaluate(replace(sample(), rollout_max_unavailable=2))["gates"]["rollout"]

def test_unit_cost():
    assert not evaluate(replace(sample(), gpu_cost_usd=9))["gates"]["unit_cost"]

def test_no_good_tokens():
    r = evaluate(replace(sample(), good=0, good_output_tokens=0))
    assert not r["gates"]["unit_cost"]
    assert r["cost_per_million_good_tokens_usd"] is None

@pytest.mark.parametrize("change", [
    {"offered": 0},
    {"completed": 1001},
    {"good": 999},
    {"ready_replicas": -1},
    {"failure_domain_replicas": 8},
    {"safe_rps_per_replica": 0},
    {"kv_peak_fraction": 1.1},
    {"ttft_p95_s": float("nan")},
])
def test_invalid_evidence(change):
    with pytest.raises(ValueError):
        evaluate(replace(sample(), **change))
```

Run:
```bash
python acceptance.py
python -m pytest -q
```
Expected: **20 pytest cases** (12 individual tests and 8 parameterized cases). Actual execution not yet verified.

## 4. Failure drills
1. Break each gate independently and confirm fail-closed behavior.
2. Simulate partial stream losses by reducing `completed`; compare offered versus good denominators.
3. Increase KV occupancy to 95%; require rollback or capacity remediation.
4. Model two replicas lost in a shared failure domain.
5. Model a rollout that temporarily removes two replicas.
6. Raise GPU-hour cost and verify unit-cost failure.
7. Set `good_output_tokens=0`; ensure cost per million good tokens is undefined rather than zero.
8. Discuss why the synthetic engine cannot prove real GPU throughput, actual p95 measurement validity, model safety or cold-start recovery.

## 5. Final capstone scorecard
| Domain | Points |
|---|---:|
| Reproducible evidence and accounting | 15 |
| SLO, goodput and tail latency | 15 |
| KV, OOM and quality gates | 15 |
| Cold start and readiness planning | 10 |
| Load/stress/soak/failure evidence | 15 |
| N-1 and rollout headroom | 15 |
| Cost, security and runbooks | 15 |
| **Total** | **100** |

Educational pass >=90/100 with no critical failed gate. Production sign-off requires separate real staging evidence, owner authorization, security review and rollback drill.
