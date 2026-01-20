# Quick Start Guide

Deploy LLM-D on a connected OpenShift cluster in minutes.

## Prerequisites

Ensure you have completed [Pre-flight Validation](01-preflight-validation.md) before proceeding.

## Step 1: Configure the Gateway

Create the GatewayClass and Gateway for LLM-D:

```bash
oc apply -k gitops/instance/llm-d/gateway
```

Verify the Gateway is ready:

```bash
oc get gateway -n openshift-ingress

# Expected output:
# NAME                     CLASS              ADDRESS   PROGRAMMED   AGE
# openshift-ai-inference   openshift-default  ...       True         ...
```

## Step 2: Create Namespace

```bash
oc apply -k gitops/instance/llm-d/namespace
```

Or manually:

```bash
oc create namespace demo-llm
```

## Step 3: Deploy an LLMInferenceService

### Option A: Using ModelCar (OCI Container)

```bash
oc apply -k gitops/instance/llm-d/intelligent-inference/gpt-oss-20b/overlays/modelcar
```

### Option B: Using HuggingFace

```bash
oc apply -k gitops/instance/llm-d/intelligent-inference/gpt-oss-20b/overlays/huggingface
```

### Option C: Manual Deployment

Create an LLMInferenceService directly:

```yaml
apiVersion: serving.kserve.io/v1alpha1
kind: LLMInferenceService
metadata:
  name: gpt-oss-20b
  namespace: demo-llm
spec:
  replicas: 2
  model:
    uri: oci://quay.io/redhat-ai-services/modelcar-catalog:gpt-oss-20b
    name: openai/gpt-oss-20b
  router:
    gateway: {}
    route: {}
    scheduler: {}
  template:
    containers:
      - name: main
        env:
          - name: VLLM_ADDITIONAL_ARGS
            value: "--disable-uvicorn-access-log"
        resources:
          limits:
            cpu: '4'
            memory: 16Gi
            nvidia.com/gpu: '1'
          requests:
            cpu: '1'
            memory: 8Gi
            nvidia.com/gpu: '1'
    tolerations:
      - effect: NoSchedule
        key: nvidia.com/gpu
        operator: Exists
```

Apply the manifest:

```bash
oc apply -f llminferenceservice.yaml
```

## Step 4: Verify Deployment

### Check LLMInferenceService Status

```bash
oc get llminferenceservice -w -n demo-llm

# Expected output:
# NAME          URL                                              READY   AGE
# gpt-oss-20b   http://<gateway-url>/demo-llm/gpt-oss-20b       True    5m
```

### Check Pods

```bash
oc get pods -w -n demo-llm

# Expected output:
# NAME                                            READY   STATUS    AGE
# gpt-oss-20b-kserve-xxxxx-xxxxx                 1/1     Running   3m
# gpt-oss-20b-kserve-xxxxx-xxxxx                 1/1     Running   3m
# gpt-oss-20b-kserve-router-scheduler-xxxxx      1/1     Running   3m
```

### Watch Pod Logs

```bash
# Watch vLLM server logs
oc logs -f -l app.kubernetes.io/name=gpt-oss-20b -n demo-llm

# Watch scheduler logs
oc logs -f -l app.kubernetes.io/component=router-scheduler -n demo-llm
```

## Step 5: Test the Endpoint

### Get the Inference URL

```bash
INFERENCE_URL=$(oc -n openshift-ingress get gateway openshift-ai-inference \
  -o jsonpath='{.status.addresses[0].value}')
echo "Inference URL: http://${INFERENCE_URL}"
```

### List Available Models

```bash
curl -s http://${INFERENCE_URL}/demo-llm/gpt-oss-20b/v1/models | jq
```

### Send a Completion Request

```bash
curl -s -X POST http://${INFERENCE_URL}/demo-llm/gpt-oss-20b/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "openai/gpt-oss-20b",
    "prompt": "Explain the difference between supervised and unsupervised learning.",
    "max_tokens": 200,
    "temperature": 0.7
  }' | jq '.choices[0].text'
```

### Send a Chat Completion Request

```bash
curl -s -X POST http://${INFERENCE_URL}/demo-llm/gpt-oss-20b/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "openai/gpt-oss-20b",
    "messages": [
      {"role": "system", "content": "You are a helpful assistant."},
      {"role": "user", "content": "Write a haiku about Kubernetes."}
    ]
  }' | jq '.choices[0].message.content'
```

## Step 6: Deploy Monitoring (Optional)

Deploy Prometheus and Grafana for performance monitoring:

```bash
until oc apply -k gitops/instance/llm-d-monitoring; do : ; done

# Get Grafana URL
oc get route grafana -n llm-d-monitoring -o jsonpath='{.spec.host}'
```

Access Grafana with default credentials: `admin` / `admin`

## Quick Start Summary

| Step | Command | Verification |
|------|---------|--------------|
| 1. Configure Gateway | `oc apply -k gitops/instance/llm-d/gateway` | `oc get gateway -n openshift-ingress` |
| 2. Create namespace | `oc apply -k gitops/instance/llm-d/namespace` | `oc get ns demo-llm` |
| 3. Deploy model | `oc apply -k gitops/instance/llm-d/intelligent-inference/gpt-oss-20b/overlays/modelcar` | `oc get llminferenceservice -n demo-llm` |
| 4. Test endpoint | `curl http://${INFERENCE_URL}/demo-llm/gpt-oss-20b/v1/models` | JSON response |

## Cleanup

```bash
# Delete LLMInferenceService
oc delete llminferenceservice gpt-oss-20b -n demo-llm

# Delete namespace
oc delete ns demo-llm
```

## Troubleshooting Quick Start Issues

### Pods Stuck in Pending

```bash
# Check events
oc get events -n demo-llm --sort-by='.lastTimestamp'

# Common causes:
# - Insufficient GPU resources
# - Missing tolerations
# - PVC not binding
```

### Model Download Slow/Failed

```bash
# Check vLLM container logs
oc logs -f <pod-name> -n demo-llm -c main

# For HuggingFace models, ensure HF_TOKEN is set if required
```

### Gateway Not Routing

```bash
# Check HTTPRoute
oc get httproute -n demo-llm

# Verify Gateway listener
oc describe gateway openshift-ai-inference -n openshift-ingress
```

## Next Steps

- [Advanced Deployment](03-advanced-deployment.md) for bare metal and custom configurations
- [Running Benchmarks](06-running-benchmarks.md) to test performance
- [Performance Debugging](07-performance-debugging.md) if you encounter performance issues
