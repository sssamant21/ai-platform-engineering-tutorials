\# AI Platform Engineering Tutorials



Production-focused tutorials, hands-on labs, troubleshooting scenarios, and operational runbooks for AI platform engineering.



\## Purpose



This repository is designed for engineers who build, operate, support, and troubleshoot production AI infrastructure.



The focus is not only on deploying AI workloads, but also on understanding the infrastructure underneath them and developing practical SRE and platform-engineering skills.



\## Audience



This repository is intended for:



\- Site Reliability Engineers (SRE)

\- Platform Engineers

\- DevOps Engineers

\- Cloud Engineers

\- ML Infrastructure Engineers

\- AI Platform Engineers

\- Engineers supporting production AI/ML workloads



\## Learning Approach



Each technology track follows a production-focused progression:



```text

Fundamentals

&#x20;   ↓

Architecture

&#x20;   ↓

Hands-On Labs

&#x20;   ↓

Deployment

&#x20;   ↓

Operations

&#x20;   ↓

Observability

&#x20;   ↓

Troubleshooting

&#x20;   ↓

Production Runbooks

&#x20;   ↓

End-to-End Projects

```



Tutorial development follows a review workflow:



```text

Draft

&#x20; ↓

Technical / Vendor Validation

&#x20; ↓

Production / SRE Review

&#x20; ↓

Revised Final

&#x20; ↓

Canonical

&#x20; ↓

Hands-On Lab

&#x20; ↓

Repository Validation

```



\## Technology Tracks



\### TrueFoundry



The TrueFoundry track covers AI platform architecture and production operations, including:



\- TrueFoundry fundamentals

\- Control Plane and Compute Plane architecture

\- `tfy-agent`

\- Kubernetes integration

\- CPU and GPU workloads

\- NVIDIA and CUDA infrastructure

\- Application deployment

\- Model serving

\- vLLM

\- Scaling and performance

\- Networking

\- Security

\- Observability

\- Production operations

\- Troubleshooting

\- SRE runbooks

\- End-to-end production projects



See:



```text

truefoundry/

```



\## Future Tracks



The repository is structured to support additional AI platform engineering topics such as:



\- vLLM

\- Kubernetes AI infrastructure

\- NVIDIA GPU infrastructure

\- CUDA

\- Model serving

\- AI gateways

\- RAG infrastructure

\- LLM observability

\- AI workload performance engineering



New tracks should be added only when sufficient hands-on and production-focused material is available.



\## Repository Principles



Content in this repository should be:



\- Production focused

\- Hands on

\- Infrastructure aware

\- Vendor validated where applicable

\- SRE oriented

\- Reproducible

\- Troubleshooting driven

\- Safe by default



Potentially destructive operations must be clearly identified.



Read-only production discovery labs should use the designation:



```text

\[SAFE-READ]

```



\## Repository Structure



```text

ai-platform-engineering-tutorials/

│

├── README.md

│

└── truefoundry/

&#x20;   ├── README.md

&#x20;   ├── MASTER-TOC.md

&#x20;   ├── STATUS-TRACKER.md

&#x20;   │

&#x20;   ├── part-01-foundations-architecture/

&#x20;   ├── part-02-kubernetes-integration/

&#x20;   ├── part-03-compute-scheduling/

&#x20;   ├── ...

&#x20;   └── part-15-end-to-end-production-project/

```



Individual directories are created as the corresponding material reaches development.



\## Current Track



Development currently begins with:



```text

TrueFoundry

└── Part 1 — Foundations \& Architecture

&#x20;   └── 1.1 — What Is TrueFoundry?

```

