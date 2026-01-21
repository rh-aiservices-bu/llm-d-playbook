# Disconnected Installs Guide

This guide covers deploying LLM-D in air-gapped and restricted network environments.

## Overview

Disconnected deployments require:
1. Mirroring operator images to an internal registry
2. Mirroring model images (ModelCar) or setting up model storage
3. Configuring ImageContentSourcePolicy or ImageDigestMirrorSet
4. Additional configuration for Connectivity Link (RHOAI 3.0+)

## Operators to Mirror

Mirror the following operators using `oc-mirror`:

| Operator | Required | Notes |
|----------|----------|-------|
| Red Hat OpenShift AI | Yes | Core requirement |
| Red Hat Service Mesh 3 | Yes | Gateway API dependency |
| Red Hat Serverless Operator | No | Not needed for RawDeployment mode |
| Red Hat Connectivity Link | Yes (3.0+) | Auth and API management |
| Cert Manager | Yes | Certificate management |
| Node Feature Discovery | Yes | GPU node labeling |
| NVIDIA GPU Operator | Yes | GPU drivers and management |
| Leader Worker Set | Optional | Only for MoE models |
| Red Hat Build of Kueue | Optional | Quota management |

## Step 1: Create ImageSetConfiguration

Create `isc-rhoai.yaml`:

```yaml
kind: ImageSetConfiguration
apiVersion: mirror.openshift.io/v1alpha2
storageConfig:
  local:
    path: ./oc-mirror
mirror:
  operators:
    # Update catalog version to match your OCP version (v4.19, v4.20, etc.)
    - catalog: registry.redhat.io/redhat/redhat-operator-index:v4.20
      packages:
        - name: rhods-operator
          channels:
            - name: fast-3.x
        - name: servicemeshoperator3
          channels:
            - name: stable
        - name: openshift-cert-manager-operator
          channels:
            - name: stable
        - name: rhcl-operator
          channels:
            - name: stable
        - name: nfd
          channels:
            - name: stable
        - name: gpu-operator-certified
          channels:
            - name: stable
  additionalImages:
    # LLM-D images (RHOAI 3.0+)
    # Note: Get exact digests from your LLMInferenceServiceConfig:
    #   oc get llminferenceserviceconfig -n redhat-ods-applications -o yaml | grep image:

    # vLLM image (note: rhaiis registry, not rhoai)
    - name: registry.redhat.io/rhaiis/vllm-cuda-rhel9@sha256:ad756c01ec99a99cc7d93401c41b8d92ca96fb1ab7c5262919d818f2be4f3768
    # Scheduler image
    - name: registry.redhat.io/rhoai/odh-llm-d-inference-scheduler-rhel9@sha256:70e900e40845e82ef9ff4ac888afa6a6e835b019ad69aae7297d111e087520a3
    # Routing sidecar image
    - name: registry.redhat.io/rhoai/odh-llm-d-routing-sidecar-rhel9@sha256:e45e6925ad930d9195d906fc922cb956621c98533cc9ad1916c510e8a79c0904
```

> **Important**: The image digests above are from RHOAI 3.0. Always verify current digests by running:
> ```bash
> oc get llminferenceserviceconfig -n redhat-ods-applications -o yaml | grep -E "image:" | sort -u
> ```

## Step 2: Mirror Images

### Set Up Environment

```bash
# Create working directory
mkdir -p scratch
cd scratch

# Copy ImageSetConfiguration
cp ../isc-rhoai.yaml .

# Set registry variables
REGISTRY=your-internal-registry.example.com:5000
```

### Run oc-mirror (Dry Run)

```bash
oc-mirror \
  -c isc-rhoai.yaml \
  --workspace file:///${PWD}/oc-mirror \
  docker://"${REGISTRY}" \
  --v2 \
  --dry-run \
  --authfile pull-secret.txt
```

### Generate Image List

```bash
DATE=$(date +%Y-%m-%d)
sed '
  s@^docker://@@g
  s@=docker://'"${REGISTRY}"'.*@@g
  /localhost/d' \
    oc-mirror/working-dir/dry-run/mapping.txt \
    > images-"${DATE}".txt
```

### Execute Mirror

```bash
oc-mirror \
  -c isc-rhoai.yaml \
  --workspace file:///${PWD}/oc-mirror \
  docker://"${REGISTRY}" \
  --v2 \
  --authfile pull-secret.txt
```

## Step 3: Apply ImageContentSourcePolicy

Apply the generated ICSP/IDMS:

```bash
oc apply -f oc-mirror/working-dir/results-*/imageContentSourcePolicy.yaml
# or for OCP 4.13+
oc apply -f oc-mirror/working-dir/results-*/imageDigestMirrorSet.yaml
```

## Step 4: Mirror RHOAI Additional Images

RHOAI requires additional images beyond the operator catalog. Use the helper script:

```bash
# Clone the helper repository
git clone https://github.com/red-hat-data-services/rhoai-disconnected-install-helper.git
cd rhoai-disconnected-install-helper

# Follow the instructions in the README
```

> **Note**: This step is often challenging and may require manual intervention. Review the image list carefully before mirroring.

## Step 5: Mirror Model Images (ModelCar)

For OCI-based model delivery (ModelCar):

```bash
# Example: Mirror Llama 3.1 8B FP8
skopeo copy --all \
  docker://registry.redhat.io/rhelai1/modelcar-llama-3-1-8b-instruct-fp8-dynamic:1.5 \
  docker://${REGISTRY}/rhelai1/modelcar-llama-3-1-8b-instruct-fp8-dynamic:1.5

# Example: Mirror custom modelcar
skopeo copy --all \
  docker://quay.io/redhat-ai-services/modelcar-catalog:gpt-oss-20b \
  docker://${REGISTRY}/redhat-ai-services/modelcar-catalog:gpt-oss-20b
```

> **Warning**: Large models (70B+) may cause issues as OCI artifacts. Consider using PVC storage for very large models.

## Step 6: Install Operators

### Install in Order

```bash
# 1. Cert Manager
oc apply -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-cert-manager-operator
  namespace: openshift-operators
spec:
  channel: stable
  name: openshift-cert-manager-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF

# Wait for cert-manager
oc wait --for=condition=ready pod -l app.kubernetes.io/name=cert-manager -n cert-manager --timeout=300s

# 2. Service Mesh 3
oc apply -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: servicemeshoperator3
  namespace: openshift-operators
spec:
  channel: stable
  name: servicemeshoperator3
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF

# 3. Connectivity Link (RHOAI 3.0+)
oc apply -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: rhcl-operator
  namespace: openshift-operators
spec:
  channel: stable
  name: rhcl-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```

### Configure Connectivity Link WASM Shim (Disconnected)

For disconnected environments, update the Connectivity Link subscription to include the WASM shim image:

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: rhcl-operator
  namespace: openshift-operators
spec:
  channel: stable
  name: rhcl-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  config:
    env:
      - name: RELATED_IMAGE_WASMSHIM
        value: your-registry.example.com/rhcl-1/wasm-shim-rhel9@sha256:...
```

### Create Kuadrant Instance

```bash
# Create Kuadrant namespace if needed
oc create namespace kuadrant-system || true

# Create Kuadrant instance
oc apply -f - <<EOF
apiVersion: kuadrant.io/v1beta1
kind: Kuadrant
metadata:
  name: kuadrant
  namespace: kuadrant-system
EOF

# Wait for Kuadrant
oc wait Kuadrant -n kuadrant-system kuadrant --for=condition=Ready --timeout=10m
```

### Configure Authorino SSL

```bash
# Add serving cert annotation
oc annotate svc/authorino-authorino-authorization \
  service.beta.openshift.io/serving-cert-secret-name=authorino-server-cert \
  -n kuadrant-system

# Update Authorino with TLS
oc apply -f - <<EOF
apiVersion: operator.authorino.kuadrant.io/v1beta1
kind: Authorino
metadata:
  name: authorino
  namespace: kuadrant-system
spec:
  replicas: 1
  clusterWide: true
  listener:
    tls:
      enabled: true
      certSecretRef:
        name: authorino-server-cert
  oidcServer:
    tls:
      enabled: false
EOF

# Wait for Authorino pods
oc wait --for=condition=ready pod -l authorino-resource=authorino \
  -n kuadrant-system --timeout=150s
```

> **Important**: If OpenShift AI was installed before setting up Red Hat Connectivity Link, you must restart the pods in `redhat-ods-applications` namespace for the auth configuration to take effect:
> ```bash
> oc delete pods --all -n redhat-ods-applications
> oc wait --for=condition=ready pod -l control-plane=kserve-controller-manager \
>   -n redhat-ods-applications --timeout=300s
> ```

### Install OpenShift AI

```bash
oc apply -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: rhods-operator
  namespace: redhat-ods-operator
spec:
  channel: stable
  name: rhods-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```

## Step 7: Configure Gateway (Bare Metal)

For disconnected bare metal environments, configure MetalLB first:

```bash
# Install MetalLB operator
oc apply -k gitops/operators/metallb-operator

# Configure IP pool (adjust for your network)
oc apply -f - <<EOF
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: llm-d-pool
  namespace: metallb-system
spec:
  addresses:
    - 10.0.0.100-10.0.0.110
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: llm-d-l2
  namespace: metallb-system
spec:
  ipAddressPools:
    - llm-d-pool
EOF
```

### Create Gateway

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
          - kind: Secret
            name: default-gateway-tls
```

## Step 8: Deploy LLMInferenceService

### Using Mirrored ModelCar

```yaml
apiVersion: serving.kserve.io/v1alpha1
kind: LLMInferenceService
metadata:
  annotations:
    opendatahub.io/hardware-profile-name: gpu-profile
    opendatahub.io/hardware-profile-namespace: redhat-ods-applications
    security.opendatahub.io/enable-auth: 'false'  # Required if no Connectivity Link
  name: llama-llm-d
  namespace: demo-llm
  labels:
    opendatahub.io/dashboard: 'true'
spec:
  model:
    name: llama-3-1-8b-instruct-fp8
    uri: 'oci://your-registry.example.com/rhelai1/modelcar-llama-3-1-8b-instruct-fp8-dynamic:1.5'
  replicas: 1
  router:
    gateway: {}
    route: {}
    scheduler: {}
  template:
    containers:
      - name: main
        env:
          - name: VLLM_ADDITIONAL_ARGS
            value: '--disable-uvicorn-access-log --max-model-len=32768'
        resources:
          limits:
            cpu: '4'
            memory: 16Gi
            nvidia.com/gpu: '1'
          requests:
            cpu: '1'
            memory: 6Gi
            nvidia.com/gpu: '1'
        livenessProbe:
          failureThreshold: 5
          httpGet:
            path: /health
            port: 8000
            scheme: HTTPS
          initialDelaySeconds: 120
          periodSeconds: 30
          timeoutSeconds: 30
    tolerations:
      - effect: NoSchedule
        key: nvidia.com/gpu
        operator: Exists
```

### Using PVC for Large Models

For models larger than 70B, use PVC storage:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: llama-70b-storage
  namespace: demo-llm
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 200Gi
---
apiVersion: serving.kserve.io/v1alpha1
kind: LLMInferenceService
metadata:
  name: llama-70b
  namespace: demo-llm
spec:
  model:
    name: meta-llama/Llama-3.1-70B-Instruct
    uri: 'pvc://llama-70b-storage/models/llama-70b'
  # ... rest of configuration
```

## Known Issues and Pain Points

### Documentation Gaps

The official documentation doesn't fully list all dependencies that need mirroring. Plan to discover additional requirements during installation.

### Certificate Handling

Ensure your internal registry certificates are trusted by the cluster:

```bash
# Add CA bundle to cluster
oc create configmap registry-ca \
  --from-file=ca-bundle.crt=/path/to/ca.crt \
  -n openshift-config

oc patch image.config.openshift.io/cluster \
  --type=merge \
  -p '{"spec":{"additionalTrustedCA":{"name":"registry-ca"}}}'
```

### Model Ingestion Challenges

- HuggingFace models require internet access by default
- Large OCI artifacts (70B+) may timeout or fail
- Consider pre-populating PVCs with model weights

### RHOAI Additional Images

The additional images collection often requires manual intervention:
- Some images may not be in the documented list
- Version mismatches can occur
- Test thoroughly in a staging environment first

### Connectivity Link in Disconnected

If auth is not required, explicitly disable it:

```yaml
metadata:
  annotations:
    security.opendatahub.io/enable-auth: 'false'
```

## Validation Script

```bash
#!/bin/bash
echo "=== Disconnected Environment Validation ==="

# Check image sources
echo "Checking image sources..."
oc get pods -A -o jsonpath='{range .items[*]}{.spec.containers[*].image}{"\n"}{end}' | \
  grep -v "your-registry.example.com" | \
  sort -u

# Check for pending images
echo "Checking for ImagePullBackOff..."
oc get pods -A | grep -E "ImagePull|ErrImagePull"

# Verify operators
echo "Checking operators..."
oc get csv -A | grep -E "rhods|servicemesh|connectivity|cert-manager"
```

## Next Steps

- [Running Benchmarks](06-running-benchmarks.md) to validate deployment
- [Performance Debugging](07-performance-debugging.md) for optimization
