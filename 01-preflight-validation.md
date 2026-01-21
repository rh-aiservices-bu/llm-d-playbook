# Pre-flight Validation

This guide helps you verify that your OpenShift cluster meets all prerequisites for LLM-D deployment.

## GPU Operator Prerequisites

Before proceeding, ensure the following operators are installed on your cluster:

- **Node Feature Discovery (NFD)** - Labels nodes with hardware capabilities
- **NVIDIA GPU Operator** - Manages GPU drivers and device plugins

These operators are typically installed via OperatorHub and must be functioning before LLM-D deployment. The GPU Availability checks below will fail if these are not installed.

## Cluster Requirements

### OpenShift Version

```bash
# Check OpenShift version
oc version

# Required: 4.19+ (4.18 experimental only)
```

> **Note**: Gateway API dependencies are included in OpenShift 4.19+. For 4.18, additional configuration is required.

### Cluster Admin Access

```bash
# Verify cluster-admin role
oc auth can-i '*' '*' --all-namespaces
# Expected: yes
```

### GPU Availability

```bash
# Check for GPU nodes
oc get nodes -l nvidia.com/gpu.present=true

# Verify GPU resources
oc describe node <gpu-node> | grep -A5 "Allocatable:"

# Check NVIDIA GPU Operator status
oc get pods -n nvidia-gpu-operator

# Expected output (all pods should be Running or Completed):
# NAME                                                  READY   STATUS      AGE
# gpu-feature-discovery-xxxxx                           1/1     Running     ...
# gpu-operator-xxxxxxx-xxxxx                            1/1     Running     ...
# nvidia-container-toolkit-daemonset-xxxxx              1/1     Running     ...
# nvidia-cuda-validator-xxxxx                           0/1     Completed   ...
# nvidia-dcgm-exporter-xxxxx                            1/1     Running     ...
# nvidia-device-plugin-daemonset-xxxxx                  1/1     Running     ...
# nvidia-driver-daemonset-xxxxx                         1/1     Running     ...
# nvidia-operator-validator-xxxxx                       1/1     Running     ...
```



## Install Required Operators

If operators are missing, install them from the included `gitops/operators/` directory.

### Option 1 Quick Install (OCP 4.19+)

For a quick install of all prerequisites on OCP 4.19 or later:

```bash
# Install all OCP 4.19+ prerequisites at once
# This installs: Cert Manager, Service Mesh 3, Connectivity Link, RHOAI
# The retry loop handles dependency ordering automatically
until oc apply -k gitops/ocp-4.19; do : ; done

# Watch operator installation progress
oc get csv -A -w
```

> **Note**: This also works for OCP 4.20. The retry loop ensures operators install in the correct order as CRDs become available.

### Option 2 Individual operator installation

Install operators in this order to satisfy dependencies:

```bash

# 1. Cert Manager
oc apply -k gitops/operators/cert-manager
oc wait --for=condition=ready pod -l app.kubernetes.io/name=cert-manager -n cert-manager --timeout=300s

# 2. MetalLB (bare metal only - skip for cloud)
oc apply -k gitops/operators/metallb-operator
oc wait --for=condition=ready pod -l control-plane=controller-manager -n metallb-system --timeout=300s

# 3. Service Mesh 3
oc apply -k gitops/operators/servicemeshoperator3/operator/overlays/stable
# Wait for operator to install (check CSV status)
oc get csv -n openshift-operators -w

# 4. Connectivity Link (required for RHOAI 3.0+)
oc apply -k gitops/operators/connectivity-link
# Note: InstallPlan may require manual approval due to dependencies
# Check and approve if needed:
oc get installplan -n openshift-operators | grep -i "requiresapproval"
# If an InstallPlan is pending, approve it:
# oc patch installplan <name> -n openshift-operators --type merge -p '{"spec":{"approved":true}}'
# Wait for operators to install
oc get csv -n openshift-operators -w | grep -E "rhcl|authorino|limitador"
# Wait for AuthPolicy CRD to be available
oc wait --for=condition=Established crd/authpolicies.kuadrant.io --timeout=300s

# 5. Red Hat OpenShift AI
oc apply -k gitops/operators/rhoai
oc get csv -n redhat-ods-operator -w

# 6. Configure OpenShift AI (DSCInitialization and DataScienceCluster)
oc apply -k gitops/instance/rhoai
# Wait for LLMInferenceService CRD to be created
oc wait --for=condition=Established crd/llminferenceservices.serving.kserve.io --timeout=300s
# Wait for controller pods to be ready (required for webhook validation)
oc wait --for=condition=ready pod -l control-plane=odh-model-controller -n redhat-ods-applications --timeout=300s
oc wait --for=condition=ready pod -l control-plane=kserve-controller-manager -n redhat-ods-applications --timeout=300s

# 7. Node Feature Discovery (if not already installed)
# NFD is typically installed via OperatorHub

# 8. NVIDIA GPU Operator (if not already installed)
# GPU Operator is typically installed via OperatorHub

# 9. Leader Worker Set (optional - only for MoE models)
oc apply -k gitops/operators/leader-worker-set
```

## Required Operators Checklist

Run the included script to check all required operators:

```bash
./scripts/check-operators.sh
```

Or run manually:

```bash
oc get csv -A | grep -E "cert-manager|servicemesh|rhcl|rhods|nfd|gpu-operator"
```

## Bare Metal Validation (Optional)

Skip this section if deploying on cloud infrastructure (AWS, Azure, GCP, etc.).

### MetalLB Operator

Bare metal deployments require MetalLB for LoadBalancer service support:

```bash
# Check MetalLB operator
echo -n "MetalLB: "
oc get csv -n metallb-system 2>/dev/null | grep -q "metallb" && echo "OK" || echo "NOT FOUND"

# Check MetalLB pods
oc get pods -n metallb-system
```

### MetalLB Configuration

Verify MetalLB is configured with an IP address pool:

```bash
# Check IPAddressPool
oc get ipaddresspools -n metallb-system

# Check L2Advertisement
oc get l2advertisements -n metallb-system
```

If MetalLB is not configured, see [Advanced Deployment](03-advanced-deployment.md) for setup instructions.

## Gateway API Validation

### Check GatewayClass

```bash
oc get gatewayclass

# Expected output should include:
# NAME                CONTROLLER                          ACCEPTED
# openshift-default   openshift.io/gateway-controller/v1  True
```

### Check Gateway

```bash
oc get gateway -n openshift-ingress

# For LLM-D, you need:
# NAME                     CLASS              PROGRAMMED
# openshift-ai-inference   openshift-default  True
```

### Validate LoadBalancer Service 

```bash
# Check if Gateway has external IP assigned
oc get svc -n openshift-ingress | grep openshift-ai-inference

# Cloud: Should show EXTERNAL-IP
# Bare metal: Requires MetalLB configuration
```

## OpenShift AI Configuration Validation

### DSCInitialization Check

```bash
oc get dscinitializations -o yaml | grep -A5 "serviceMesh:"

# For LLM-D, verify:
# serviceMesh:
#   managementState: Removed
```

### DataScienceCluster Check

```bash
oc get datasciencecluster -o yaml | grep -A15 "kserve:"

# Verify:
# kserve:
#   defaultDeploymentMode: RawDeployment
#   managementState: Managed
#   rawDeploymentServiceConfig: Headed
#   serving:
#     managementState: Removed
```

> **Important**: The `rawDeploymentServiceConfig` setting defaults to `Headless` but **must be set to `Headed`** for proper load balancing. The `Headless` option is only intended for WatsonX integrations and will not properly load balance inference traffic. In RHOAI 3.x, this option is not shown in the default DSC and must be manually specified.

### Verify KServe Configuration

```bash
# Check KServe is configured for RawDeployment
oc get inferenceservices -A

# Check LLMInferenceServiceConfigs exist
oc get llminferenceserviceconfigs -n redhat-ods-applications
```

## Network Validation

### Test Internal Service Resolution

```bash
# Create a test pod to verify DNS resolution
oc run test-dns --rm -it --restart=Never --image=registry.access.redhat.com/ubi9/ubi -- \
  getent hosts openshift-ai-inference-openshift-default.openshift-ingress.svc.cluster.local

# Expected output: <IP>  openshift-ai-inference-openshift-default.openshift-ingress.svc.cluster.local
```

### Validate Gateway Connectivity

```bash
# Get Gateway address
GATEWAY_URL=$(oc -n openshift-ingress get gateway openshift-ai-inference \
  -o jsonpath='{.status.addresses[0].value}')

# Test connectivity (should return 404 if no services deployed)
curl -s -o /dev/null -w "%{http_code}" http://${GATEWAY_URL}/
```

## Storage Validation

### Check Default StorageClass

```bash
oc get storageclass

# Verify a default StorageClass exists (marked with "(default)")
```

### Test PVC Creation (Optional)

```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF

# Check if PVC is bound
oc get pvc test-pvc -n default

# Cleanup
oc delete pvc test-pvc -n default
```

## GPU Validation

### Verify GPU Scheduling

```bash
# Check GPU allocatable resources
oc get nodes -o json | jq '.items[] | select(.status.allocatable["nvidia.com/gpu"] != null) | {name: .metadata.name, gpus: .status.allocatable["nvidia.com/gpu"]}'
```

### Test GPU Pod Scheduling

```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: gpu-test
  namespace: default
spec:
  restartPolicy: Never
  containers:
  - name: cuda-test
    image: nvcr.io/nvidia/cuda:12.0.0-base-ubi8
    command: ["nvidia-smi"]
    resources:
      limits:
        nvidia.com/gpu: 1
  tolerations:
  - key: nvidia.com/gpu
    operator: Exists
    effect: NoSchedule
EOF

# Check pod logs
oc logs gpu-test -n default

# Cleanup
oc delete pod gpu-test -n default
```

## Pre-flight Summary Script

Run the comprehensive validation script:

```bash
./scripts/preflight-validation.sh
```

This checks:
- OpenShift version (4.19+)
- Cluster admin access
- GPU node availability
- All required operators (Cert Manager, Service Mesh 3, Connectivity Link, RHOAI, GPU Operator)
- Gateway configuration
- KServe RawDeployment mode

## Common Pre-flight Issues

### Gateway Not Getting External IP

**Symptom**: Gateway shows `<pending>` for external IP

**Resolution**:
- **Cloud**: Check cloud provider LoadBalancer quota and permissions
- **Bare metal**: Install and configure MetalLB (see [Advanced Deployment](03-advanced-deployment.md))

### Service Mesh Conflicts

**Symptom**: Errors about Service Mesh version conflicts

**Resolution**:
```bash
# Service Mesh 3 is installed with Manual approval
# Check pending InstallPlans
oc get installplan -n openshift-operators

# Approve if needed
oc patch installplan <name> -n openshift-operators --type merge -p '{"spec":{"approved":true}}'
```

### GPU Not Detected

**Symptom**: Nodes don't show GPU resources

**Resolution**:
1. Verify NFD is labeling nodes correctly
2. Check NVIDIA GPU Operator pods are running
3. Verify GPU driver compatibility with OpenShift version

### Connectivity Link Installation Issues

**Symptom**: `rhcl-operator` CSV stuck in Pending or InstallPlan not progressing

**Resolution**:
```bash
# Check for pending InstallPlans
oc get installplan -n openshift-operators

# Approve pending InstallPlan
oc patch installplan <name> -n openshift-operators --type merge -p '{"spec":{"approved":true}}'

# Verify all dependency CSVs are Succeeded
oc get csv -n openshift-operators | grep -E "rhcl|authorino|limitador|dns"
# Expected: All should show "Succeeded"
```

**Symptom**: "AuthPolicy CRD is not available" error in LLMInferenceService

**Resolution**:
1. Verify Connectivity Link is fully installed:
   ```bash
   oc get csv -n openshift-operators | grep rhcl-operator
   # Should show "Succeeded"
   ```

2. Verify AuthPolicy CRD exists with correct version:
   ```bash
   oc get crd authpolicies.kuadrant.io
   oc api-resources --api-group=kuadrant.io | grep authpolic
   # Should show kuadrant.io/v1 (not v1beta2)
   ```

3. Restart the kserve controller to pick up the new CRD:
   ```bash
   oc delete pod -n redhat-ods-applications -l control-plane=kserve-controller-manager
   oc wait --for=condition=ready pod -l control-plane=kserve-controller-manager \
     -n redhat-ods-applications --timeout=120s
   ```

> **Important**: Use the official `rhcl-operator` from `redhat-operators` catalog.
> Do NOT use the deprecated community `kuadrant-operator` from `community-operators` -
> it provides incompatible CRD versions (v1beta2 vs v1) that RHOAI 3.0 cannot use.

## Next Steps

Once all pre-flight checks pass, proceed to:
- [Quick Start](02-quick-start.md) for connected deployments
- [Disconnected Installs](05-disconnected-installs.md) for air-gapped environments
