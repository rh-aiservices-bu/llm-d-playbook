# Pre-flight Validation

This guide helps you verify that your OpenShift cluster meets all prerequisites for LLM-D deployment.

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
```

## Required Operators Checklist

Run this script to check all required operators:

```bash
#!/bin/bash

echo "=== Checking Required Operators ==="

# Check Cert Manager
echo -n "Cert Manager: "
oc get csv -A | grep -q "cert-manager" && echo "OK" || echo "MISSING"

# Check Service Mesh 3
echo -n "Service Mesh 3: "
oc get csv -n openshift-operators | grep -q "servicemesh" && echo "OK" || echo "MISSING"

# Check Connectivity Link (RHOAI 3.0+)
echo -n "Connectivity Link: "
oc get csv -A | grep -q "connectivity-link\|kuadrant" && echo "OK" || echo "NOT FOUND (optional for 2.25)"

# Check OpenShift AI
echo -n "OpenShift AI: "
oc get csv -n redhat-ods-operator | grep -q "rhods\|openshift-ai" && echo "OK" || echo "MISSING"

# Check MetalLB (bare metal only)
echo -n "MetalLB (bare metal): "
oc get csv -n metallb-system 2>/dev/null | grep -q "metallb" && echo "OK" || echo "NOT FOUND (cloud environments skip)"

# Check NFD
echo -n "Node Feature Discovery: "
oc get csv -A | grep -q "nfd" && echo "OK" || echo "MISSING"

# Check NVIDIA GPU Operator
echo -n "NVIDIA GPU Operator: "
oc get csv -n nvidia-gpu-operator | grep -q "gpu-operator" && echo "OK" || echo "MISSING"
```

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
oc get datasciencecluster -o yaml | grep -A10 "kserve:"

# Verify:
# kserve:
#   defaultDeploymentMode: RawDeployment
#   managementState: Managed
#   serving:
#     managementState: Removed
```

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
# Create a test pod
oc run test-dns --rm -it --restart=Never --image=registry.access.redhat.com/ubi9/ubi-minimal -- \
  nslookup openshift-ai-inference-openshift-default.openshift-ingress.svc.cluster.local
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

Run this comprehensive validation script:

```bash
#!/bin/bash

echo "=========================================="
echo "LLM-D Pre-flight Validation"
echo "=========================================="

PASS=0
FAIL=0

check() {
  if eval "$2" > /dev/null 2>&1; then
    echo "[PASS] $1"
    ((PASS++))
  else
    echo "[FAIL] $1"
    ((FAIL++))
  fi
}

# Core checks
check "OpenShift 4.19+" "oc version | grep -E 'Server Version: 4\.(19|[2-9][0-9])'"
check "Cluster admin access" "oc auth can-i '*' '*' --all-namespaces"
check "GPU nodes available" "oc get nodes -l nvidia.com/gpu.present=true | grep -v NAME"

# Operator checks
check "Cert Manager installed" "oc get csv -A | grep -i cert-manager | grep -i succeeded"
check "Service Mesh 3 installed" "oc get csv -n openshift-operators | grep -i servicemesh | grep -i succeeded"
check "OpenShift AI installed" "oc get csv -A | grep -E 'rhods|openshift-ai' | grep -i succeeded"
check "NVIDIA GPU Operator installed" "oc get csv -n nvidia-gpu-operator | grep gpu-operator | grep -i succeeded"

# Gateway checks
check "GatewayClass exists" "oc get gatewayclass openshift-default"
check "AI Inference Gateway exists" "oc get gateway openshift-ai-inference -n openshift-ingress"

# Configuration checks
check "KServe in RawDeployment mode" "oc get datasciencecluster -o yaml | grep -A5 kserve | grep RawDeployment"

echo ""
echo "=========================================="
echo "Results: $PASS passed, $FAIL failed"
echo "=========================================="

if [ $FAIL -gt 0 ]; then
  echo "Please resolve failed checks before proceeding."
  exit 1
fi
```

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

## Next Steps

Once all pre-flight checks pass, proceed to:
- [Quick Start](02-quick-start.md) for connected deployments
- [Disconnected Installs](05-disconnected-installs.md) for air-gapped environments
