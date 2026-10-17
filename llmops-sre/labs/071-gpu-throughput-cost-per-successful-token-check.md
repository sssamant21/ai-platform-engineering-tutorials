# Chapter 071 — Executable GPU Throughput and Cost per Successful Token Lab

**Status:** Published; execution/reviews pending. Synthetic pricing and traffic only.

## Setup
Python 3.11+; `python -m pip install pytest`.

Save as `gpu_cost.py`:
```python
from dataclasses import dataclass
from decimal import Decimal

@dataclass(frozen=True)
class Request:
    output_tokens: int
    complete: bool
    within_slo: bool

def cost_report(requests, gpu_count, billed_hours, usd_per_gpu_hour):
    if type(gpu_count) is not int or gpu_count < 1:
        raise ValueError("invalid GPU count")
    hours, price = Decimal(str(billed_hours)), Decimal(str(usd_per_gpu_hour))
    if not hours.is_finite() or not price.is_finite() or hours <= 0 or price < 0:
        raise ValueError("invalid billing inputs")
    if not requests:
        raise ValueError("empty traffic sample")
    for r in requests:
        if type(r.output_tokens) is not int or r.output_tokens < 0:
            raise ValueError("invalid tokens")
        if type(r.complete) is not bool or type(r.within_slo) is not bool:
            raise ValueError("invalid status")
    completed = [r for r in requests if r.complete]
    good = [r for r in completed if r.within_slo]
    successful_tokens = sum(r.output_tokens for r in completed)
    good_tokens = sum(r.output_tokens for r in good)
    gpu_cost = gpu_count * hours * price
    return {
        "attempted": len(requests), "completed": len(completed),
        "good_requests": len(good), "successful_tokens": successful_tokens,
        "good_tokens": good_tokens, "gpu_cost_usd": gpu_cost,
        "cost_per_million_good_tokens_usd": gpu_cost * Decimal(1000000) / good_tokens if good_tokens else None,
        "good_tokens_per_gpu_hour": Decimal(good_tokens) / (gpu_count * hours),
    }

if __name__ == "__main__":
    sample = [Request(100,True,True), Request(200,True,False), Request(80,False,False)]
    print(cost_report(sample, 2, 1, "3.00"))
```

Save as `test_gpu_cost.py`:
```python
import pytest
from decimal import Decimal
from gpu_cost import Request, cost_report
S = [Request(100,True,True), Request(200,True,False), Request(80,False,False)]

def test_accounting():
    r=cost_report(S,2,1,3)
    assert (r["attempted"],r["completed"],r["good_requests"]) == (3,2,1)
def test_good_tokens_exclude_failures():
    assert cost_report(S,2,1,3)["good_tokens"] == 100
def test_successful_tokens_include_slo_miss():
    assert cost_report(S,2,1,3)["successful_tokens"] == 300
def test_cost_includes_idle_replica():
    assert cost_report(S,2,1,3)["gpu_cost_usd"] == Decimal("6")
def test_unit_cost():
    assert cost_report(S,2,1,3)["cost_per_million_good_tokens_usd"] == Decimal("60000")
def test_no_good_tokens_is_unknown():
    assert cost_report([Request(100,False,False)],1,1,3)["cost_per_million_good_tokens_usd"] is None
@pytest.mark.parametrize("count,hours,price", [(0,1,3),(1,0,3),(1,1,-1),(1,float("nan"),3)])
def test_invalid_billing(count,hours,price):
    with pytest.raises(ValueError): cost_report(S,count,hours,price)
def test_invalid_tokens():
    with pytest.raises(ValueError): cost_report([Request(-1,True,True)],1,1,3)
def test_empty_sample():
    with pytest.raises(ValueError): cost_report([],1,1,3)
```

Run `python gpu_cost.py && python -m pytest -q` (PowerShell: separate commands). **Expected: 12 collected cases**; actual execution not yet performed.

## Drills and acceptance
Double billed GPU-hours while keeping good tokens fixed; observe unit cost double. Reclassify completed requests as SLO misses; observe denominator collapse. Introduce partial stream failures; confirm no good-token credit. Simulate one standby replica and compare N-1 safety versus cost. Review rubric: accounting 20, SLO denominator 20, capacity controls 20, price assumptions 20, incident readiness 20 (100 total). Real billing, GPU telemetry and production traffic are separate acceptance requirements.
