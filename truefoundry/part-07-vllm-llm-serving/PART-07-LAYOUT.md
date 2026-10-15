# Part 07 — vLLM & LLM Serving: Master Layout

**Scope:** vLLM architecture, TrueFoundry deployments, GPU/CUDA compatibility, model loading, distributed inference, KV-cache scheduling, performance tuning and incident recovery.

| Chapter | Topic | Canonical | Hands-on Lab |
|---|---|---|---|
| 7.1 | vLLM Fundamentals and Serving Architecture | [Tutorial](01-vllm-fundamentals-serving-architecture/01-vllm-fundamentals-serving-architecture.md) | [Lab](labs/01-vllm-fundamentals-serving-architecture-check.md) |
| 7.2 | vLLM Deployment with TrueFoundry and Kubernetes | [Tutorial](02-vllm-deployment-truefoundry-kubernetes/02-vllm-deployment-truefoundry-kubernetes.md) | [Lab](labs/02-vllm-deployment-truefoundry-kubernetes-check.md) |
| 7.3 | GPU CUDA PyTorch and Device Detection | [Tutorial](03-gpu-cuda-pytorch-device-detection/03-gpu-cuda-pytorch-device-detection.md) | [Lab](labs/03-gpu-cuda-pytorch-device-detection-check.md) |
| 7.4 | Model Loading Tokenizers and Tensor Parallelism | Planned | Planned |
| 7.5 | PagedAttention KV Cache and Continuous Batching | Planned | Planned |
| 7.6 | vLLM Performance Tuning Observability and Troubleshooting | Planned | Planned |
| 7.7 | Production vLLM Deployment and Recovery Capstone | Planned | Planned |

**Publication:** 3/7 chapters and 3/7 labs published. Formal technical/vendor validation, production/SRE review, revised-final approval and live lab execution remain separate pending milestones.

**Operating principles:** pin immutable model/image/runtime; verify actual GPU device chain; measure TTFT/ITL and KV cache; use lowest proven healthy layer and first failed transition; recover via authoritative TrueFoundry/GitOps release.
