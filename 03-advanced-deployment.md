# Advanced Deployment Guide

This guide covers advanced deployment scenarios including bare metal installations, MetalLB configuration, custom Gateway setups, and prefill/decode disaggregation.

All artifacts referenced in this guide are included in the playbook's `gitops/` directory.

## Bare Metal Deployments

### Why MetalLB is Required

LLM-D requires the Gateway API which recommends `Service` objects with `type: LoadBalancer`. Cloud environments automatically provision external IPs, but bare metal clusters require MetalLB to provide this functionality.

> **Note**: While it's possible to use `type: ClusterIP` with manual exposure methods, this is not recommended and would require a Support Exception.

### Installing MetalLB Operator

```bash
# Install MetalLB operator
oc apply -k gitops/operators/metallb-operator

# Wait for operator to be ready
oc wait --for=condition=ready pod -l control-plane=controller-manager -n metallb-system --timeout=300s
```

### Configure MetalLB Instance

```bash
# Apply base MetalLB configuration
oc apply -k gitops/instance/metallb-operator/base
```

Base configuration creates:
```yaml
apiVersion: metallb.io/v1beta1
kind: MetalLB
metadata:
  name: metallb
  namespace: metallb-system
```

### Configure IP Address Pool

Create an IP pool with addresses available on your network:

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: llm-d-pool
  namespace: metallb-system
spec:
  addresses:
    - 192.168.1.240-192.168.1.250  # Adjust for your network
```

### Configure L2 Advertisement

For simple L2 (layer 2) networks:

```yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: llm-d-l2-advertisement
  namespace: metallb-system
spec:
  ipAddressPools:
    - llm-d-pool
```

### Configure BGP Advertisement (Advanced)

For environments using BGP routing:

```yaml
apiVersion: metallb.io/v1beta1
kind: BGPPeer
metadata:
  name: llm-d-bgp-peer
  namespace: metallb-system
spec:
  myASN: 64500
  peerASN: 64501
  peerAddress: 10.0.0.1
---
apiVersion: metallb.io/v1beta1
kind: BGPAdvertisement
metadata:
  name: llm-d-bgp-advertisement
  namespace: metallb-system
spec:
  ipAddressPools:
    - llm-d-pool
```

### Verify MetalLB Configuration

```bash
# Check MetalLB pods
oc get pods -n metallb-system

# Check IP pools
oc get ipaddresspool -n metallb-system

# Verify Gateway gets external IP
oc get svc -n openshift-ingress | grep openshift-ai-inference
```

## Custom Gateway Configuration

### Basic Gateway with Namespace Restrictions

Restrict which namespaces can use the Gateway to prevent HTTPRoute hijacking:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: openshift-ai-inference
spec:
  controllerName: openshift.io/gateway-controller/v1
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: openshift-ai-inference
  namespace: openshift-ingress
spec:
  gatewayClassName: openshift-ai-inference
  listeners:
    - name: http
      port: 80
      protocol: HTTP
      allowedRoutes:
        namespaces:
          from: Selector
          selector:
            matchExpressions:
              - key: kubernetes.io/metadata.name
                operator: In
                values:
                  - openshift-ingress
                  - redhat-ods-applications
                  - demo-llm  # Add your namespaces here
```

### HTTPS Gateway with TLS

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  labels:
    istio.io/rev: openshift-gateway
  name: openshift-ai-inference
  namespace: openshift-ingress
spec:
  gatewayClassName: openshift-ai-inference
  listeners:
    - name: https
      port: 443
      protocol: HTTPS
      hostname: inference-gateway.apps.example.com
      allowedRoutes:
        namespaces:
          from: Selector
          selector:
            matchExpressions:
              - key: kubernetes.io/metadata.name
                operator: In
                values:
                  - openshift-ingress
                  - redhat-ods-applications
                  - demo-llm
      tls:
        mode: Terminate
        certificateRefs:
          - group: ''
            kind: Secret
            name: gateway-tls-secret
```

### Per-Namespace Gateway (HTTPRoute Hijacking Mitigation)

For multi-tenant environments, create separate Gateways per namespace:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: tenant-a-gateway
  namespace: tenant-a
spec:
  gatewayClassName: openshift-ai-inference
  listeners:
    - name: http
      port: 80
      protocol: HTTP
      allowedRoutes:
        namespaces:
          from: Same
```

Reference this Gateway in the LLMInferenceService:

```yaml
apiVersion: serving.kserve.io/v1alpha1
kind: LLMInferenceService
metadata:
  name: my-model
  namespace: tenant-a
spec:
  router:
    gateway:
      ref:
        - name: tenant-a-gateway
          namespace: tenant-a
```

## Prefill/Decode Disaggregation

Prefill/Decode (P/D) disaggregation separates the compute-intensive prefill phase from the memory-bandwidth-bound decode phase for improved performance.

### Requirements

- **High-speed networking**: InfiniBand or RoCE recommended
- **Multiple GPUs**: Separate pools for prefill and decode
- **RHOAI 2.25+**: P/D support included

### Basic P/D Configuration

```yaml
apiVersion: serving.kserve.io/v1alpha1
kind: LLMInferenceService
metadata:
  name: my-model-pd
  namespace: demo-llm
spec:
  replicas: 2  # Decode replicas
  model:
    uri: oci://quay.io/redhat-ai-services/modelcar-catalog:llama-3-1-8b
    name: meta-llama/Llama-3.1-8B-Instruct
  router:
    gateway: {}
    route: {}
    scheduler: {}
  # Main template becomes "Decode" instances
  template:
    containers:
      - name: main
        env:
          - name: VLLM_ADDITIONAL_ARGS
            value: "--disable-uvicorn-access-log --kv-transfer-config '{\"kv_connector\":\"NixlConnector\",\"kv_role\":\"kv_both\"}' --block-size 128"
          - name: VLLM_NIXL_SIDE_CHANNEL_HOST
            valueFrom:
              fieldRef:
                fieldPath: status.podIP
        resources:
          limits:
            nvidia.com/gpu: '1'
          requests:
            nvidia.com/gpu: '1'
    tolerations:
      - effect: NoSchedule
        key: nvidia.com/gpu
        operator: Exists
  # Prefill instances
  prefill:
    replicas: 2
    template:
      containers:
        - name: main
          env:
            - name: VLLM_ADDITIONAL_ARGS
              value: "--disable-uvicorn-access-log --kv-transfer-config '{\"kv_connector\":\"NixlConnector\",\"kv_role\":\"kv_both\"}' --block-size 128"
            - name: VLLM_NIXL_SIDE_CHANNEL_HOST
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
          resources:
            limits:
              nvidia.com/gpu: '1'
            requests:
              nvidia.com/gpu: '1'
      tolerations:
        - effect: NoSchedule
          key: nvidia.com/gpu
          operator: Exists
```

### P/D with InfiniBand/RoCE

For optimal KV cache transfer performance:

```yaml
apiVersion: serving.kserve.io/v1alpha1
kind: LLMInferenceService
metadata:
  name: my-model-pd-ib
  namespace: demo-llm
  annotations:
    k8s.v1.cni.cncf.io/networks: roce-p2  # Your RoCE network attachment
spec:
  template:
    containers:
      - name: main
        env:
          - name: KSERVE_INFER_ROCE
            value: "true"
          - name: UCX_PROTO_INFO
            value: "y"  # Enable debug logging
          - name: VLLM_ADDITIONAL_ARGS
            value: "--disable-uvicorn-access-log --kv-transfer-config '{\"kv_connector\":\"NixlConnector\",\"kv_role\":\"kv_both\"}' --block-size 128"
          - name: VLLM_NIXL_SIDE_CHANNEL_HOST
            valueFrom:
              fieldRef:
                fieldPath: status.podIP
        resources:
          limits:
            nvidia.com/gpu: '1'
            rdma/roce_gdr: 1
          requests:
            nvidia.com/gpu: '1'
            rdma/roce_gdr: 1
  prefill:
    template:
      containers:
        - name: main
          env:
            - name: KSERVE_INFER_ROCE
              value: "true"
            - name: UCX_PROTO_INFO
              value: "y"
            - name: VLLM_ADDITIONAL_ARGS
              value: "--disable-uvicorn-access-log --kv-transfer-config '{\"kv_connector\":\"NixlConnector\",\"kv_role\":\"kv_both\"}' --block-size 128"
            - name: VLLM_NIXL_SIDE_CHANNEL_HOST
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
          resources:
            limits:
              nvidia.com/gpu: '1'
              rdma/roce_gdr: 1
            requests:
              nvidia.com/gpu: '1'
              rdma/roce_gdr: 1
```

> **Warning**: Without InfiniBand/RoCE, KV cache transfer falls back to TCP, resulting in significantly degraded performance.

### Custom EndpointPicker for P/D

For RHOAI 2.25/3.0, you must manually configure the EndpointPicker for P/D:

```yaml
apiVersion: serving.kserve.io/v1alpha1
kind: LLMInferenceService
metadata:
  name: my-model-pd
spec:
  router:
    scheduler:
      endpointPickerConfig: |
        apiVersion: inference.networking.x-k8s.io/v1alpha1
        kind: EndpointPickerConfig
        plugins:
        - type: pd-profile-handler
          config:
            threshold: 500  # Requests below this use decode-only
        - type: prefill-filter
        - type: decode-filter
        - type: prefix-cache-scorer
        - type: load-aware-scorer
        - type: max-score-picker
        schedulingProfiles:
        - name: prefill
          plugins:
          - pluginRef: prefill-filter
          - pluginRef: load-aware-scorer
            weight: 1.0
          - pluginRef: max-score-picker
        - name: decode
          plugins:
          - pluginRef: decode-filter
          - pluginRef: prefix-cache-scorer
            weight: 2.0
          - pluginRef: load-aware-scorer
            weight: 1.0
          - pluginRef: max-score-picker
```

## Advanced vLLM Configuration

### Custom Probes for Large Models

Large models may require extended startup times:

```yaml
spec:
  template:
    containers:
      - name: main
        livenessProbe:
          initialDelaySeconds: 10
          periodSeconds: 30
          timeoutSeconds: 30
          failureThreshold: 5
        startupProbe:
          httpGet:
            path: /health
            port: 8000
            scheme: HTTPS
          initialDelaySeconds: 15
          timeoutSeconds: 10
          periodSeconds: 10
          failureThreshold: 60  # Allow up to 10 minutes for startup
```

### Setting Max Model Length

```yaml
spec:
  template:
    containers:
      - name: main
        env:
          - name: VLLM_ADDITIONAL_ARGS
            value: "--disable-uvicorn-access-log --max-model-len=32768"
```

### Multi-GPU with Tensor Parallelism

```yaml
spec:
  template:
    containers:
      - name: main
        env:
          - name: VLLM_ADDITIONAL_ARGS
            value: "--disable-uvicorn-access-log --tensor-parallel-size=4"
        resources:
          limits:
            nvidia.com/gpu: '4'
          requests:
            nvidia.com/gpu: '4'
```

## Authentication Configuration (RHOAI 3.0+)

### Enable Authentication

```yaml
apiVersion: serving.kserve.io/v1alpha1
kind: LLMInferenceService
metadata:
  name: my-model
  annotations:
    security.opendatahub.io/enable-auth: 'true'
```

### Test with Authentication

```bash
# Get token
TOKEN=$(oc whoami --show-token)

# Make authenticated request
curl -s http://${INFERENCE_URL}/demo-llm/my-model/v1/models \
  -H "Authorization: Bearer ${TOKEN}" | jq
```

### Disable Authentication (Development Only)

```yaml
apiVersion: serving.kserve.io/v1alpha1
kind: LLMInferenceService
metadata:
  name: my-model
  annotations:
    security.opendatahub.io/enable-auth: 'false'
```

> **Warning**: Auth is broken in RHOAI 3.0. If Connectivity Link is not installed, you must explicitly set `enable-auth: 'false'`.

## Dashboard Display Labels

Configure labels for RHOAI Dashboard visibility:

```yaml
apiVersion: serving.kserve.io/v1alpha1
kind: LLMInferenceService
metadata:
  name: my-model
  annotations:
    opendatahub.io/connections: my-model
    opendatahub.io/hardware-profile-name: nvidia-gpu-serving
    opendatahub.io/hardware-profile-namespace: redhat-ods-applications
    opendatahub.io/model-type: generative
    openshift.io/display-name: My Custom Model
  labels:
    opendatahub.io/dashboard: "true"
```

## Next Steps

- [Automated Deployment](04-automated-deployment.md) for GitOps patterns
- [Running Benchmarks](06-running-benchmarks.md) to validate performance
- [Performance Debugging](07-performance-debugging.md) for optimization
