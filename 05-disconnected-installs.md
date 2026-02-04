# Disconnected Installs Guide

This guide covers deploying LLM-D in air-gapped and restricted network environments.

## Overview

Disconnected deployments require:
1. Mirroring operator images to an internal registry
2. Mirroring model images (ModelCar) or setting up model storage
3. Configuring ImageContentSourcePolicy or ImageDigestMirrorSet
4. Additional configuration for Connectivity Link (RHOAI 3.0+)

### Mirroring Approaches

There are two general approaches to disconnected deployments:

| Approach | Description | Security Level |
|----------|-------------|----------------|
| **Full Mirror** | Mirror all images to an offline registry using a "jump box" with internet access | Strict air-gap |
| **Pull-Through Cache** | Configure a container registry to proxy and cache images on-demand | Relaxed disconnected |

This guide focuses on full mirroring for strict environments. For pull-through caching, consult your registry documentation (Quay, Nexus, Artifactory, etc.).

### The Mirroring Process

1. Set up a "jump box" with access to both the internet and your internal registry
2. Use `oc-mirror` to generate an image list from the operator catalog
3. Mirror the images to your internal registry
4. Apply ImageContentSourcePolicy/ImageDigestMirrorSet to redirect image pulls

> **Reference**: [Deploying OpenShift AI in a Disconnected Environment](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.0/html/installing_and_uninstalling_openshift_ai_self-managed/deploying-openshift-ai-in-a-disconnected-environment_install)

## Operators to Mirror

> **Warning**: The official RHOAI documentation does not fully list all operator dependencies. Users often discover missing dependencies during installation. The list below is based on field experience.

Mirror the following operators using `oc-mirror`:

| Operator | Required | Notes |
|----------|----------|-------|
| Red Hat OpenShift AI | Yes | Core requirement |
| Red Hat Service Mesh 3 | Yes | Gateway API dependency |
| Red Hat Connectivity Link | Yes (3.0+) | Auth and API management |
| Cert Manager | Yes | Certificate management (dependency for others) |
| Node Feature Discovery | Yes | GPU node labeling |
| NVIDIA GPU Operator | Yes | GPU drivers and management |
| Leader Worker Set | Optional | Only for MoE models |
| Red Hat Build of Kueue | Optional | Quota management |
| Red Hat Serverless Operator | No | Not needed for RawDeployment mode |

### Dependency Chain

The operators have dependencies that must be installed in order:

```
Cert Manager
    └── Service Mesh 3
            └── Connectivity Link
                    └── OpenShift AI (RHOAI)

Node Feature Discovery
    └── NVIDIA GPU Operator
```

> **Note**: Service Mesh 3 and Connectivity Link may have their own image dependencies beyond the operator catalog. If you encounter `ImagePullBackOff` errors after installation, check the pod events to identify additional images that need mirroring.

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

RHOAI requires additional images beyond the operator catalog. This is documented in [Step 4 of the official docs](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.0/html/installing_and_uninstalling_openshift_ai_self-managed/deploying-openshift-ai-in-a-disconnected-environment_install) and uses a helper repository:

```bash
# Clone the helper repository
git clone https://github.com/red-hat-data-services/rhoai-disconnected-install-helper.git
cd rhoai-disconnected-install-helper

# Follow the instructions in the README
```

> **Warning**: This step is a known pain point:
> - The helper script may not include all required images
> - Version mismatches between RHOAI and the helper can occur
> - Some images may need to be discovered through `ImagePullBackOff` errors
> - **Test thoroughly in a staging environment before production**
>
> This is often where customers encounter unexpected issues - plan for troubleshooting time.

## Step 5: Mirror Model Images (ModelCar)

For OCI-based model delivery (ModelCar):

> **Important**: Model images are NOT automatically mirrored with operators. You must explicitly identify and mirror each model you need.

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

> **Warning**:
> - Large models (70B+) often fail as OCI artifacts due to timeouts and size limits - use PVC storage instead
> - Model OCI URLs must be discovered through the RHOAI Dashboard UI - they are not listed on the RH Container Catalog website
> - Mirroring the full Model Catalog would be 100s of TB - only mirror what you need

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

For models larger than 70B, use PVC storage instead of OCI. This avoids registry size limits, timeout issues, and layer decompression challenges.

#### Why PVC for Large Models?

| Method | Pros | Cons |
|--------|------|------|
| OCI (ModelCar) | Simple, immutable, versioned | Size limits, timeouts, storage overhead |
| PVC | No size limits, faster startup | Manual management, storage provisioning |

**Recommendation**: Use OCI for models ≤30B, PVC for models ≥70B.

#### Pre-populate PVC from Connected Environment

```bash
# Option 1: Download from HuggingFace to PVC (from connected jump box)
# Create a temporary pod with PVC mounted
oc run model-downloader --rm -it \
  --image=python:3.11-slim \
  --overrides='{"spec":{"volumes":[{"name":"model-vol","persistentVolumeClaim":{"claimName":"llama-70b-storage"}}],"containers":[{"name":"model-downloader","image":"python:3.11-slim","volumeMounts":[{"mountPath":"/models","name":"model-vol"}]}]}}' \
  -- bash -c "pip install huggingface_hub && huggingface-cli download meta-llama/Llama-3.1-70B-Instruct --local-dir /models/llama-70b"

# Option 2: Copy from local storage to PVC using rsync pod
oc rsync ./llama-70b/ model-downloader:/models/llama-70b/
```

#### PVC Configuration

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: llama-70b-storage
  namespace: demo-llm
spec:
  accessModes:
    - ReadWriteOnce  # Use ReadWriteMany if multiple replicas need access
  resources:
    requests:
      storage: 200Gi  # Adjust based on model size + buffer
  storageClassName: your-storage-class  # Use fast storage if available
---
apiVersion: serving.kserve.io/v1alpha1
kind: LLMInferenceService
metadata:
  name: llama-70b
  namespace: demo-llm
  annotations:
    security.opendatahub.io/enable-auth: 'false'
spec:
  model:
    name: meta-llama/Llama-3.1-70B-Instruct
    uri: 'pvc://llama-70b-storage/llama-70b'
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
            value: '--disable-uvicorn-access-log --tensor-parallel-size=4'
        resources:
          limits:
            nvidia.com/gpu: '4'  # 70B typically needs 4+ GPUs
          requests:
            nvidia.com/gpu: '4'
    tolerations:
      - effect: NoSchedule
        key: nvidia.com/gpu
        operator: Exists
```

> **Note**: For PVC-based models with multiple replicas, ensure your storage class supports `ReadWriteMany` access mode, or use a separate PVC per replica.

## Known Issues and Pain Points

### Documentation Gaps

The official documentation doesn't fully list all dependencies that need mirroring. Plan to discover additional requirements during installation.

### Certificate Handling

Disconnected environments typically use self-signed certificates. This requires configuration at multiple levels.

#### Cluster-Wide Registry Trust

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

#### Ingress Certificate Configuration

OpenShift should be configured to use your organization's CA for ingress. When properly configured, RHOAI components should automatically trust the cluster's ingress certificates.

> **Reference**: [Configuring Ingress Cluster Traffic](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/security_and_compliance/configuring-certificates)

In an ideal configuration:
1. OpenShift is configured with a self-signed cert (or corporate CA)
2. The cluster trusts that CA via the `openshift-config/user-ca-bundle` ConfigMap
3. RHOAI and LLM-D components inherit this trust automatically

> **Note**: In earlier RHOAI versions, separate certificate configuration was required. Current versions (2.25+) should inherit cluster certificate trust. If you encounter TLS errors, check that your cluster's CA bundle is correctly configured.

### Model Ingestion Challenges

Getting models into a disconnected environment is one of the most challenging aspects:

#### HuggingFace Models

- Require internet access by default
- Policy restrictions on downloading models may apply
- Need a "jump box" with both HuggingFace access and internal storage access
- Some models require acceptance of license agreements

#### Red Hat Model Catalog (OCI ModelCar)

The RH Model Catalog provides models as OCI containers, which is more air-gap friendly:

**Challenges:**
- **Discovery is difficult**: Model OCI URLs are only visible through the RHOAI Dashboard UI
- Models are NOT listed on the Red Hat Container Catalog website
- The actual OCI URL is buried in the model card within the Dashboard
- The Dashboard "Deploy" button only supports vanilla vLLM (not LLM-D at this time)
- Model Catalog depends on Model Registry, which some customers don't want

**Mirroring Considerations:**
- Model images may NOT be automatically included in the standard mirroring process
- The full catalog could be 100s of TB - selective mirroring is essential
- Manually identify and mirror only the models you need

```bash
# Example: List models available in the catalog
# (Must be done from a connected environment)
skopeo list-tags docker://registry.redhat.io/rhelai1/modelcar-catalog

# Mirror a specific model
skopeo copy --all \
  docker://registry.redhat.io/rhelai1/modelcar-llama-3-1-8b-instruct-fp8-dynamic:1.5 \
  docker://${REGISTRY}/rhelai1/modelcar-llama-3-1-8b-instruct-fp8-dynamic:1.5
```

#### Large Models (70B+)

- OCI artifacts this size often encounter timeouts, registry limits, or storage issues
- **Recommendation**: Use PVC storage instead of OCI for 70B+ models
- Pre-populate PVCs from a connected staging environment

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
