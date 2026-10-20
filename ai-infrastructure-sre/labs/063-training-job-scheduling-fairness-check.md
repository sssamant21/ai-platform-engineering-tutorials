# Lab 063 — Training Job Scheduling and Fairness Gate

**Python 3.11+ · pytest · offline, fail-closed policy.** No scheduler or cloud API calls.

## File: `scheduling_fairness_policy.py`

```python
from dataclasses import dataclass
import json
import math

@dataclass(frozen=True)
class Plan:
    queue: str
    nodes: int
    gpus_per_node: int
    largest_failure_domain_gpus: int
    reserved_gpus: int
    admitted_batch_gpus: int
    gang_job_gpus: int
    queue_wait_p95_min: float
    queue_wait_slo_min: float
    max_fairshare_deviation: float
    observed_fairshare_deviation: float
    checkpoint_age_min: float
    checkpoint_rpo_min: float
    preemption_grace_min: float
    checkpoint_commit_min: float
    telemetry_age_s: float
    telemetry_max_age_s: float
    queue_ownership_defined: bool
    gang_admission_verified: bool
    gpu_quota_verified: bool
    tenant_isolation_verified: bool
    priority_classes_reviewed: bool
    fair_share_reviewed: bool
    preemption_grace_verified: bool
    checkpoint_before_eviction_verified: bool
    backfill_policy_reviewed: bool
    reservation_policy_reviewed: bool
    node_topology_verified: bool
    gpu_resource_inventory_verified: bool
    autoscaler_bounds_verified: bool
    cloud_quota_verified: bool
    budget_admission_verified: bool
    starvation_alerts_verified: bool
    queue_metrics_verified: bool
    gpu_utilization_verified: bool
    incident_runbooks_tested: bool
    rollback_verified: bool
    independent_review_approved: bool

def evaluate(p):
    if not isinstance(p, Plan):
        raise TypeError("Plan required")
    if not isinstance(p.queue, str) or not p.queue.strip():
        return {"accepted": False, "reason": "missing_queue"}
    counts = (p.nodes, p.gpus_per_node, p.largest_failure_domain_gpus,
              p.reserved_gpus, p.admitted_batch_gpus, p.gang_job_gpus)
    if any(type(x) is not int or x < 0 for x in counts) or min(p.nodes, p.gpus_per_node, p.largest_failure_domain_gpus, p.gang_job_gpus) == 0:
        return {"accepted": False, "reason": "invalid_count"}
    metrics = (p.queue_wait_p95_min, p.queue_wait_slo_min,
               p.max_fairshare_deviation, p.observed_fairshare_deviation,
               p.checkpoint_age_min, p.checkpoint_rpo_min,
               p.preemption_grace_min, p.checkpoint_commit_min,
               p.telemetry_age_s, p.telemetry_max_age_s)
    if (any(type(x) not in (int, float) or not math.isfinite(x) or x < 0 for x in metrics)
            or min(p.queue_wait_slo_min, p.checkpoint_rpo_min, p.telemetry_max_age_s) <= 0):
        return {"accepted": False, "reason": "invalid_metric"}
    total = p.nodes * p.gpus_per_node
    degraded = total - p.largest_failure_domain_gpus - p.reserved_gpus
    gates = {
        "failure_domain_geometry": p.largest_failure_domain_gpus <= total,
        "reserved_capacity": p.reserved_gpus <= total - p.largest_failure_domain_gpus,
        "n_minus_one_admission": p.admitted_batch_gpus <= degraded,
        "gang_capacity": p.gang_job_gpus <= degraded,
        "queue_wait_slo": p.queue_wait_p95_min <= p.queue_wait_slo_min,
        "fairshare_deviation": p.observed_fairshare_deviation <= p.max_fairshare_deviation,
        "checkpoint_rpo": p.checkpoint_age_min <= p.checkpoint_rpo_min,
        "preemption_checkpoint": p.checkpoint_commit_min <= p.preemption_grace_min,
        "telemetry_freshness": p.telemetry_age_s <= p.telemetry_max_age_s,
        "queue_ownership_defined": p.queue_ownership_defined is True,
        "gang_admission_verified": p.gang_admission_verified is True,
        "gpu_quota_verified": p.gpu_quota_verified is True,
        "tenant_isolation_verified": p.tenant_isolation_verified is True,
        "priority_classes_reviewed": p.priority_classes_reviewed is True,
        "fair_share_reviewed": p.fair_share_reviewed is True,
        "preemption_grace_verified": p.preemption_grace_verified is True,
        "checkpoint_before_eviction_verified": p.checkpoint_before_eviction_verified is True,
        "backfill_policy_reviewed": p.backfill_policy_reviewed is True,
        "reservation_policy_reviewed": p.reservation_policy_reviewed is True,
        "node_topology_verified": p.node_topology_verified is True,
        "gpu_resource_inventory_verified": p.gpu_resource_inventory_verified is True,
        "autoscaler_bounds_verified": p.autoscaler_bounds_verified is True,
        "cloud_quota_verified": p.cloud_quota_verified is True,
        "budget_admission_verified": p.budget_admission_verified is True,
        "starvation_alerts_verified": p.starvation_alerts_verified is True,
        "queue_metrics_verified": p.queue_metrics_verified is True,
        "gpu_utilization_verified": p.gpu_utilization_verified is True,
        "incident_runbooks_tested": p.incident_runbooks_tested is True,
        "rollback_verified": p.rollback_verified is True,
        "independent_review_approved": p.independent_review_approved is True,
    }
    failed = sorted(k for k, ok in gates.items() if not ok)
    return {"accepted": not failed, "failed_gates": failed,
            "total_gates": len(gates), "total_gpus": total,
            "n_minus_one_batch_gpus": degraded}

def example():
    return Plan("synthetic-gpu-batch", 8, 8, 8, 16, 32, 16,
                10.0, 20.0, 0.2, 0.1, 5.0, 30.0,
                10.0, 5.0, 10.0, 120.0, *([True] * 21))

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```

## File: `test_scheduling_fairness_policy.py`

```python
from dataclasses import replace
import pytest
from scheduling_fairness_policy import evaluate, example

def test_baseline():
    assert evaluate(example())["accepted"]

def test_type():
    with pytest.raises(TypeError):
        evaluate({})

def test_missing_queue():
    assert evaluate(replace(example(), queue=" "))["reason"] == "missing_queue"

def test_bool_count():
    assert evaluate(replace(example(), nodes=True))["reason"] == "invalid_count"

def test_negative_count():
    assert evaluate(replace(example(), reserved_gpus=-1))["reason"] == "invalid_count"

def test_nan():
    assert evaluate(replace(example(), queue_wait_p95_min=float("nan")))["reason"] == "invalid_metric"

def test_inf():
    assert evaluate(replace(example(), telemetry_age_s=float("inf")))["reason"] == "invalid_metric"

def test_bool_metric():
    assert evaluate(replace(example(), checkpoint_rpo_min=True))["reason"] == "invalid_metric"

def test_gate_count():
    assert evaluate(example())["total_gates"] == 30

def test_degraded_capacity():
    assert evaluate(example())["n_minus_one_batch_gpus"] == 40

@pytest.mark.parametrize("field,gate,value", [
    ("largest_failure_domain_gpus", "failure_domain_geometry", 65),
    ("reserved_gpus", "reserved_capacity", 57),
    ("admitted_batch_gpus", "n_minus_one_admission", 41),
    ("gang_job_gpus", "gang_capacity", 41),
    ("queue_wait_p95_min", "queue_wait_slo", 21.0),
    ("observed_fairshare_deviation", "fairshare_deviation", 0.3),
    ("checkpoint_age_min", "checkpoint_rpo", 31.0),
    ("checkpoint_commit_min", "preemption_checkpoint", 11.0),
    ("telemetry_age_s", "telemetry_freshness", 121.0),
])
def test_numeric_gate(field, gate, value):
    assert gate in evaluate(replace(example(), **{field: value}))["failed_gates"]

@pytest.mark.parametrize("field,gate", [
    ("queue_ownership_defined", "queue_ownership_defined"),
    ("gang_admission_verified", "gang_admission_verified"),
    ("gpu_quota_verified", "gpu_quota_verified"),
    ("tenant_isolation_verified", "tenant_isolation_verified"),
    ("priority_classes_reviewed", "priority_classes_reviewed"),
    ("fair_share_reviewed", "fair_share_reviewed"),
    ("preemption_grace_verified", "preemption_grace_verified"),
    ("checkpoint_before_eviction_verified", "checkpoint_before_eviction_verified"),
    ("backfill_policy_reviewed", "backfill_policy_reviewed"),
    ("reservation_policy_reviewed", "reservation_policy_reviewed"),
    ("node_topology_verified", "node_topology_verified"),
    ("gpu_resource_inventory_verified", "gpu_resource_inventory_verified"),
    ("autoscaler_bounds_verified", "autoscaler_bounds_verified"),
    ("cloud_quota_verified", "cloud_quota_verified"),
    ("budget_admission_verified", "budget_admission_verified"),
    ("starvation_alerts_verified", "starvation_alerts_verified"),
    ("queue_metrics_verified", "queue_metrics_verified"),
    ("gpu_utilization_verified", "gpu_utilization_verified"),
    ("incident_runbooks_tested", "incident_runbooks_tested"),
    ("rollback_verified", "rollback_verified"),
    ("independent_review_approved", "independent_review_approved"),
])
def test_boolean_gate(field, gate):
    assert gate in evaluate(replace(example(), **{field: False}))["failed_gates"]
```

Run locally:

```bash
python -m pip install pytest
python scheduling_fairness_policy.py
python -m pytest -q
```

**40 planned pytest cases:** 10 standalone + 9 numeric + 21 Boolean. Tests and live scheduling acceptance have not been executed.

## Ten synthetic incident exercises

1. Gang job partially admitted; evaluate relevant gates and preserve evidence.
2. High-priority job starves research queue; evaluate relevant gates and preserve evidence.
3. GPU quota allows tenant monopoly; evaluate relevant gates and preserve evidence.
4. Karpenter cannot provision requested accelerator; evaluate relevant gates and preserve evidence.
5. Slurm fair-share weights drift; evaluate relevant gates and preserve evidence.
6. Backfill reservation delays critical job; evaluate relevant gates and preserve evidence.
7. Preemption kills uncommitted checkpoint; evaluate relevant gates and preserve evidence.
8. Topology fragmentation prevents placement; evaluate relevant gates and preserve evidence.
9. Cloud GPU quota exhausted; evaluate relevant gates and preserve evidence.
10. Scheduler telemetry stale during queue spike; evaluate relevant gates and preserve evidence.

## 100-point review

Queue architecture 20; fairness and quotas 20; capacity and checkpoint 20; SLOs/security/FinOps 20; staging validation and independent review 20. Production acceptance requires live evidence and no critical gate failure.
