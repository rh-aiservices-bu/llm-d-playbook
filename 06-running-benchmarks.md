# Running Benchmarks

This guide covers performance benchmarking for LLM-D deployments using GuideLLM and synthetic test data generators designed to demonstrate LLM-D's intelligent routing benefits.

## Overview

Benchmarking helps you:
- Validate deployment performance
- Compare LLM-D intelligent routing vs. vanilla vLLM (round-robin)
- Demonstrate prefix caching effectiveness
- Test heterogeneous workload handling
- Establish performance baselines for SLAs

## GuideLLM Overview

GuideLLM is the recommended benchmarking tool for LLM inference. It supports:
- Concurrent request testing
- Request-per-second (RPS) rate limiting
- Custom data files for realistic workloads
- JSON output for analysis

## Step 0: Deploy Monitoring Stack

Deploy Prometheus and Grafana for real-time metrics visualization.

All artifacts are included in this playbook. Run commands from the playbook directory.

```bash
# From the playbook directory
cd "llm-d playbook"

# Deploy monitoring (using the included monitoring stack)
oc apply -k monitoring

# Wait for Grafana
oc wait --for=condition=ready pod -l app=grafana -n llm-d-monitoring --timeout=300s

# Get Grafana URL
export GRAFANA_URL=$(oc get route grafana-secure -n llm-d-monitoring -o jsonpath='{.spec.host}')
echo "Grafana: https://$GRAFANA_URL"
```

Access Grafana with default credentials: `admin` / `admin`

### Key Metrics to Watch

| Metric | What to Look For |
|--------|------------------|
| **KV Cache Hit Rate** | Higher is better - LLM-D should show 90%+ vs ~25% for round-robin |
| **Time to First Token (TTFT)** | Lower P95/P99 indicates better tail latency |
| **Requests per Second** | Overall throughput |
| **GPU Utilization** | Balanced utilization across replicas |

## Step 1: Generate Test Data

LLM-D includes synthetic test data generators specifically designed to demonstrate the benefits of intelligent routing.

### Install Dependencies

```bash
cd gitops/instance/guidellm/llm-d-test-data-generator
pip install -r requirements.txt
```

### Prefix Cache Generator

The prefix cache generator creates prompt pairs with shared prefixes to simulate multi-turn conversations. This demonstrates how LLM-D's prefix-aware routing improves cache hit rates.

**How it works:**
- Generates pairs of prompts where the second prompt contains the first as a prefix
- Simulates multi-turn conversations with shared context
- Interleaves prefix-only and full prompts to test cache reuse

**Generate test data:**

```bash
cd prefix

# Quick test (10 concurrent users)
python prefix-cache-generator.py \
  --target-prefix-words 5000 \
  --target-continuation-words 1000 \
  --num-pairs 100 \
  --chunk-size 20 \
  --output-prefix-csv "pairs-10.csv" \
  --output-guidellm-csv "prompts-10.csv"

# Generate data sets for various concurrency levels
./generate-all.sh
```

**Output files:**
- `prefix-pairs.csv` - Side-by-side view of prefix and full prompts
- `prefix-prompts.csv` - GuideLLM-ready format with interleaved prompts

### Heterogeneous Workload Generator

The heterogeneous generator creates mixed workloads with different request sizes. This is useful for testing P/D disaggregation scenarios.

**Generate test data:**

```bash
cd heterogeneous

# Generate mixed workload (90% short, 10% long)
python heterogeneous-workload-generator.py \
  --workload-n-words 500 \
  --workload-m-words 10000 \
  --total-prompts 10000 \
  --ratio-n-to-m 9 \
  --output-tokens 250 \
  --output-csv "heterogeneous-prompts.csv"
```

**Parameters:**
- `--workload-n-words`: Word count for "small" requests (default: 500)
- `--workload-m-words`: Word count for "large" requests (default: 10000)
- `--ratio-n-to-m`: Ratio of small to large requests (e.g., 9 means 9:1)

## Step 2: Run GuideLLM Benchmarks

### Install GuideLLM

```bash
pip install guidellm[recommended]==0.3.1
```

### Get Inference Endpoint

```bash
# Get Gateway URL
export INFERENCE_URL=$(oc -n openshift-ingress get gateway openshift-ai-inference \
  -o jsonpath='{.status.addresses[0].value}')

# Set target endpoint
export TARGET="http://${INFERENCE_URL}/<namespace>/<llm-d-instance>"
export MODEL="Qwen/Qwen3-4B"  # Match your deployed model

echo "Target: $TARGET"
```

### Run Single Benchmark

```bash
guidellm benchmark run \
  --target $TARGET \
  --model $MODEL \
  --data prompts-10.csv \
  --rate-type concurrent \
  --rate 10 \
  --max-seconds 120 \
  --output-path results-10.json
```

### Run Benchmark Suite

Use the provided script to run benchmarks at multiple concurrency levels:

```bash
#!/bin/bash
# bench-all.sh

TARGET=http://<gateway-hostname>/<namespace>/<llm-d-instance>
MODEL=Qwen/Qwen3-4B
SCENARIO_NAME="llm-d-intelligent-inference-x2"
MAX_SECONDS=120

# Benchmark configurations: rate and data file
BENCHMARKS=(
  "500 prompts-500.csv"
  "250 prompts-250.csv"
  "100 prompts-100.csv"
  "50 prompts-50.csv"
  "25 prompts-25.csv"
  "10 prompts-10.csv"
)

for benchmark in "${BENCHMARKS[@]}"; do
  RATE=$(echo $benchmark | awk '{print $1}')
  DATA=$(echo $benchmark | awk '{print $2}')

  echo "Running benchmark with rate=$RATE and data=$DATA"
  guidellm benchmark run --target $TARGET \
    --model $MODEL \
    --data $DATA \
    --rate-type concurrent \
    --rate $RATE \
    --max-seconds $MAX_SECONDS \
    --output-path $SCENARIO_NAME-$RATE.json
done

# Archive results
tar -cf $SCENARIO_NAME.tar $SCENARIO_NAME-*.json
echo "Archive created: $SCENARIO_NAME.tar"
```

## Step 3: Run as Kubernetes Job

For production benchmarks, run GuideLLM as a Kubernetes Job:

```yaml
kind: Job
apiVersion: batch/v1
metadata:
  name: guidellm-benchmark-job
  namespace: demo-llm
spec:
  backoffLimit: 1
  completions: 1
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: guidellm
          image: 'ghcr.io/vllm-project/guidellm@sha256:f7123f5a4b9283e721a9b43bc99e8b2a1d9eac1c1e1ecba47b5368998c341ff3'
          command:
            - guidellm
          args:
            - benchmark
            - '--target'
            - 'http://openshift-ai-inference-openshift-default.openshift-ingress.svc.cluster.local/<namespace>/<model>'
            - '--model'
            - 'Qwen/Qwen3-4B'
            - '--processor'
            - 'Qwen/Qwen3-4B'
            - '--data'
            - '{"prompt_tokens":1000,"output_tokens":1000}'
            - '--rate-type'
            - concurrent
            - '--max-seconds'
            - '300'
            - '--rate'
            - '1,2,4,8,16'
            - '--output-path'
            - /results/output.json
          env:
            - name: HUGGING_FACE_HUB_TOKEN
              valueFrom:
                secretKeyRef:
                  name: huggingface-secret
                  key: hf_token
            - name: HF_HOME
              value: /tmp/huggingface_cache
          volumeMounts:
            - name: results-volume
              mountPath: /results
      volumes:
        - name: results-volume
          persistentVolumeClaim:
            claimName: benchmark-results-pvc
```

### Using Custom Test Data in Kubernetes

Create a ConfigMap with your generated test data:

```bash
# Create ConfigMap from test data
oc create configmap benchmark-data -n demo-llm \
  --from-file=prompts-10.csv \
  --from-file=prompts-50.csv \
  --from-file=prompts-100.csv
```

Mount in the Job:

```yaml
spec:
  template:
    spec:
      containers:
        - name: guidellm
          volumeMounts:
            - name: data-volume
              mountPath: /data
          args:
            - benchmark
            - '--data'
            - '/data/prompts-100.csv'
            # ... other args
      volumes:
        - name: data-volume
          configMap:
            name: benchmark-data
```

## Step 4: Compare LLM-D vs vLLM

The playbook includes pre-configured vLLM and LLM-D deployments for comparison benchmarks.

### Deploy vLLM Baseline

First, deploy vanilla vLLM to establish a baseline:

```bash
# Deploy vLLM (round-robin load balancing) - included in playbook
oc apply -k vllm

# Wait for pods
oc wait --for=condition=ready pod -l serving.kserve.io/inferenceservice=qwen-vllm \
  -n demo-llm --timeout=300s
```

### Run Baseline Benchmark

```bash
export VLLM_TARGET="http://qwen-vllm-lb.demo-llm.svc.cluster.local:8000"

guidellm benchmark run \
  --target $VLLM_TARGET \
  --model $MODEL \
  --data prompts-100.csv \
  --rate-type concurrent \
  --rate 100 \
  --max-seconds 300 \
  --output-path vllm-baseline.json
```

### Deploy LLM-D

```bash
# Clean up vLLM
oc delete -k vllm

# Reset Prometheus
oc delete pod -l app=prometheus -n llm-d-monitoring
oc wait --for=condition=ready pod -l app=prometheus -n llm-d-monitoring --timeout=120s

# Deploy LLM-D - included in playbook
oc apply -k llm-d

# Wait for pods
oc wait --for=condition=ready pod -l app.kubernetes.io/name=qwen \
  -n demo-llm --timeout=300s
```

### Run LLM-D Benchmark

```bash
export LLMD_TARGET="http://openshift-ai-inference-openshift-default.openshift-ingress.svc.cluster.local/demo-llm/qwen"

guidellm benchmark run \
  --target $LLMD_TARGET/v1 \
  --model $MODEL \
  --data prompts-100.csv \
  --rate-type concurrent \
  --rate 100 \
  --max-seconds 300 \
  --output-path llm-d-results.json
```

## Expected Results

### vLLM Baseline (Round-Robin)

```
Time to First Token (TTFT):
  P50:        123.22 ms
  P95:        744.71 ms    <-- High tail latency (frustrated users)
  P99:        840.95 ms

First Turn vs Subsequent Turns (Prefix Caching):
  First turn avg:      351.64 ms
  Later turns avg:     196.29 ms
  Speedup ratio:         1.79x   <-- Suboptimal cache reuse
```

### LLM-D (Intelligent Routing)

```
Time to First Token (TTFT):
  P50:         92.09 ms
  P95:        271.60 ms    <-- Significantly lower tail latency
  P99:        674.21 ms

First Turn vs Subsequent Turns (Prefix Caching):
  First turn avg:      361.79 ms
  Later turns avg:      94.22 ms
  Speedup ratio:         3.84x   <-- Excellent cache reuse
```

### Results Comparison

| Metric | vLLM | LLM-D | Improvement |
|--------|------|-------|-------------|
| P50 TTFT | 123 ms | 92 ms | 25% faster |
| P95 TTFT | 745 ms | 272 ms | **63% faster** |
| P99 TTFT | 841 ms | 674 ms | 20% faster |
| Cache Speedup | 1.79x | 3.84x | **2.1x better** |
| Cache Hit Rate | ~25% | ~90%+ | 3.6x better |

### Why LLM-D Performs Better

| Feature | vLLM (Round-Robin) | LLM-D (Intelligent Routing) |
|---------|-------------------|----------------------------|
| **Routing Strategy** | Random/Round-robin | Prefix-aware scoring |
| **Cache Hits** | ~25% (1 in 4 replicas) | ~90%+ (routes to cached replica) |
| **P95 Latency** | High variance | Consistent, lower |
| **GPU Utilization** | Imbalanced | Balanced via KV-cache scoring |

## GuideLLM Parameters Reference

| Parameter | Description | Example |
|-----------|-------------|---------|
| `--target` | Inference endpoint URL | `http://gateway/ns/model` |
| `--model` | Model name for tokenizer | `Qwen/Qwen3-4B` |
| `--processor` | Processor name (optional) | `Qwen/Qwen3-4B` |
| `--data` | Data file or inline JSON | `prompts.csv` or `{"prompt_tokens":1000}` |
| `--rate-type` | `concurrent` or `constant` | `concurrent` |
| `--rate` | Concurrency or RPS | `1,2,4,8,16` |
| `--max-seconds` | Benchmark duration | `300` |
| `--max-requests` | Maximum requests | `1000` |
| `--output-path` | Results file path | `results.json` |

## Disconnected Environment Notes

For benchmarking in disconnected environments:

1. **Copy tokenizer files:**
```bash
oc cp tokenizer_config.json guidellm:/config
oc cp tokenizer.json guidellm:/config
```

2. **Use local tokenizer:**
```bash
guidellm benchmark run \
  --target $TARGET \
  --model /config \
  --processor /config \
  # ... other args
```

3. **Watch benchmark logs:**
```bash
oc logs -f <guidellm-pod>
```

## Interpreting Results

### Healthy Deployment Indicators

- **P95 TTFT < 500ms**: Good tail latency
- **Cache Hit Rate > 80%**: Effective prefix caching (LLM-D)
- **No waiting requests**: Not saturated
- **Consistent ITL**: Stable generation speed

### Warning Signs

| Symptom | Likely Cause | Action |
|---------|--------------|--------|
| P95 >> P50 | Request queueing | Add replicas or GPUs |
| Low cache hit rate | Routing not working | Check scheduler logs |
| High TTFT, low ITL | Prefill bottleneck | Consider P/D disaggregation |
| Increasing ITL | Batch saturation | Reduce concurrency or add replicas |

## Next Steps

- [Performance Debugging](07-performance-debugging.md) if results don't meet expectations
- Review [Advanced Deployment](03-advanced-deployment.md) for optimization options like P/D disaggregation
