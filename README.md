# LLM-D Deployment Playbook for OpenShift

This playbook provides comprehensive guidance for deploying, operating, and troubleshooting LLM-D (Distributed LLM Inference) on Red Hat OpenShift AI.

## What is LLM-D?

LLM-D enables intelligent routing and distributed inference for Large Language Models. It provides significant performance improvements over naive load balancing through:

- **Prefix-aware routing**: Routes requests to replicas with cached prefixes, improving KV cache hit rates from ~25% to 90%+
- **Prefill/Decode disaggregation**: Separates compute-intensive prefill from memory-bandwidth-bound decode phases
- **Load-aware scheduling**: Balances traffic based on real-time metrics from vLLM instances

## Playbook Contents

This playbook is self-contained with all deployment artifacts included.

### Guides

| Guide | Description |
|-------|-------------|
| [Pre-flight Validation](01-preflight-validation.md) | Verify cluster readiness and prerequisites |
| [Quick Start](02-quick-start.md) | Connected environment deployment in minutes |
| [Advanced Deployment](03-advanced-deployment.md) | Bare metal, MetalLB, custom configurations |
| [Automated Deployment](04-automated-deployment.md) | GitOps and automation patterns |
| [Disconnected Installs](05-disconnected-installs.md) | Air-gapped and restricted network deployments |
| [Running Benchmarks](06-running-benchmarks.md) | Performance testing with GuideLLM |
| [Performance Debugging](07-performance-debugging.md) | Diagnosing and resolving performance issues |

### Included Artifacts

| Directory | Contents |
|-----------|----------|
| `gitops/operators/` | Operator installation manifests (MetalLB, Service Mesh, RHOAI, etc.) |
| `gitops/instance/` | Instance configurations (LLM-D, Gateway, monitoring, GuideLLM) |
| `gitops/ocp-4.19/` | OCP 4.19 prerequisites and configs |
| `gitops/ocp-4.18/` | OCP 4.18 prerequisites (experimental) |
| `gitops/disconnected/` | ImageSetConfigurations for air-gapped installs |
| `monitoring/` | Prometheus and Grafana stack for metrics |
| `vllm/` | Vanilla vLLM deployment for baseline comparison |
| `llm-d/` | LLM-D deployment configurations |
| `guidellm/` | GuideLLM benchmark configurations and overlays |
| `benchmark-job/` | Kubernetes job templates for benchmarking |
| `assets/` | Screenshots and images for documentation |

## Prerequisites Overview

### Minimum Requirements

- **OpenShift**: 4.19+
- **OpenShift AI**: 2.25+ (3.0+ recommended)
- **GPU**: NVIDIA GPU with appropriate drivers
- **Role**: `cluster-admin`

### Required Operators

Install in this order:

1. Cert Manager
2. MetalLB (bare metal only)
3. Service Mesh 3
4. Connectivity Link (RHOAI 3.0+)
5. Red Hat OpenShift AI
6. Node Feature Discovery
7. NVIDIA GPU Operator

### Optional Operators

- **LeaderWorkerSet**: Required only for large MoE models with expert parallelism


## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                         Gateway API                              │
│                    (openshift-ai-inference)                      │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    EPP (Scheduler)                               │
│              - Prefix-aware scoring                              │
│              - Load-aware routing                                │
│              - KV cache utilization                              │
└─────────────────────────────┬───────────────────────────────────┘
                              │
            ┌─────────────────┼─────────────────┐
            ▼                 ▼                 ▼
┌───────────────────┐ ┌───────────────────┐ ┌───────────────────┐
│   vLLM Replica 1  │ │   vLLM Replica 2  │ │   vLLM Replica N  │
│   (KV Cache)      │ │   (KV Cache)      │ │   (KV Cache)      │
└───────────────────┘ └───────────────────┘ └───────────────────┘
```

## Support Resources

- [Red Hat OpenShift AI Documentation](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/)
- [LLM-D GitHub Repository](https://github.com/llm-d/llm-d)
- [Gateway API Inference Extension](https://gateway-api-inference-extension.sigs.k8s.io/)
- [vLLM Documentation](https://docs.vllm.ai/)

## Contributing

This playbook consolidates learnings from real-world LLM-D implementations. Please contribute updates as the tooling evolves or new lessons are learned.
