# Chapter 073 — Executable Load, Stress, Soak and Failure Testing Lab

**Status:** Published; live execution pending. **Scope:** deterministic local simulator; no GPU, cloud endpoint or production traffic.

## 1. Prerequisites
Python 3.11+ and pytest:
```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install pytest
```
PowerShell: `.venv\Scripts\Activate.ps1`.

## 2. Deterministic benchmark harness
Save as `load_lab.py`:
```python
from dataclasses import dataclass
import heapq
import json
import math

@dataclass(frozen=True)
class Config:
    workers: int = 2
    service_s: float = 0.2
    queue_limit: int = 8
    deadline_s: float = 2.0
    slo_s: float = 1.0
    max_rejection_ratio: float = 0.10

def arrivals(phases):
    """Phases are (duration_s, offered_rps) pairs with deterministic spacing."""
    out, offset = [], 0.0
    for duration, rate in phases:
        if not all(math.isfinite(x) and x > 0 for x in (duration, rate)):
            raise ValueError("invalid phase")
        count = math.floor(duration * rate)
        out.extend(offset + i / rate for i in range(count))
        offset += duration
    return out

def run(schedule, cfg=Config(), failed_workers=0):
    if type(cfg.workers) is not int or cfg.workers <= 0:
        raise ValueError("invalid workers")
    if type(failed_workers) is not int or not 0 <= failed_workers < cfg.workers:
        raise ValueError("invalid failure count")
    if type(cfg.queue_limit) is not int or cfg.queue_limit < 0:
        raise ValueError("invalid queue limit")
    if any(not math.isfinite(x) or x <= 0 for x in
           (cfg.service_s, cfg.deadline_s, cfg.slo_s)):
        raise ValueError("invalid timing")
    if not 0 <= cfg.max_rejection_ratio <= 1:
        raise ValueError("invalid rejection threshold")
    if any(not math.isfinite(t) or t < 0 for t in schedule):
        raise ValueError("invalid schedule")
    if list(schedule) != sorted(schedule):
        raise ValueError("unsorted schedule")
    workers = [0.0] * (cfg.workers - failed_workers)
    heapq.heapify(workers)
    pending_starts = []
    rows = []
    for t in schedule:
        pending_starts = [x for x in pending_starts if x > t]
        start = max(t, workers[0])
        wait = start - t
        if wait > 0 and len(pending_starts) >= cfg.queue_limit:
            rows.append(("rejected_queue", None))
            continue
        if wait + cfg.service_s > cfg.deadline_s:
            rows.append(("rejected_deadline", None))
            continue
        heapq.heapreplace(workers, start + cfg.service_s)
        if wait > 0:
            pending_starts.append(start)
        latency = wait + cfg.service_s
        rows.append(("good" if latency <= cfg.slo_s else "slow", latency))
    return rows

def report(rows, cfg=Config()):
    if not rows:
        raise ValueError("empty results")
    n = len(rows)
    rejected = sum(state.startswith("rejected") for state, _ in rows)
    good = sum(state == "good" for state, _ in rows)
    latencies = sorted(v for _, v in rows if v is not None)
    p95 = latencies[math.ceil(len(latencies) * .95) - 1] if latencies else None
    return {"offered": n, "completed": n - rejected,
            "rejected": rejected, "good": good,
            "good_ratio": good / n, "rejection_ratio": rejected / n,
            "latency_p95_s": p95,
            "abort": rejected / n > cfg.max_rejection_ratio}

if __name__ == "__main__":
    for name, phases in {
        "baseline": [(10, 2)],
        "load": [(10, 8)],
        "stress": [(5, 5), (5, 10), (5, 20)],
        "spike": [(5, 2), (1, 40), (5, 2)],
        "soak_sample": [(30, 5)],
    }.items():
        print(name, json.dumps(report(run(arrivals(phases))), sort_keys=True))
    print("N-1", json.dumps(report(run(arrivals([(10, 8)]), failed_workers=1)), sort_keys=True))
```

**Simulation limitations:** fixed service time, no prefill/decode separation, no GPU, KV-cache, retries, autoscaling or actual time passage. The “soak_sample” is a virtual schedule, not a real soak test. A fixed failure count applies to the entire run; it does not model mid-run worker loss.

## 3. Automated tests
Save as `test_load_lab.py`:
```python
import pytest
from load_lab import Config, arrivals, run, report

def test_schedule_count():
    assert len(arrivals([(10, 2), (5, 4)])) == 40

def test_schedule_sorted():
    assert arrivals([(2, 2), (2, 2)]) == [0, .5, 2, 2.5]

def test_baseline_good():
    r = report(run(arrivals([(10, 2)])))
    assert r["good"] == r["offered"]

def test_overload_rejections():
    r = report(run(arrivals([(10, 40)])))
    assert r["rejected"] > 0

def test_abort_gate():
    cfg = Config(max_rejection_ratio=0)
    assert report(run(arrivals([(10, 40)]), cfg), cfg)["abort"]

def test_n_minus_one_degrades():
    schedule = arrivals([(10, 8)])
    normal = report(run(schedule))
    failed = report(run(schedule, failed_workers=1))
    assert failed["good"] < normal["good"]

def test_accounting():
    r = report(run(arrivals([(5, 20)])))
    assert r["completed"] + r["rejected"] == r["offered"]

def test_reproducible():
    schedule = arrivals([(5, 5)])
    assert run(schedule) == run(schedule)

@pytest.mark.parametrize("phases", [[(0, 1)], [(1, 0)], [(1, -1)]])
def test_invalid_phases(phases):
    with pytest.raises(ValueError):
        arrivals(phases)

def test_invalid_failure_count():
    with pytest.raises(ValueError):
        run([0], failed_workers=2)

def test_empty_report():
    with pytest.raises(ValueError):
        report([])
```

Run:
```bash
python load_lab.py
python -m pytest -q
```
Expected: **13 pytest cases** (10 standalone and 3 parameterized); execution not yet performed.

## 4. Experiments and failure drills
1. **Baseline:** 2 RPS; record good ratio and p95 completion latency.
2. **Load:** 8 RPS; compare against 2 RPS.
3. **Stress:** ramp 5 → 10 → 20 RPS; identify first SLO/abort threshold.
4. **Spike:** 40 RPS for one simulated second; observe queue rejection.
5. **Virtual soak:** hold 5 RPS over a longer synthetic schedule; discuss why this cannot detect real memory leaks.
6. **N-1:** compare two workers to one at identical offered load.
7. **Deadline:** reduce deadline and verify early rejection.
8. **Recovery:** compare separate healthy and N-1 runs; note lack of dynamic recovery modeling.

## 5. Evidence and 100-point acceptance
Record Python version, pytest output, schedule, worker capacity, queue cap, SLO, offered/completed/rejected counts, p95 latency and abort decisions.

| Criterion | Points |
|---|---:|
| Workload phases and open-loop schedule | 20 |
| Accounting and SLO goodput | 20 |
| Stress/spike abort and bounded admission | 20 |
| N-1 and failure-drill evidence | 20 |
| Safety, runbooks and limitations | 20 |
| **Total** | **100** |

Recommended educational acceptance >=90/100. Real soak testing, GPU capacity acceptance and failure recovery sign-off require a separate approved staging exercise.
