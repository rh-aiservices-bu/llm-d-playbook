# Performance Debugging Guide

This guide helps diagnose and resolve performance bottlenecks in LLM-D deployments. It is based on the PSAP Guide to LLM Inference Performance Diagnosis.

## Scope

This document focuses on common performance issues for:
- Single-node deployments
- Single and multi-replica configurations
- Standard inference workloads

**Out of scope:** Multi-node deployments, tool-calling/structured-output, and reasoning models are additional topics with specific considerations.

## A Note on Tuning

vLLM comes with **excellent defaults out-of-the-box**. It is designed to maximize hardware utilization by dynamically managing batch sizes.

> **Warning:** Before adjusting advanced tuning knobs, use this guide to understand what might be limiting performance. Usually, latency/throughput cannot be significantly improved by changing CLI args alone. Significant improvements might require:
> - Scaling horizontally (more replicas)
> - Scaling vertically (more/larger GPUs)
> - Using a smaller or quantized model

## Key Performance Metrics

### Breaking Down Latency

Total request latency (E2E latency) must be broken into components to diagnose root causes:

| Metric | Description | What It Tells You |
|--------|-------------|-------------------|
| **E2E Latency** | Total time from request to final response | Overall user experience |
| **Time to First Token (TTFT)** | Time until first token appears | Includes queuing + prefill phase |
| **Inter-Token Latency (ITL)** | Time between subsequent tokens | Decode phase performance |
| **Generation Throughput** | Output tokens per second | Server capacity |

### Prometheus Queries

```promql
# E2E Latency (Median)
histogram_quantile(0.5, sum by(model_name, pod, le)
  (rate(vllm:e2e_request_latency_seconds_bucket{}[1m])))

# TTFT (Median)
histogram_quantile(0.5, sum by(model_name, pod, le)
  (rate(vllm:time_to_first_token_seconds_bucket{}[1m])))

# ITL (Median)
histogram_quantile(0.5, sum by(model_name, pod, le)
  (rate(vllm:inter_token_latency_seconds_bucket{}[1m])))

# Throughput (Mean)
rate(vllm:generation_tokens_total[1m])
```

## System Health Metrics

These metrics explain the server state during inference:

| Metric | Description | Prometheus Query |
|--------|-------------|-----------------|
| **Running requests** | Currently being processed | `vllm:num_requests_running` |
| **Waiting requests** | Queued due to capacity | `vllm:num_requests_waiting` |
| **Queue Time** | Time spent waiting | `vllm:request_queue_time_seconds_sum` |
| **KV Cache usage** | Cache memory utilization | `vllm:gpu_cache_usage_perc` |
| **Preemptions** | Requests stopped mid-generation | `vllm:num_preemptions_total` |

### Reading vLLM Logs

Snapshots of system health appear in vLLM logs:

```
(APIServer pid=1) INFO: Engine 000: Avg prompt throughput: 4003.6 tokens/s,
Avg generation throughput: 1124.3 tokens/s, Running: 48 reqs, Waiting: 0 reqs,
GPU KV cache usage: 28.9%, Prefix cache hit rate: 0.0%
```

### Startup Capacity Checks

At startup, vLLM calculates KV cache capacity:

```
(Worker_TP0_EP0 pid=415) INFO: Available KV cache memory: 66.15 GiB
(EngineCore_DP0 pid=279) INFO: GPU KV cache size: 1,475,824 tokens
(EngineCore_DP0 pid=279) INFO: Maximum concurrency for 10,000 tokens per request: 147.58x
```

### Sanity Check

`num_requests_running + num_requests_waiting` should roughly equal total in-flight requests (e.g., load generator concurrency).

If the sum is **lower** than expected, the bottleneck is **upstream from vLLM**:
- Load balancer
- Ingress/Gateway
- Vector DB (for RAG)
- Network

## Hardware Constraints

### VRAM Allocation

GPU memory is split between:

| Component | Type | Scaling |
|-----------|------|---------|
| **Model Weights** | Static | Fixed by model size and precision |
| **KV Cache** | Dynamic | Scales with batch size and sequence length |

### Capacity Bottleneck

If model weights consume >75% of VRAM:
- Limited KV Cache space
- Limited maximum concurrency
- Requests queue under load

### Multi-GPU Considerations

| Technique | Use Case | Communication Needs |
|-----------|----------|---------------------|
| **Tensor Parallelism (TP)** | Split layers across GPUs | High bandwidth (NVLink) |
| **Expert Parallelism (EP)** | MoE model distribution | High bandwidth (NVLink) |
| **Pipeline Parallelism (PP)** | Sequential GPU stages | Moderate bandwidth |

> **Important:** For TP and EP, high-bandwidth interconnects (NVLink) are essential. PCIe-only communication introduces significant ITL latency.

Check GPU topology:
```bash
nvidia-smi topo -m
```

## Diagnosis Playbook

### 1. High TTFT (Slow Start)

#### Check: Queueing Issue

```promql
# Are requests waiting AND cache full?
vllm:num_requests_waiting > 0
vllm:gpu_cache_usage_perc > 90
```

**Diagnosis:** System is full. New requests wait for a slot.

**Remediation:**
- Add more GPUs or larger GPUs
- Add more replicas (scale out)
- Use smaller/quantized model (AWQ/GPTQ/FP8)

#### Check: Compute-Bound Prefill

```promql
# Requests waiting is low but TTFT is high
vllm:num_requests_waiting ~ 0
```

**Diagnosis:** Long input prompts (ISL) taking time to process.

**Remediation:**
- Add more GPUs/replicas
- Use Prefill/Decode disaggregation (LLM-D)

### 2. High ITL (Slow Generation)

#### Check: Single User Slow

**Is it slow even with one user (no load)?**

**If using Tensor Parallelism:**
- Are GPUs connected via NVLink?
- Check with `nvidia-smi topo -m`
- PCIe-only TP will have poor ITL

**If single GPU:**
- May be limited by GPU memory bandwidth
- Consider larger GPU or TP across NVLink-connected GPUs

#### Check: Slow Under Load

**Only slow with many concurrent users?**

**Diagnosis:** Batch size impact. vLLM batches requests to maximize throughput, adding latency per token.

**Remediation:**
- Run more replicas to spread load
- Set `--max-num-seqs` to limit batch size (will cause queueing)

### 3. Poor Throughput Scaling

#### Check: Memory Bound

```promql
# KV Cache full AND requests waiting
vllm:gpu_cache_usage_perc ~ 100
vllm:num_requests_waiting > 0
```

**Diagnosis:** Cannot fit more concurrent requests. Additional users just queue (increasing TTFT).

**Remediation:**
- Add more VRAM (larger GPUs)
- Add more replicas
- Reduce sequence lengths if possible

## Sequence Length Considerations

### Impact of ISL/OSL

| Parameter | Impact |
|-----------|--------|
| **Input Sequence Length (ISL)** | High ISL increases prefill compute, directly increasing TTFT |
| **Output Sequence Length (OSL)** | High OSL linearly increases E2E latency, occupies GPU slots longer |
| **Total Sequence Length** | KV Cache consumption = sum of ISL+OSL across all active requests |

### Example

A deployment might handle:
- 400 requests at 1,000 token ISL ✓
- But NOT 400 requests at 10,000 token ISL (insufficient KV Cache)

### RAG Pipeline Considerations

RAG dramatically expands ISL:
- User query: 50 tokens
- After retrieval: 4,000+ tokens

**Diagnosis steps:**
1. Validate the **actual** token count reaching vLLM
2. Check if latency occurs before vLLM (Vector DB, reranking)

### Multimodal Inputs

Images and audio require encoding before prefill:
- Adds computational overhead to prefill phase
- Increases TTFT independent of text length

## LLM-D Specific Debugging

### Scheduler Not Routing Correctly

**Symptoms:**
- Low cache hit rate despite LLM-D deployment
- Similar performance to vanilla vLLM

**Check scheduler logs:**
```bash
oc logs -l app.kubernetes.io/component=router-scheduler -n <namespace>
```

**Verify EndpointPicker configuration:**
```bash
oc get llminferenceservice <name> -n <namespace> -o yaml | grep -A50 scheduler
```

### HTTPRoute Hijacking

**Symptoms:**
- Requests not reaching expected LLMInferenceService
- 404 errors or unexpected responses

**Check HTTPRoutes:**
```bash
oc get httproute -A
```

**Resolution:** See [Advanced Deployment](03-advanced-deployment.md) for Gateway namespace restrictions.

### P/D Disaggregation Issues

**Symptoms:**
- KV transfer extremely slow
- No improvement from P/D pattern

**Check network:**
```bash
# Verify InfiniBand/RoCE
oc exec <pod> -- env | grep -E "KSERVE_INFER|UCX"

# Check for TCP fallback warnings in logs
oc logs <pod> | grep -i "tcp\|nixl"
```

**Resolution:** Ensure InfiniBand/RoCE is configured. TCP fallback results in severe performance degradation.

## Prometheus Dashboard Queries

### Performance Overview

```promql
# TTFT P95
histogram_quantile(0.95, sum by(model_name, pod, le)
  (rate(vllm:time_to_first_token_seconds_bucket{}[5m])))

# ITL P95
histogram_quantile(0.95, sum by(model_name, pod, le)
  (rate(vllm:inter_token_latency_seconds_bucket{}[5m])))

# Throughput
sum(rate(vllm:generation_tokens_total[5m]))
```

### Saturation Indicators

```promql
# KV Cache utilization
vllm:gpu_cache_usage_perc

# Waiting queue depth
vllm:num_requests_waiting

# Total preemptions (should be 0)
increase(vllm:num_preemptions_total[1h])
```

### Cache Efficiency (LLM-D)

```promql
# Prefix cache hit rate
vllm:prefix_cache_hit_rate

# Cache-aware routing effectiveness
# Compare TTFT between first turn and subsequent turns
```

## Quick Diagnosis Checklist

```
□ Check vLLM logs for startup capacity
□ Verify GPU memory allocation (model weights vs KV cache)
□ Monitor num_requests_waiting (should be 0 under normal load)
□ Check KV cache usage (>90% indicates saturation)
□ Verify no preemptions occurring
□ For TP: confirm NVLink connectivity
□ For P/D: confirm InfiniBand/RoCE working
□ For LLM-D: verify scheduler routing decisions
```

## Escalation Information

When escalating to engineering, gather:

1. **Model and vLLM deployment args**
2. **Hardware resources:**
   - GPU type (e.g., A100-80GB)
   - Count and interconnects (NVLink, PCIe)
3. **Metrics:** TTFT, ITL, throughput numbers
4. **Workload:** ISL and OSL of requests
5. **Deployment mode:**
   - RHOAI version
   - RawDeployment vs LLMInferenceService
6. **Versions:**
   - vLLM version
   - OpenShift version
   - RHOAI version
7. **Goal:**
   - Specific SLA requirements
   - Performance regressions observed
   - Expected vs actual performance

## Common Remediation Summary

| Problem | Quick Fix | Long-term Solution |
|---------|-----------|-------------------|
| High TTFT (queueing) | Reduce load | Add replicas/GPUs |
| High TTFT (prefill) | Reduce ISL | P/D disaggregation |
| High ITL (single user) | - | Larger GPU / NVLink TP |
| High ITL (under load) | Limit batch size | Add replicas |
| Low cache hits | Check routing | Verify LLM-D scheduler |
| Memory exhaustion | Reduce concurrency | Larger GPUs / more replicas |

## Next Steps

- Review [Running Benchmarks](06-running-benchmarks.md) for baseline establishment
- See [Advanced Deployment](03-advanced-deployment.md) for P/D disaggregation setup
- Check [Pre-flight Validation](01-preflight-validation.md) for infrastructure issues
