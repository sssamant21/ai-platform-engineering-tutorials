# Chapter 069 — Executable Batching, KV Cache and Memory Tuning Lab

**Status:** Published; live execution pending. **Safety:** offline memory arithmetic and bounded admission only; no GPU, model or network calls.

## 1. Setup
Python 3.11+ and pytest:
```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install pytest
```
PowerShell: `.venv\Scripts\Activate.ps1`.

## 2. KV cache and admission simulator
Save as `kv_lab.py`:

```python
from dataclasses import dataclass
import json
import math

@dataclass(frozen=True)
class ModelKV:
    layers: int
    kv_heads: int
    head_dim: int
    bytes_per_element: int = 2
    block_size: int = 16

def validate(model):
    for name in ("layers", "kv_heads", "head_dim", "bytes_per_element", "block_size"):
        value = getattr(model, name)
        if type(value) is not int or value <= 0:
            raise ValueError(f"invalid {name}")

def bytes_per_token(model):
    validate(model)
    return 2 * model.layers * model.kv_heads * model.head_dim * model.bytes_per_element

def allocated_tokens(sequence_tokens, block_size):
    if type(sequence_tokens) is not int or sequence_tokens < 0:
        raise ValueError("invalid sequence length")
    if type(block_size) is not int or block_size <= 0:
        raise ValueError("invalid block size")
    return ((sequence_tokens + block_size - 1) // block_size) * block_size

def cache_estimate(model, sequences):
    validate(model)
    if any(type(n) is not int or n < 0 for n in sequences):
        raise ValueError("invalid sequence tokens")
    useful = sum(sequences)
    allocated = sum(allocated_tokens(n, model.block_size) for n in sequences)
    return {
        "sequences": len(sequences),
        "useful_tokens": useful,
        "allocated_tokens": allocated,
        "fragmentation_tokens": allocated - useful,
        "kv_bytes": allocated * bytes_per_token(model),
    }

def admit(model, current_tokens, requested_tokens, kv_budget_bytes):
    validate(model)
    if type(kv_budget_bytes) is not int or kv_budget_bytes < 0:
        raise ValueError("invalid budget")
    if any(type(n) is not int or n < 0 for n in current_tokens):
        raise ValueError("invalid resident tokens")
    if type(requested_tokens) is not int or requested_tokens <= 0:
        return {"admitted": False, "reason": "invalid_request"}
    before = cache_estimate(model, current_tokens)["kv_bytes"]
    after = cache_estimate(model, [*current_tokens, requested_tokens])["kv_bytes"]
    if after > kv_budget_bytes:
        return {"admitted": False, "reason": "kv_budget_exceeded", "required_bytes": after, "available_bytes": kv_budget_bytes}
    return {"admitted": True, "reason": "admitted", "required_bytes": after, "available_bytes": kv_budget_bytes}

if __name__ == "__main__":
    model = ModelKV(32, 8, 128, 2, 16)
    workload = [128, 1024, 8192, 33]
    print("bytes_per_token", bytes_per_token(model))
    print(json.dumps(cache_estimate(model, workload), indent=2))
    budget = 2 * 1024**3
    print(json.dumps(admit(model, workload, 4096, budget), indent=2))
```

The estimator ignores model weights, CUDA workspace, allocator overhead, prefix sharing, cache eviction, KV quantization beyond the specified element size, offload and distributed partitioning. Set a **separate** validated KV budget after reserving all other GPU memory.

## 3. Automated tests
Save as `test_kv_lab.py`:

```python
import pytest
from kv_lab import ModelKV, bytes_per_token, allocated_tokens, cache_estimate, admit

M = ModelKV(32, 8, 128, 2, 16)

def test_bytes_per_token():
    assert bytes_per_token(M) == 131072

def test_block_rounding():
    assert allocated_tokens(17, 16) == 32
    assert allocated_tokens(16, 16) == 16

def test_fragmentation():
    result = cache_estimate(M, [1, 17, 32])
    assert result["useful_tokens"] == 50
    assert result["allocated_tokens"] == 80
    assert result["fragmentation_tokens"] == 30

def test_empty_cache():
    assert cache_estimate(M, [])["kv_bytes"] == 0

def test_admission_success():
    assert admit(M, [128], 256, 1024**3)["admitted"]

def test_budget_rejection():
    result = admit(M, [8192], 8192, 1024**3)
    assert not result["admitted"]
    assert result["reason"] == "kv_budget_exceeded"

def test_invalid_request_rejected():
    assert admit(M, [], 0, 1024**3)["reason"] == "invalid_request"

def test_gqa_saves_memory():
    full = ModelKV(32, 32, 128)
    assert bytes_per_token(M) < bytes_per_token(full)

@pytest.mark.parametrize("bad", [
    ModelKV(0, 8, 128), ModelKV(32, 0, 128),
    ModelKV(32, 8, 0), ModelKV(32, 8, 128, 0),
    ModelKV(32, 8, 128, 2, 0),
])
def test_invalid_model(bad):
    with pytest.raises(ValueError):
        bytes_per_token(bad)

def test_invalid_sequence():
    with pytest.raises(ValueError):
        cache_estimate(M, [-1])

def test_invalid_budget():
    with pytest.raises(ValueError):
        admit(M, [], 128, -1)
```

Run:
```bash
python kv_lab.py
python -m pytest -q
```
Expected: **15 passing pytest cases** (10 individual tests + 5 parameterized cases).

## 4. Experiments and failure drills
1. **Long context:** compare KV memory for 128 vs 8,192 resident tokens.
2. **Decode growth:** add 1,024 generated tokens to each sequence; recompute occupancy.
3. **Block fragmentation:** compare block sizes 8, 16, 32 and 64 for short sequences.
4. **Concurrency:** increase resident sequence count while holding total token budget fixed.
5. **Grouped-query attention:** compare 8 vs 32 KV heads.
6. **KV precision:** compare 1, 2 and 4 bytes per element; note real quantization scale metadata and kernel constraints.
7. **Overload:** set KV budget below required allocation; verify early rejection.
8. **Prefix sharing:** discuss why naive per-sequence summation overestimates memory if shared prefix blocks are truly deduplicated.

## 5. Acceptance evidence
Record Python/pytest versions, model architecture assumptions, block size, useful vs allocated tokens, fragmentation, budget, admitted/rejected requests and limitations.

| Dimension | Points |
|---|---:|
| KV cache formula and model assumptions | 20 |
| Block allocation and fragmentation | 20 |
| Admission and memory headroom | 20 |
| Reproducible tests and failure drills | 20 |
| Privacy, safety and production applicability | 20 |
| **Total** | **100** |

Recommended educational acceptance >=90/100; do not infer real GPU capacity without profiling and staging benchmarks. Preserve sanitized evidence before cleaning up disposable lab files.
