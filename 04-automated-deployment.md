# Automated Deployment Guide

This guide covers GitOps patterns and automation strategies for LLM-D deployments.

## Repository Structure

The demo-ocp-llm-d repository provides a Kustomize-based structure for automated deployments:

```
gitops/
├── operators/               # Operator installations
│   ├── metallb-operator/
│   ├── servicemeshoperator3/
│   ├── serverless/
│   ├── cert-manager/
│   ├── rhoai/
│   └── leader-worker-set/
├── instance/               # Instance configurations
│   ├── llm-d/
│   │   ├── gateway/
│   │   ├── intelligent-inference/
│   │   ├── pd-disaggregation/
│   │   └── namespace/
│   ├── metallb-operator/
│   │   ├── base/
│   │   ├── l2/
│   │   └── bgp/
│   └── llm-d-monitoring/
├── ocp-4.19/               # OCP version-specific configs
│   ├── configs/
│   └── prereqs/
└── disconnected/           # Disconnected environment configs
```

## Kustomize-Based Deployment

### Full Stack Deployment

Deploy all prerequisites and configurations:

```bash
# Install OCP 4.19 prerequisites
until oc apply -k gitops/ocp-4.19; do : ; done

# Wait for operators
watch oc get csv -A

# Deploy Gateway
oc apply -k gitops/instance/llm-d/gateway

# Deploy namespace and model
oc apply -k gitops/instance/llm-d/namespace
oc apply -k gitops/instance/llm-d/intelligent-inference/gpt-oss-20b/overlays/modelcar
```

### Modular Deployment

Deploy components individually:

```bash
# Just operators
oc apply -k gitops/operators/servicemeshoperator3
oc apply -k gitops/operators/cert-manager
oc apply -k gitops/operators/rhoai

# Just MetalLB (bare metal)
oc apply -k gitops/operators/metallb-operator
oc apply -k gitops/instance/metallb-operator/base
oc apply -k gitops/instance/metallb-operator/l2  # or bgp
```

## Creating Custom Overlays

### Model Overlay Structure

```
gitops/instance/llm-d/intelligent-inference/my-model/
├── base/
│   ├── kustomization.yaml
│   └── llminferenceservice.yaml
└── overlays/
    ├── hf/
    │   └── kustomization.yaml
    ├── modelcar/
    │   └── kustomization.yaml
    └── pvc/
        └── kustomization.yaml
```

### Base Configuration

`base/kustomization.yaml`:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - llminferenceservice.yaml
```

`base/llminferenceservice.yaml`:
```yaml
apiVersion: serving.kserve.io/v1alpha1
kind: LLMInferenceService
metadata:
  name: my-model
  namespace: demo-llm
spec:
  replicas: 2
  model:
    name: my-org/my-model
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

### ModelCar Overlay

`overlays/modelcar/kustomization.yaml`:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
patches:
  - patch: |
      - op: add
        path: /spec/model/uri
        value: oci://quay.io/my-org/modelcar:my-model
    target:
      kind: LLMInferenceService
      name: my-model
```

### HuggingFace Overlay

`overlays/hf/kustomization.yaml`:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
patches:
  - patch: |
      - op: add
        path: /spec/model/uri
        value: hf://my-org/my-model
    target:
      kind: LLMInferenceService
      name: my-model
```

### PVC Overlay

`overlays/pvc/kustomization.yaml`:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
  - pvc.yaml
patches:
  - patch: |
      - op: add
        path: /spec/model/uri
        value: pvc://model-storage/my-model
    target:
      kind: LLMInferenceService
      name: my-model
```

`overlays/pvc/pvc.yaml`:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: model-storage
  namespace: demo-llm
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
```

## ArgoCD Integration

### Application Definition

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: llm-d-demo
  namespace: openshift-gitops
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/llm-d-config.git
    targetRevision: main
    path: gitops/instance/llm-d/intelligent-inference/gpt-oss-20b/overlays/modelcar
  destination:
    server: https://kubernetes.default.svc
    namespace: demo-llm
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### ApplicationSet for Multiple Models

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: llm-d-models
  namespace: openshift-gitops
spec:
  generators:
    - list:
        elements:
          - model: gpt-oss-20b
            namespace: demo-llm
            overlay: modelcar
          - model: qwen3-0.6b
            namespace: demo-llm-small
            overlay: hf
  template:
    metadata:
      name: 'llm-d-{{model}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/your-org/llm-d-config.git
        targetRevision: main
        path: 'gitops/instance/llm-d/intelligent-inference/{{model}}/overlays/{{overlay}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{namespace}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

## Shell Script Automation

### Deployment Script

```bash
#!/bin/bash
set -e

NAMESPACE=${1:-demo-llm}
MODEL=${2:-gpt-oss-20b}
OVERLAY=${3:-modelcar}

echo "=== LLM-D Deployment Script ==="
echo "Namespace: $NAMESPACE"
echo "Model: $MODEL"
echo "Overlay: $OVERLAY"

# Check prerequisites
echo "Checking prerequisites..."
oc get gatewayclass openshift-default > /dev/null || {
  echo "ERROR: GatewayClass not found. Run prerequisites first."
  exit 1
}

# Create namespace if needed
oc get ns $NAMESPACE > /dev/null 2>&1 || {
  echo "Creating namespace $NAMESPACE..."
  oc create ns $NAMESPACE
}

# Deploy model
echo "Deploying $MODEL..."
oc apply -k gitops/instance/llm-d/intelligent-inference/$MODEL/overlays/$OVERLAY

# Wait for deployment
echo "Waiting for LLMInferenceService to be ready..."
oc wait --for=condition=Ready llminferenceservice/$MODEL -n $NAMESPACE --timeout=600s

# Get endpoint
INFERENCE_URL=$(oc -n openshift-ingress get gateway openshift-ai-inference \
  -o jsonpath='{.status.addresses[0].value}')
echo "=== Deployment Complete ==="
echo "Endpoint: http://$INFERENCE_URL/$NAMESPACE/$MODEL/v1"
```

### Cleanup Script

```bash
#!/bin/bash
set -e

NAMESPACE=${1:-demo-llm}

echo "=== Cleaning up LLM-D deployment in $NAMESPACE ==="

# Delete LLMInferenceServices
echo "Deleting LLMInferenceServices..."
oc delete llminferenceservice --all -n $NAMESPACE

# Delete namespace
echo "Deleting namespace..."
oc delete ns $NAMESPACE

echo "=== Cleanup Complete ==="
```

## Ansible Integration

### Playbook Structure

```yaml
# deploy-llm-d.yaml
---
- name: Deploy LLM-D on OpenShift
  hosts: localhost
  connection: local
  vars:
    namespace: demo-llm
    model: gpt-oss-20b
    overlay: modelcar
  tasks:
    - name: Check OpenShift connection
      kubernetes.core.k8s_info:
        api_version: v1
        kind: Namespace
        name: default
      register: cluster_check

    - name: Create namespace
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: v1
          kind: Namespace
          metadata:
            name: "{{ namespace }}"

    - name: Deploy Gateway configuration
      kubernetes.core.k8s:
        state: present
        src: gitops/instance/llm-d/gateway/gateway.yaml

    - name: Deploy LLMInferenceService
      kubernetes.core.k8s:
        state: present
        src: "gitops/instance/llm-d/intelligent-inference/{{ model }}/overlays/{{ overlay }}/"

    - name: Wait for LLMInferenceService to be ready
      kubernetes.core.k8s_info:
        api_version: serving.kserve.io/v1alpha1
        kind: LLMInferenceService
        name: "{{ model }}"
        namespace: "{{ namespace }}"
      register: llmisvc
      until: llmisvc.resources[0].status.conditions | selectattr('type', 'equalto', 'Ready') | selectattr('status', 'equalto', 'True') | list | length > 0
      retries: 60
      delay: 10
```

### Inventory Example

```yaml
# inventory.yaml
all:
  vars:
    ocp_clusters:
      - name: production
        api_url: https://api.prod.example.com:6443
        namespace: llm-production
      - name: staging
        api_url: https://api.staging.example.com:6443
        namespace: llm-staging
```

## Helm Chart (Custom)

While LLM-D doesn't provide an official Helm chart, you can create one:

### Chart Structure

```
llm-d-chart/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── namespace.yaml
    ├── llminferenceservice.yaml
    └── _helpers.tpl
```

### values.yaml

```yaml
namespace: demo-llm

model:
  name: gpt-oss-20b
  displayName: GPT OSS 20B
  uri: oci://quay.io/redhat-ai-services/modelcar-catalog:gpt-oss-20b
  modelName: openai/gpt-oss-20b

replicas: 2

resources:
  limits:
    cpu: '4'
    memory: 16Gi
    gpu: '1'
  requests:
    cpu: '1'
    memory: 8Gi
    gpu: '1'

vllm:
  additionalArgs: "--disable-uvicorn-access-log"
```

### templates/llminferenceservice.yaml

```yaml
apiVersion: serving.kserve.io/v1alpha1
kind: LLMInferenceService
metadata:
  name: {{ .Values.model.name }}
  namespace: {{ .Values.namespace }}
  annotations:
    openshift.io/display-name: {{ .Values.model.displayName }}
spec:
  replicas: {{ .Values.replicas }}
  model:
    uri: {{ .Values.model.uri }}
    name: {{ .Values.model.modelName }}
  router:
    gateway: {}
    route: {}
    scheduler: {}
  template:
    containers:
      - name: main
        env:
          - name: VLLM_ADDITIONAL_ARGS
            value: {{ .Values.vllm.additionalArgs | quote }}
        resources:
          limits:
            cpu: {{ .Values.resources.limits.cpu | quote }}
            memory: {{ .Values.resources.limits.memory }}
            nvidia.com/gpu: {{ .Values.resources.limits.gpu | quote }}
          requests:
            cpu: {{ .Values.resources.requests.cpu | quote }}
            memory: {{ .Values.resources.requests.memory }}
            nvidia.com/gpu: {{ .Values.resources.requests.gpu | quote }}
    tolerations:
      - effect: NoSchedule
        key: nvidia.com/gpu
        operator: Exists
```

## CI/CD Pipeline Example

### GitHub Actions Workflow

```yaml
name: Deploy LLM-D
on:
  push:
    branches: [main]
    paths:
      - 'models/**'

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Install oc CLI
        uses: redhat-actions/oc-installer@v1

      - name: Login to OpenShift
        uses: redhat-actions/oc-login@v1
        with:
          openshift_server_url: ${{ secrets.OPENSHIFT_SERVER }}
          openshift_token: ${{ secrets.OPENSHIFT_TOKEN }}

      - name: Deploy LLM-D
        run: |
          oc apply -k gitops/instance/llm-d/intelligent-inference/gpt-oss-20b/overlays/modelcar

      - name: Wait for deployment
        run: |
          oc wait --for=condition=Ready llminferenceservice/gpt-oss-20b \
            -n demo-llm --timeout=600s
```

## Next Steps

- [Disconnected Installs](05-disconnected-installs.md) for air-gapped automation
- [Running Benchmarks](06-running-benchmarks.md) for automated testing
