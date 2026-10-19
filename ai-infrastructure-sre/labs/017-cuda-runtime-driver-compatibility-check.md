# Chapter 017 — Executable CUDA Runtime and Driver Compatibility Lab

**Status:** Published. Offline tests and live CUDA validation pending.

## 1. Setup
Requires Python 3.11+ and pytest; no GPU or cloud credentials needed for the synthetic evaluator.

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install pytest
```

PowerShell: `.venv\Scripts\Activate.ps1`.

## 2. Explicit vendor-matrix policy gate
Save as `cuda_compatibility.py`:

```python
from dataclasses import dataclass
import json
import re

DIGEST = re.compile(r"^sha256:[0-9a-f]{64}$")

@dataclass(frozen=True)
class Stack:
    node: str
    owner: str
    environment: str
    gpu_sku: str
    driver_version: str
    cuda_runtime: str
    framework_build: str
    node_image: str
    workload_image: str
    gpu_count_expected: int
    gpu_count_detected: int
    driver_owner_unique: bool
    kernel_module_healthy: bool
    toolkit_runtime_healthy: bool
    device_plugin_healthy: bool
    cuda_kernel_smoke_passed: bool
    framework_smoke_passed: bool
    workload_slo_healthy: bool
    checkpoint_verified: bool
    n_minus_one_verified: bool
    telemetry_healthy: bool
    security_approved: bool
    change_approved: bool
    rollback_tested: bool

def evaluate(stack, approved_pairs):
    if not isinstance(stack, Stack):
        raise TypeError("Stack required")
    if any(not isinstance(s, str) or not s.strip() for s in
           (stack.node, stack.owner, stack.gpu_sku, stack.driver_version,
            stack.cuda_runtime, stack.framework_build)):
        return {"accepted": False, "reason": "missing_metadata"}
    if stack.environment not in {"dev", "staging", "prod"}:
        return {"accepted": False, "reason": "invalid_environment"}
    if any(not isinstance(d, str) or not DIGEST.fullmatch(d)
           for d in (stack.node_image, stack.workload_image)):
        return {"accepted": False, "reason": "invalid_image_digest"}
    if (type(stack.gpu_count_expected) is not int or type(stack.gpu_count_detected) is not int
            or stack.gpu_count_expected <= 0 or stack.gpu_count_detected < 0):
        return {"accepted": False, "reason": "invalid_gpu_count"}
    if not isinstance(approved_pairs, (set, frozenset)):
        return {"accepted": False, "reason": "invalid_policy"}
    key = (stack.gpu_sku, stack.driver_version, stack.cuda_runtime, stack.framework_build)
    gates = {
        "vendor_matrix": key in approved_pairs,
        "gpu_count": stack.gpu_count_detected == stack.gpu_count_expected,
        "driver_owner": stack.driver_owner_unique is True,
        "kernel_module": stack.kernel_module_healthy is True,
        "container_runtime": stack.toolkit_runtime_healthy is True,
        "device_plugin": stack.device_plugin_healthy is True,
        "cuda_kernel_smoke": stack.cuda_kernel_smoke_passed is True,
        "framework_smoke": stack.framework_smoke_passed is True,
        "workload_slo": stack.workload_slo_healthy is True,
        "checkpoint": stack.checkpoint_verified is True,
        "n_minus_one": stack.n_minus_one_verified is True,
        "telemetry": stack.telemetry_healthy is True,
        "security": stack.security_approved is True,
        "change_approval": stack.change_approved is True,
        "rollback": stack.rollback_tested is True,
    }
    failed = sorted(name for name, ok in gates.items() if not ok)
    return {"accepted": not failed, "failed_gates": failed,
            "node": stack.node, "environment": stack.environment}

def example():
    stack = Stack(
        "gpu-staging-01", "gpu-platform-sre", "staging",
        "synthetic-gpu", "synthetic-driver", "synthetic-cuda",
        "synthetic-framework",
        "sha256:" + "a" * 64, "sha256:" + "b" * 64,
        2, 2, *([True] * 13)
    )
    approved = {(stack.gpu_sku, stack.driver_version,
                 stack.cuda_runtime, stack.framework_build)}
    return stack, approved

if __name__ == "__main__":
    s, policy = example()
    print(json.dumps(evaluate(s, policy), indent=2))
```

**Important:** The allowlist in `example()` is intentionally **synthetic** and proves no real NVIDIA compatibility. Production policies must reference independently verified official compatibility matrices, GPU architecture support and workload test evidence. Boolean inputs are external attestations, not independently measured facts.

## 3. Automated tests
Save as `test_cuda_compatibility.py`:

```python
from dataclasses import replace
import pytest
from cuda_compatibility import evaluate, example

def test_baseline():
    s, p = example()
    assert evaluate(s, p)["accepted"]

def test_wrong_type():
    _, p = example()
    with pytest.raises(TypeError):
        evaluate({}, p)

def test_missing_owner():
    s, p = example()
    assert evaluate(replace(s, owner=" "), p)["reason"] == "missing_metadata"

def test_environment():
    s, p = example()
    assert evaluate(replace(s, environment="other"), p)["reason"] == "invalid_environment"

def test_bad_digest():
    s, p = example()
    assert evaluate(replace(s, node_image="latest"), p)["reason"] == "invalid_image_digest"

def test_bool_gpu():
    s, p = example()
    assert evaluate(replace(s, gpu_count_expected=True), p)["reason"] == "invalid_gpu_count"

def test_negative_gpu():
    s, p = example()
    assert evaluate(replace(s, gpu_count_detected=-1), p)["reason"] == "invalid_gpu_count"

def test_bad_policy():
    s, _ = example()
    assert evaluate(s, [])["reason"] == "invalid_policy"

def test_unknown_pair_fails_closed():
    s, p = example()
    assert "vendor_matrix" in evaluate(replace(s, cuda_runtime="unknown"), p)["failed_gates"]

def test_missing_gpu():
    s, p = example()
    assert "gpu_count" in evaluate(replace(s, gpu_count_detected=1), p)["failed_gates"]

@pytest.mark.parametrize("field,gate", [
    ("driver_owner_unique", "driver_owner"),
    ("kernel_module_healthy", "kernel_module"),
    ("toolkit_runtime_healthy", "container_runtime"),
    ("device_plugin_healthy", "device_plugin"),
    ("cuda_kernel_smoke_passed", "cuda_kernel_smoke"),
    ("framework_smoke_passed", "framework_smoke"),
    ("workload_slo_healthy", "workload_slo"),
    ("checkpoint_verified", "checkpoint"),
    ("n_minus_one_verified", "n_minus_one"),
    ("telemetry_healthy", "telemetry"),
    ("security_approved", "security"),
    ("change_approved", "change_approval"),
    ("rollback_tested", "rollback"),
])
def test_failed_gate(field, gate):
    s, p = example()
    assert gate in evaluate(replace(s, **{field: False}), p)["failed_gates"]

def test_nonboolean_approval():
    s, p = example()
    assert "change_approval" in evaluate(
        replace(s, change_approved="yes"), p)["failed_gates"]

def test_nonboolean_security():
    s, p = example()
    assert "security" in evaluate(
        replace(s, security_approved=1), p)["failed_gates"]
```

Run:
```bash
python cuda_compatibility.py
python -m pytest -q
```

Expected **25 pytest cases** (12 standalone + 13 parameterized); execution not yet performed.

## 4. Optional approved live GPU qualification
1. Capture `nvidia-smi`, `nvcc --version` (if present), `uname -r` and installed runtime/framework versions.
2. Confirm driver/runtime/GPU support using current NVIDIA and framework documentation.
3. Compile and execute the chapter's minimal CUDA kernel on a staging GPU.
4. Run pinned, digest-verified application and Kubernetes GPU smoke jobs.
5. Test CUDA error, wrong GPU architecture, missing device plugin and driver mismatch using **synthetic fixtures**, not production faults.
6. Validate workload SLO, NCCL where applicable, checkpoint restore, N-1 capacity, telemetry and rollback.
7. Record UTC evidence, signed image digests, independent reviewer and change ticket.

## 5. 100-point rubric
Architecture (20), compatibility matrix (20), diagnostics (20), incidents/runbooks (20), executable evidence (20). Pass >=90 with no critical gate failure.
