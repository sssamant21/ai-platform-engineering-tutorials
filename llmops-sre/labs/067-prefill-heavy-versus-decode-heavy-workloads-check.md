# Chapter 067 — Executable Prefill-Heavy Versus Decode-Heavy Workloads Lab

**Status:** Published; execution pending. **Scope:** deterministic offline phase/queue simulation; no model, GPU, cloud endpoint or production traffic.

## 1. Setup
Python 3.11+ and pytest:
```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install pytest
```
PowerShell: `.venv\Scripts\Activate.ps1`.

## 2. Synthetic phase simulator
Create `phase_benchmark.py`:

```python
from dataclasses import dataclass
from collections import defaultdict
import math
import json

@dataclass(frozen=True)
class Cohort:
    name: str
    input_tokens: int
    output_tokens: int
    arrival_rate: float
    count: int

@dataclass(frozen=True)
class Observation:
    cohort: str
    arrival: float
    queue_s: float
    prefill_s: float
    ttft_s: float
    tpot_s: float | None
    e2e_s: float

def p95(values):
    if not values:
        return None
    values = sorted(values)
    return values[math.ceil(len(values) * 0.95) - 1]

def simulate(cohorts, prefill_tps=20000.0, decode_tps=80.0):
    if not all(math.isfinite(v) and v > 0 for v in (prefill_tps, decode_tps)):
        raise ValueError("invalid service rates")
    observations = []
    for c in cohorts:
        if not c.name or type(c.input_tokens) is not int or type(c.output_tokens) is not int:
            raise ValueError("invalid cohort")
        if c.input_tokens <= 0 or c.output_tokens <= 0 or type(c.count) is not int or c.count <= 0:
            raise ValueError("invalid tokens or count")
        if not math.isfinite(c.arrival_rate) or c.arrival_rate <= 0:
            raise ValueError("invalid arrival rate")
        worker_free = 0.0
        for i in range(c.count):
            arrival = i / c.arrival_rate
            start = max(arrival, worker_free)
            queue_s = start - arrival
            prefill_s = 0.02 + c.input_tokens / prefill_tps
            decode_s = (c.output_tokens - 1) / decode_tps
            first = start + prefill_s
            finish = first + decode_s
            worker_free = finish
            observations.append(Observation(
                c.name, arrival, queue_s, prefill_s, first - arrival,
                1 / decode_tps if c.output_tokens > 1 else None,
                finish - arrival))
    return observations

def summarize(observations):
    groups = defaultdict(list)
    for row in observations:
        groups[row.cohort].append(row)
    out = {}
    for cohort, rows in groups.items():
        out[cohort] = {
            "count": len(rows),
            "queue_p95_s": p95([r.queue_s for r in rows]),
            "prefill_p95_s": p95([r.prefill_s for r in rows]),
            "ttft_p95_s": p95([r.ttft_s for r in rows]),
            "tpot_p95_s": p95([r.tpot_s for r in rows if r.tpot_s is not None]),
            "e2e_p95_s": p95([r.e2e_s for r in rows]),
        }
    return out

if __name__ == "__main__":
    cohorts = [
        Cohort("prefill-heavy", 8192, 32, 0.1, 40),
        Cohort("decode-heavy", 128, 1024, 0.1, 40),
        Cohort("balanced", 1024, 256, 0.1, 40),
    ]
    print(json.dumps(summarize(simulate(cohorts)), indent=2, sort_keys=True))
```

**Model limitation:** Each cohort runs on its own synthetic single worker; cohorts are not competing for the same scheduler. The simulator is useful for phase attribution and within-cohort queueing, not for mixed-workload contention, GPU efficiency or real token generation.

## 3. Automated tests
Create `test_phase_benchmark.py`:

```python
import pytest
from phase_benchmark import Cohort, simulate, summarize, p95

def test_prefill_heavy_longer_prefill():
    a = simulate([Cohort("short", 128, 32, 0.01, 5)])
    b = simulate([Cohort("long", 8192, 32, 0.01, 5)])
    assert b[0].prefill_s > a[0].prefill_s
    assert b[0].ttft_s > a[0].ttft_s

def test_decode_heavy_longer_completion():
    a = simulate([Cohort("short", 128, 32, 0.01, 5)])
    b = simulate([Cohort("decode", 128, 1024, 0.01, 5)])
    assert b[0].e2e_s > a[0].e2e_s
    assert b[0].prefill_s == a[0].prefill_s

def test_queue_grows_under_overload():
    rows = simulate([Cohort("overload", 128, 1024, 10.0, 10)])
    assert rows[-1].queue_s > rows[0].queue_s

def test_single_token_tpot_undefined():
    rows = simulate([Cohort("single", 128, 1, 0.01, 1)])
    assert rows[0].tpot_s is None

def test_repeatable():
    c = [Cohort("stable", 1024, 256, 0.1, 10)]
    assert simulate(c) == simulate(c)

def test_p95_nearest_rank():
    assert p95(list(range(1, 21))) == 19
    assert p95([]) is None

def test_summary_cohorts():
    rows = simulate([Cohort("a", 128, 32, 0.1, 4), Cohort("b", 8192, 32, 0.1, 4)])
    assert set(summarize(rows)) == {"a", "b"}

@pytest.mark.parametrize("cohort", [
    Cohort("bad", 0, 32, 1, 10),
    Cohort("bad", 128, 0, 1, 10),
    Cohort("bad", 128, 32, 0, 10),
    Cohort("bad", 128, 32, 1, 0),
])
def test_invalid_cohort(cohort):
    with pytest.raises(ValueError):
        simulate([cohort])

def test_invalid_service_rate():
    with pytest.raises(ValueError):
        simulate([Cohort("a", 128, 32, 1, 2)], decode_tps=0)
```

Run:
```bash
python phase_benchmark.py
python -m pytest -q
```
Expected: **12 passing pytest cases** (8 standalone tests plus four parameterized cases).

## 4. Experiments and failure drills
1. **Prefill sweep:** increase input tokens from 128 to 8,192 while holding output tokens fixed; compare prefill and TTFT.
2. **Decode sweep:** increase output tokens from 32 to 1,024; compare TPOT and end-to-end.
3. **Queue pressure:** raise arrival rate from 0.1 to 10 RPS; observe queue and TTFT growth.
4. **Slow prefill:** reduce `prefill_tps` by 50%; inspect sensitivity.
5. **Slow decode:** reduce `decode_tps` by 50%; inspect end-to-end and queue.
6. **Single-token response:** ensure TPOT is null, not zero.
7. **Invalid inputs:** verify tests reject zero tokens, zero arrivals and invalid service rates.

## 5. Evidence and acceptance
Record Python version, test results, cohort configuration, synthetic rates, p95 queue/prefill/TTFT/TPOT/E2E and limitations. Do not extrapolate the simulator to GPU sizing.

| Acceptance dimension | Points |
|---|---:|
| Cohort and measurement definitions | 20 |
| Phase attribution | 20 |
| Queue/saturation behavior | 20 |
| Deterministic tests and evidence | 20 |
| Security, cost and safety | 20 |
| **Total** | **100** |

Recommend >=90/100 for educational acceptance, with **separate** technical/security review and production-like benchmarking. Remove disposable lab files only after retaining sanitized evidence.
