# Chapter 066 — Executable Inference Benchmark Design and Traffic Cohorts Lab

**Status:** Published; live execution pending. **Safety:** synthetic offline simulation only; no model server or external network.

## 1. Prerequisites
Python 3.11+; pytest for tests.

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install pytest
```
PowerShell: `.venv\Scripts\Activate.ps1`.

## 2. Deterministic cohort benchmark simulator
Save as `benchmark.py`:

```python
from dataclasses import dataclass
from collections import defaultdict
import math
import random
import json

@dataclass(frozen=True)
class Cohort:
    name: str
    input_tokens: int
    output_tokens: int
    arrival_rate_rps: float
    count: int

@dataclass(frozen=True)
class Result:
    cohort: str
    arrival_s: float
    first_token_s: float | None
    finish_s: float
    output_tokens: int
    ok: bool

def percentile(values, p):
    if not values:
        return None
    values = sorted(values)
    index = math.ceil(p * len(values)) - 1
    return values[max(0, index)]

def simulate(cohorts, seed=42):
    rng = random.Random(seed)
    results = []
    for c in cohorts:
        if c.count <= 0 or c.input_tokens <= 0 or c.output_tokens <= 0:
            raise ValueError("invalid cohort size or tokens")
        if not math.isfinite(c.arrival_rate_rps) or c.arrival_rate_rps <= 0:
            raise ValueError("invalid arrival rate")
        # Independent open-loop Poisson arrivals, synthetic queue approximation.
        arrivals = []
        t = 0.0
        for _ in range(c.count):
            t += rng.expovariate(c.arrival_rate_rps)
            arrivals.append(t)
        worker_available = 0.0
        for i, arrival in enumerate(arrivals):
            service_start = max(arrival, worker_available)
            prefill_s = 0.02 + c.input_tokens / 20000
            decode_s = c.output_tokens / 80
            first_token = service_start + prefill_s
            finish = first_token + decode_s
            worker_available = finish
            ok = i % 20 != 19  # 5% deterministic synthetic failures
            results.append(Result(c.name, arrival, first_token if ok else None,
                                  finish, c.output_tokens if ok else 0, ok))
    return results

def summarize(results, ttft_budget_s=2.0, e2e_budget_s=10.0):
    grouped = defaultdict(list)
    for r in results:
        grouped[r.cohort].append(r)
    summary = {}
    for cohort, rows in grouped.items():
        arrivals = [r.arrival_s for r in rows]
        finishes = [r.finish_s for r in rows]
        elapsed = max(finishes) - min(arrivals)
        ttft = [r.first_token_s - r.arrival_s for r in rows if r.ok and r.first_token_s is not None]
        e2e = [r.finish_s - r.arrival_s for r in rows if r.ok]
        good = [r for r in rows if r.ok and r.first_token_s is not None
                and r.first_token_s - r.arrival_s <= ttft_budget_s
                and r.finish_s - r.arrival_s <= e2e_budget_s]
        summary[cohort] = {
            "attempts": len(rows),
            "errors": sum(not r.ok for r in rows),
            "error_ratio": sum(not r.ok for r in rows) / len(rows),
            "achieved_rps": sum(r.ok for r in rows) / elapsed,
            "output_tokens_per_s": sum(r.output_tokens for r in rows if r.ok) / elapsed,
            "goodput_rps": len(good) / elapsed,
            "ttft_p95_s": percentile(ttft, 0.95),
            "e2e_p95_s": percentile(e2e, 0.95),
        }
    return summary

def tpot(first_token_s, last_token_s, output_tokens):
    if output_tokens <= 1 or last_token_s < first_token_s:
        return None
    return (last_token_s - first_token_s) / (output_tokens - 1)

if __name__ == "__main__":
    cohorts = [
        Cohort("short", 128, 32, 2.0, 100),
        Cohort("long-context", 8192, 256, 0.5, 100),
    ]
    print(json.dumps(summarize(simulate(cohorts)), indent=2, sort_keys=True))
```

This is a **toy single-worker queue model**, not a vLLM/Triton scheduler model. It is suitable for methodology exercises, not hardware sizing.

## 3. Automated tests
Save as `test_benchmark.py`:

```python
import pytest
from benchmark import Cohort, simulate, summarize, percentile, tpot

COHORTS = [Cohort("short", 128, 32, 2, 40), Cohort("long", 8192, 256, 0.5, 40)]

def test_reproducible_seed():
    assert simulate(COHORTS, 7) == simulate(COHORTS, 7)

def test_cohort_isolation():
    out = summarize(simulate(COHORTS))
    assert set(out) == {"short", "long"}
    assert out["short"]["attempts"] == 40
    assert out["long"]["attempts"] == 40

def test_failures_counted():
    out = summarize(simulate(COHORTS))
    assert out["short"]["errors"] == 2
    assert out["long"]["errors"] == 2

def test_goodput_bounded():
    for row in summarize(simulate(COHORTS)).values():
        assert 0 <= row["goodput_rps"] <= row["achieved_rps"]

def test_p95_nearest_rank():
    assert percentile([1, 2, 3, 4, 5, 6, 7, 8, 9, 10], 0.95) == 10

def test_tpot():
    assert tpot(1, 5, 5) == 1
    assert tpot(1, 1, 1) is None

@pytest.mark.parametrize("cohort", [
    Cohort("bad-count", 128, 32, 2, 0),
    Cohort("bad-input", 0, 32, 2, 10),
    Cohort("bad-output", 128, 0, 2, 10),
    Cohort("bad-rate", 128, 32, 0, 10),
])
def test_invalid_cohort(cohort):
    with pytest.raises(ValueError):
        simulate([cohort])

def test_long_context_higher_ttft():
    out = summarize(simulate([Cohort("short", 128, 32, 0.01, 20),
                              Cohort("long", 8192, 32, 0.01, 20)], seed=1))
    assert out["long"]["ttft_p95_s"] > out["short"]["ttft_p95_s"]
```

Run:
```bash
python benchmark.py
python -m pytest -q
```
Expected: **11 passing tests** (7 standalone + 4 parameterized).

## 4. Cohort and load experiments
1. Increase short-cohort offered load from 2 to 10 RPS; compare p95 TTFT, p95 end-to-end, achieved throughput and goodput.
2. Hold arrival rate constant; increase output length from 32 to 256. Observe decode occupancy and queue buildup.
3. Compare short and long-context cohorts. The synthetic prefill penalty grows with input tokens.
4. Increase failure frequency by changing the deterministic `ok` rule in a disposable copy; verify throughput and goodput differ.
5. Change the TTFT budget from 2.0 to 0.5 seconds and measure goodput loss.
6. Compare independent open-loop arrivals to a conceptual closed-loop driver; document why closed-loop can under-sample overload.
7. Document how results would differ with real continuous batching, multiple workers and GPU contention.

## 5. Benchmark evidence template
Record: simulator version, Python version, seed, cohort weights, arrival model, concurrency assumptions, time window, latency budgets, attempts, errors, throughput, goodput, p95 TTFT and p95 E2E. Mark synthetic results explicitly.

## 6. 100-point review rubric
| Dimension | Points |
|---|---:|
| Cohort representativeness and privacy | 20 |
| Arrival model and experimental controls | 15 |
| TTFT/TPOT/E2E measurement correctness | 20 |
| Throughput/goodput and error accounting | 15 |
| Saturation and failure experiments | 15 |
| Evidence, cost caps and safety | 15 |
| **Total** | **100** |

**Pass recommendation:** >=90 with no critical privacy or unsafe load-generation gaps. Production benchmarking requires separate staging execution, capacity headroom and change approval.

## 7. Cleanup
Deactivate virtual environment and remove disposable lab files only after saving sanitized test output. No real customer data or credentials should be used.
