# Chapter 002 — Executable Workload Taxonomy and Placement Acceptance Lab

**Status:** Published; local pytest execution and real GPU validation pending.

## 1. Setup
Python 3.11+ and pytest:
```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install pytest
```
PowerShell: `.venv\Scripts\Activate.ps1`.

## 2. Offline workload classifier and placement policy
Save as `workload_placement.py`:
```python
from dataclasses import dataclass
import json
import math

CLASSES = {"pretraining", "full_finetuning", "lora", "online_inference",
           "batch_inference", "embedding", "rag_agent"}
LATENCY_CRITICAL = {"online_inference", "rag_agent"}

@dataclass(frozen=True)
class Workload:
    kind: str
    requested_gpus: int
    estimated_vram_gib: float
    gpu_vram_gib: float
    free_gpus: int
    safety_fraction: float
    owner: str
    environment: str
    workload_identity: bool
    artifact_provenance_verified: bool
    data_residency_approved: bool
    tenant_isolation_verified: bool
    quota_approved: bool
    checkpoint_restore_tested: bool
    latency_slo_defined: bool
    idempotent_outputs: bool

def evaluate(w):
    if not isinstance(w, Workload):
        raise TypeError("Workload required")
    if w.kind not in CLASSES:
        return {"accepted": False, "reason": "unsupported_class"}
    if w.environment not in {"dev", "staging", "prod"} or not w.owner.strip():
        return {"accepted": False, "reason": "invalid_ownership"}
    if any(type(v) is not int or v < 0 for v in (w.requested_gpus, w.free_gpus)) or w.requested_gpus == 0:
        return {"accepted": False, "reason": "invalid_gpu_count"}
    vals = (w.estimated_vram_gib, w.gpu_vram_gib, w.safety_fraction)
    if any(type(v) not in (int, float) or not math.isfinite(v) for v in vals):
        return {"accepted": False, "reason": "invalid_memory"}
    if w.estimated_vram_gib <= 0 or w.gpu_vram_gib <= 0 or not 0 <= w.safety_fraction < 1:
        return {"accepted": False, "reason": "invalid_memory"}
    gates = {
        "gpu_count": w.requested_gpus <= w.free_gpus,
        "vram_headroom": w.estimated_vram_gib <= w.gpu_vram_gib * (1 - w.safety_fraction),
        "workload_identity": w.workload_identity is True,
        "provenance": w.artifact_provenance_verified is True,
        "residency": w.data_residency_approved is True,
        "tenant_isolation": w.tenant_isolation_verified is True,
        "quota": w.quota_approved is True,
        "recovery": w.checkpoint_restore_tested is True if w.kind in {"pretraining", "full_finetuning", "lora"} else True,
        "latency_slo": w.latency_slo_defined is True if w.kind in LATENCY_CRITICAL else True,
        "idempotency": w.idempotent_outputs is True if w.kind in {"batch_inference", "rag_agent"} else True,
    }
    failed = sorted(k for k, ok in gates.items() if not ok)
    return {"accepted": not failed, "failed_gates": failed,
            "class": w.kind, "vram_headroom_gib": round(w.gpu_vram_gib - w.estimated_vram_gib, 3)}

def example(kind="online_inference"):
    return Workload(kind, 2, 30.0, 80.0, 4, 0.2, "ai-platform-sre", "staging",
                    True, True, True, True, True, True, True, True)

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```
The per-GPU VRAM estimate assumes the model fits independently on each requested GPU. Real tensor/pipeline parallelism, training sharding, fragmentation, KV cache and runtime overhead require actual profiling; this is a **teaching policy evaluator**, not a GPU scheduler.

## 3. Pytest tests
Save as `test_workload_placement.py`:
```python
from dataclasses import replace
import pytest
from workload_placement import evaluate, example, CLASSES

@pytest.mark.parametrize("kind", sorted(CLASSES))
def test_supported_classes(kind):
    assert evaluate(example(kind))["accepted"]

def test_unsupported_class():
    assert evaluate(replace(example(), kind="unknown"))["reason"] == "unsupported_class"

def test_missing_owner():
    assert evaluate(replace(example(), owner=" "))["reason"] == "invalid_ownership"

def test_gpu_shortfall():
    assert "gpu_count" in evaluate(replace(example(), free_gpus=1))["failed_gates"]

def test_vram_shortfall():
    assert "vram_headroom" in evaluate(replace(example(), estimated_vram_gib=70))["failed_gates"]

def test_vram_boundary():
    assert evaluate(replace(example(), estimated_vram_gib=64))["accepted"]

@pytest.mark.parametrize("field", [
    "workload_identity", "artifact_provenance_verified", "data_residency_approved",
    "tenant_isolation_verified", "quota_approved",
])
def test_security_gates(field):
    assert not evaluate(replace(example(), **{field: False}))["accepted"]

@pytest.mark.parametrize("kind", ["pretraining", "full_finetuning", "lora"])
def test_training_recovery(kind):
    assert "recovery" in evaluate(replace(example(kind), checkpoint_restore_tested=False))["failed_gates"]

@pytest.mark.parametrize("kind", ["online_inference", "rag_agent"])
def test_latency_slo(kind):
    assert "latency_slo" in evaluate(replace(example(kind), latency_slo_defined=False))["failed_gates"]

@pytest.mark.parametrize("kind", ["batch_inference", "rag_agent"])
def test_idempotency(kind):
    assert "idempotency" in evaluate(replace(example(kind), idempotent_outputs=False))["failed_gates"]

def test_negative_gpu():
    assert evaluate(replace(example(), requested_gpus=-1))["reason"] == "invalid_gpu_count"

def test_nan_memory():
    assert evaluate(replace(example(), estimated_vram_gib=float("nan")))["reason"] == "invalid_memory"

def test_invalid_input():
    with pytest.raises(TypeError):
        evaluate({})
```
Run:
```bash
python workload_placement.py
python -m pytest -q
```
Expected: **27 test cases** (7 workload classes + 5 security gates + 3 recovery + 2 latency + 2 idempotency + 8 standalone). Execution is pending.

## 4. Staging validation
1. Record GPU type, free/allocated VRAM, driver, CUDA, NCCL and GPU Operator versions.
2. Profile online inference across context length, concurrency, TTFT/TPOT and stream completion.
3. Profile LoRA/full fine-tuning with synthetic data, optimizer states and checkpoint restore.
4. Test batch idempotency and RAG/tool ACLs with synthetic tenants.
5. Verify workload identity, artifact signatures, quota, topology, recovery and budget.
6. Reconcile real measurements with the lab's declared estimates; update capacity gates.

## 5. 100-point assessment
Workload taxonomy (20), resource sizing (20), scheduling/security (20), observability/cost (20), incident runbooks and evidence (20). Educational pass >=90 and no critical policy bypass.
