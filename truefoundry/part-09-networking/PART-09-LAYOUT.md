# Part 09 — Networking: Master Layout

**Scope:** TrueFoundry networking architecture, Kubernetes Services, ingress/gateways, DNS, TLS, network security, streaming request paths and operational recovery.

| Chapter | Topic | Canonical | Hands-on Lab |
|---|---|---|---|
| 9.1 | TrueFoundry Networking Architecture and End-to-End Request Path | [Tutorial](01-truefoundry-networking-architecture-request-path/01-truefoundry-networking-architecture-request-path.md) | [Lab](labs/01-truefoundry-networking-architecture-request-path-check.md) |
| 9.2 | Kubernetes Services, EndpointSlices and Service Discovery | Planned | Planned |
| 9.3 | Ingress, Gateway API and Cloud Load Balancers | Planned | Planned |
| 9.4 | DNS, TLS, Certificates and Secure Endpoint Exposure | Planned | Planned |
| 9.5 | NetworkPolicy, Private Networking and Egress Security | Planned | Planned |
| 9.6 | Inference Streaming, Timeouts and Networking Troubleshooting | Planned | Planned |
| 9.7 | Production Networking and Recovery Capstone | Planned | Planned |

**Publication:** 1/7 chapters and 1/7 labs published. Formal vendor/SRE reviews and live lab execution remain pending.

**Operating principles:** separate management and runtime paths; validate the actual deployed gateway and cloud architecture; trace client → DNS → LB → gateway → Service → EndpointSlice → Pod → model; preserve TLS and identity boundaries; document LPHL/FFT, rollback and SLO recovery.
