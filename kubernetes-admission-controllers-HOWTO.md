# Kubernetes Admission Controllers - Student Study Guide

> **A practical guide for learning Kubernetes Admission Controllers with hands-on examples**

---

## Table of Contents
1. [What are Admission Controllers?](#what-are-admission-controllers)
2. [How Admission Controllers Work](#how-admission-controllers-work)
3. [Types of Admission Controllers](#types-of-admission-controllers)
4. [Common Built-in Admission Controllers](#common-built-in-admission-controllers)
5. [Dynamic Admission Controllers (Webhooks)](#dynamic-admission-controllers-webhooks)
6. [Viewing Admission Controllers in Managed Kubernetes](#viewing-admission-controllers-in-managed-kubernetes)
7. [Hands-On Examples](#hands-on-examples)
8. [Best Practices](#best-practices)
9. [Quick Reference](#quick-reference)

---

## What are Admission Controllers?

Admission controllers intercept requests to the Kubernetes API server after authentication and authorization but before the resource is persisted. Think of them as **gatekeepers** that can:

- ✅ **Validate** requests (reject invalid configurations)
- ✏️ **Mutate** requests (modify or add defaults)
- 🛡️ **Enforce policies** (security, resource limits, naming conventions)

### Why Do We Need Them?

Without admission controllers, you might face issues like:
- Pods running as root (security risk)
- Missing resource limits (resource exhaustion)
- Invalid configurations (runtime failures)
- Namespace misconfigurations

---

## How Admission Controllers Work

### Request Flow Diagram

```
┌─────────────┐
│   kubectl   │
│   create    │
└──────┬──────┘
       │
       ▼
┌─────────────────────────────────────┐
│      Authentication Layer           │
│  (Who are you?)                     │
└──────────────┬──────────────────────┘
               ▼
┌─────────────────────────────────────┐
│      Authorization Layer            │
│  (Are you allowed?)                 │
└──────────────┬──────────────────────┘
               ▼
┌─────────────────────────────────────┐
│  MUTATING Admission Controllers     │
│  (Modify the request)               │
│  - MutatingWebhook                  │
│  - DefaultStorageClass              │
└──────────────┬──────────────────────┘
               ▼
┌─────────────────────────────────────┐
│      Schema Validation              │
│  (Is the object valid?)             │
└──────────────┬──────────────────────┘
               ▼
┌─────────────────────────────────────┐
│  VALIDATING Admission Controllers   │
│  (Accept or reject)                 │
│  - ValidatingWebhook                │
│  - ResourceQuota                    │
└──────────────┬──────────────────────┘
               ▼
┌─────────────────────────────────────┐
│      Persist to etcd                │
└─────────────────────────────────────┘
```

### Key Points:
1. **Mutating** controllers run first (they can modify objects)
2. **Validating** controllers run after mutation (they accept/reject)
3. If any controller rejects → entire request fails
4. All changes happen **before** storing in etcd

---

## Types of Admission Controllers

### 1. Mutating Admission Controllers
These **modify** the incoming request.

**Example**: Add a sidecar container, set default values, inject labels

### 2. Validating Admission Controllers
These **validate** and accept/reject requests.

**Example**: Ensure resource limits exist, check naming conventions

### 3. Combined Controllers
Some do both! They mutate first, then validate.

---

## Common Built-in Admission Controllers

### Currently Enabled by Default (Kubernetes 1.35)

```bash
# View enabled admission controllers
kube-apiserver -h | grep enable-admission-plugins
```

**Default admission controllers:**
```
CertificateApproval
CertificateSigning
CertificateSubjectRestriction
DefaultIngressClass
DefaultStorageClass
DefaultTolerationSeconds
LimitRanger
MutatingAdmissionWebhook
NamespaceLifecycle
PersistentVolumeClaimResize
PodSecurity
Priority
ResourceQuota
RuntimeClass
ServiceAccount
StorageObjectInUseProtection
TaintNodesByCondition
ValidatingAdmissionPolicy
ValidatingAdmissionWebhook
```

### Important Admission Controllers Explained

#### 1. **NamespaceLifecycle** (Validating)

**What it does:**
- Prevents creating objects in non-existent namespaces
- Prevents deleting system namespaces: `default`, `kube-system`, `kube-public`
- Rejects requests to terminating namespaces

**Example:**
```yaml
# This will FAIL if namespace "test" doesn't exist
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: test  # ❌ Namespace must exist first!
spec:
  containers:
  - name: nginx
    image: nginx
```

**Fix:**
```bash
# Create namespace first
kubectl create namespace test

# Now create the pod
kubectl apply -f pod.yaml
```

---

#### 2. **LimitRanger** (Mutating + Validating)

**What it does:**
- Enforces resource limits defined in LimitRange
- Sets default resource requests/limits if not specified

**Example:**

**Step 1: Create a LimitRange**
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: resource-limits
  namespace: demo
spec:
  limits:
  - max:
      cpu: "2"
      memory: "2Gi"
    min:
      cpu: "100m"
      memory: "128Mi"
    default:  # Default limits
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:  # Default requests
      cpu: "200m"
      memory: "256Mi"
    type: Container
```

**Step 2: Create a Pod without limits**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: demo
spec:
  containers:
  - name: nginx
    image: nginx
    # No resources specified!
```

**What happens:**
```bash
kubectl get pod nginx -n demo -o yaml | grep -A 5 resources
```

**Output** (defaults automatically applied):
```yaml
resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 200m
    memory: 256Mi
```

---

#### 3. **ResourceQuota** (Validating)

**What it does:**
- Enforces total resource consumption limits per namespace

**Example:**

**Step 1: Create ResourceQuota**
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: demo
spec:
  hard:
    requests.cpu: "4"
    requests.memory: "8Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"
    pods: "10"
```

**Step 2: Try to create a pod that exceeds quota**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: big-pod
  namespace: demo
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:
        cpu: "5"      # ❌ Exceeds quota!
        memory: "10Gi"
```

**Result:**
```
Error: exceeded quota: compute-quota
```

---

#### 4. **PodSecurity** (Validating)

**What it does:**
- Enforces Pod Security Standards (replaces PodSecurityPolicy)

**Example:**

**Step 1: Label namespace with security level**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: restricted-ns
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

**Step 2: Try to create a privileged pod**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: privileged-pod
  namespace: restricted-ns
spec:
  containers:
  - name: nginx
    image: nginx
    securityContext:
      privileged: true  # ❌ Not allowed in restricted mode!
```

**Result:**
```
Error: pods "privileged-pod" is forbidden: violates PodSecurity "restricted:latest"
```

**Fix:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
  namespace: restricted-ns
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: nginx
    image: nginx
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop:
        - ALL
```

---

#### 5. **DefaultStorageClass** (Mutating)

**What it does:**
- Automatically assigns default StorageClass to PVCs without one specified

**Example:**

**Step 1: Create default StorageClass**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
```

**Step 2: Create PVC without storageClassName**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  # No storageClassName specified!
```

**Result** (default automatically added):
```yaml
spec:
  storageClassName: fast-ssd  # ← Automatically added!
```

---

## Dynamic Admission Controllers (Webhooks)

Admission webhooks are HTTP callbacks that receive admission requests and can implement custom validation or mutation logic.

### Types of Webhooks

1. **MutatingAdmissionWebhook** - Modifies objects
2. **ValidatingAdmissionWebhook** - Validates objects
3. **ValidatingAdmissionPolicy** - In-cluster validation using CEL

### When to Use Webhooks?

Use webhooks when built-in controllers can't handle your use case:
- Custom business logic validation
- Integration with external systems
- Organization-specific policies
- Complex multi-field validation

---

## Hands-On Examples

### Example 1: Enable/Disable Admission Controllers

**Check current admission controllers:**
```bash
ps aux | grep kube-apiserver | grep admission-plugins
```

**Enable additional controller:**
```bash
# Edit kube-apiserver manifest
sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml
```

Add to `--enable-admission-plugins`:
```yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota,NodeRestriction
```

**Disable a controller:**
```yaml
- --disable-admission-plugins=DefaultStorageClass
```

---

### Example 2: Create a Simple Validating Webhook

**Step 1: Create a webhook server (Go example)**

```go
package main

import (
    "encoding/json"
    "fmt"
    "io/ioutil"
    "net/http"
    
    admissionv1 "k8s.io/api/admission/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

func validatePod(w http.ResponseWriter, r *http.Request) {
    // Read request body
    body, _ := ioutil.ReadAll(r.Body)
    
    // Parse AdmissionReview
    var review admissionv1.AdmissionReview
    json.Unmarshal(body, &review)
    
    // Create response
    response := &admissionv1.AdmissionResponse{
        UID: review.Request.UID,
        Allowed: true,
    }
    
    // Validate: Reject pods without resource limits
    pod := review.Request.Object.Raw
    if !hasResourceLimits(pod) {
        response.Allowed = false
        response.Result = &metav1.Status{
            Message: "Pods must have resource limits",
        }
    }
    
    // Send response
    review.Response = response
    json.NewEncoder(w).Encode(review)
}

func main() {
    http.HandleFunc("/validate", validatePod)
    http.ListenAndServeTLS(":443", "tls.crt", "tls.key", nil)
}
```

**Step 2: Deploy the webhook**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pod-validator
  namespace: webhook-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pod-validator
  template:
    metadata:
      labels:
        app: pod-validator
    spec:
      containers:
      - name: webhook
        image: your-registry/pod-validator:v1
        ports:
        - containerPort: 443
---
apiVersion: v1
kind: Service
metadata:
  name: pod-validator
  namespace: webhook-system
spec:
  selector:
    app: pod-validator
  ports:
  - port: 443
    targetPort: 443
```

**Step 3: Register the webhook**

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: pod-validator
webhooks:
- name: pod-validator.example.com
  clientConfig:
    service:
      name: pod-validator
      namespace: webhook-system
      path: "/validate"
    caBundle: LS0tLS1CRUdJTi... # Base64 encoded CA cert
  rules:
  - operations: ["CREATE", "UPDATE"]
    apiGroups: [""]
    apiVersions: ["v1"]
    resources: ["pods"]
  admissionReviewVersions: ["v1"]
  sideEffects: None
  timeoutSeconds: 5
  failurePolicy: Fail  # Reject on webhook failure
```

**Test it:**
```bash
# This should be rejected
kubectl run nginx --image=nginx --restart=Never

# Output:
# Error from server: admission webhook "pod-validator.example.com" denied the request: 
# Pods must have resource limits
```

---

### Example 3: Simple Mutating Webhook (Add Labels)

**Webhook response to add labels:**

```json
{
  "apiVersion": "admission.k8s.io/v1",
  "kind": "AdmissionReview",
  "response": {
    "uid": "<request-uid>",
    "allowed": true,
    "patchType": "JSONPatch",
    "patch": "W3sib3AiOiAiYWRkIiwgInBhdGgiOiAiL21ldGFkYXRhL2xhYmVscyIsICJ2YWx1ZSI6IHsiZW52IjogInByb2QiLCAidGVhbSI6ICJkZXZvcHMifX1d"
  }
}
```

**Decoded patch:**
```json
[
  {
    "op": "add",
    "path": "/metadata/labels",
    "value": {
      "env": "prod",
      "team": "devops"
    }
  }
]
```

**MutatingWebhookConfiguration:**

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: label-injector
webhooks:
- name: label-injector.example.com
  clientConfig:
    service:
      name: label-injector
      namespace: webhook-system
      path: "/mutate"
    caBundle: LS0tLS1CRUdJTi...
  rules:
  - operations: ["CREATE"]
    apiGroups: [""]
    apiVersions: ["v1"]
    resources: ["pods"]
  admissionReviewVersions: ["v1"]
  sideEffects: None
```

---

### Example 4: Using ValidatingAdmissionPolicy (No Webhook Needed!)

**Create a policy to require labels:**

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: require-labels
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      operations: ["CREATE", "UPDATE"]
      resources: ["pods"]
  validations:
  - expression: "has(object.metadata.labels.env)"
    message: "Pod must have 'env' label"
  - expression: "has(object.metadata.labels.team)"
    message: "Pod must have 'team' label"
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: require-labels-binding
spec:
  policyName: require-labels
  validationActions: [Deny]
  matchResources:
    namespaceSelector:
      matchLabels:
        environment: production
```

**Test:**
```bash
# This will fail
kubectl run test --image=nginx -n production

# This will succeed
kubectl run test --image=nginx -n production --labels=env=prod,team=platform
```

---

## Best Practices

### 1. **Ordering Matters**
- Mutating webhooks run before validating webhooks
- Order within each type is sequential
- Design accordingly!

### 2. **Failure Policy**
```yaml
failurePolicy: Fail    # Reject if webhook unavailable (safe)
failurePolicy: Ignore  # Allow if webhook unavailable (risky)
```

**Recommendation:** Use `Fail` for critical validations, `Ignore` for non-critical mutations.

### 3. **Timeouts**
```yaml
timeoutSeconds: 10  # Default
timeoutSeconds: 30  # Max allowed
```

Keep webhooks fast! Slow webhooks block all requests.

### 4. **Scope Your Webhooks**
```yaml
rules:
- operations: ["CREATE"]  # Only on create
  apiGroups: ["apps"]     # Only apps group
  resources: ["deployments"]  # Only deployments
  scope: "Namespaced"     # Not cluster-scoped resources
```

Don't intercept everything!

### 5. **Use Object/Namespace Selectors**
```yaml
objectSelector:
  matchLabels:
    validate: "true"  # Only objects with this label

namespaceSelector:
  matchExpressions:
  - key: environment
    operator: In
    values: ["prod", "staging"]
```

### 6. **Webhook Security**
- Always use TLS
- Validate webhook certificates
- Use mutual TLS when possible
- Implement proper RBAC

### 7. **Testing**
```bash
# Test in dry-run mode first
kubectl apply -f pod.yaml --dry-run=server

# Check webhook logs
kubectl logs -n webhook-system deployment/my-webhook
```

---

## Quick Reference

### Common Commands

```bash
# List enabled admission plugins
kube-apiserver -h | grep enable-admission-plugins

# Check admission controller configuration
kubectl get validatingwebhookconfigurations
kubectl get mutatingwebhookconfigurations
kubectl get validatingadmissionpolicies

# Describe webhook
kubectl describe validatingwebhookconfiguration my-webhook

# Test webhook with dry-run
kubectl create -f pod.yaml --dry-run=server -v=8

# Check logs
kubectl logs -n kube-system kube-apiserver-<node>
```

### Troubleshooting

| Problem | Solution |
|---------|----------|
| Webhook timing out | Increase `timeoutSeconds`, optimize webhook code |
| Webhook rejecting valid requests | Check webhook logs, verify matching rules |
| Webhook not being called | Verify rules, namespaceSelector, objectSelector |
| Certificate errors | Update caBundle, check cert expiry |
| Pods stuck in pending | Check if webhook is failing with `failurePolicy: Fail` |

### Key Files Locations

```
/etc/kubernetes/manifests/kube-apiserver.yaml  # API server config
/var/log/kube-apiserver.log                    # API server logs
```

---

## Viewing Admission Controllers in Managed Kubernetes

### EKS (Amazon Elastic Kubernetes Service)

In managed Kubernetes clusters like EKS, you **cannot directly access** the kube-apiserver configuration because AWS manages the control plane. However, you can still discover which admission controllers are enabled.

#### Method 1: Check API Server Logs (Limited Access)

EKS doesn't expose kube-apiserver logs directly, but you can infer admission controller behavior through CloudWatch Logs if enabled.

**Enable Control Plane Logging:**
```bash
# Enable API server logs in EKS
aws eks update-cluster-config \
    --name my-cluster \
    --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}'
```

**View logs in CloudWatch:**
```bash
# List log streams
aws logs describe-log-streams \
    --log-group-name /aws/eks/my-cluster/cluster \
    --log-stream-name-prefix kube-apiserver

# Get logs
aws logs tail /aws/eks/my-cluster/cluster --follow
```

#### Method 2: Test Specific Admission Controllers

You can verify which admission controllers are active by testing their behavior:

**Test 1: NamespaceLifecycle**
```bash
# Try creating a pod in non-existent namespace
kubectl run test --image=nginx -n non-existent-namespace

# If NamespaceLifecycle is enabled, you'll see:
# Error: namespaces "non-existent-namespace" not found
```

**Test 2: ServiceAccount**
```bash
# Create a pod without specifying serviceAccount
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-sa
spec:
  containers:
  - name: nginx
    image: nginx
EOF

# Check if default SA was injected
kubectl get pod test-sa -o jsonpath='{.spec.serviceAccountName}'
# Output: default (if ServiceAccount admission controller is enabled)
```

**Test 3: MutatingAdmissionWebhook**
```bash
# Check if mutating webhooks are configured
kubectl get mutatingwebhookconfigurations
```

**Test 4: ValidatingAdmissionWebhook**
```bash
# Check if validating webhooks are configured
kubectl get validatingwebhookconfigurations
```

**Test 5: DefaultStorageClass**
```bash
# Create PVC without storageClassName
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF

# Check if default storage class was assigned
kubectl get pvc test-pvc -o jsonpath='{.spec.storageClassName}'
```

**Test 6: ResourceQuota**
```bash
# Create a namespace with quota
kubectl create namespace quota-test
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: test-quota
  namespace: quota-test
spec:
  hard:
    pods: "1"
EOF

# Try creating 2 pods
kubectl run pod1 --image=nginx -n quota-test
kubectl run pod2 --image=nginx -n quota-test

# Second pod should fail with quota exceeded error
```

#### Method 3: Use Kubernetes API to Query

```bash
# Check API server version and features
kubectl version --short

# Get cluster info
kubectl cluster-info

# Check API resources (some depend on admission controllers)
kubectl api-resources

# Describe nodes (some info added by admission controllers)
kubectl describe node | grep -i "admission"
```

#### Method 4: Create a Debug Pod to Inspect

```bash
# Create a pod with kubectl
kubectl run debug-pod --image=nicolaka/netshoot --rm -it -- /bin/bash

# Inside the pod, try to access API server metadata
curl -k https://kubernetes.default.svc/version
```

#### Method 5: Check EKS Documentation

AWS documents the admission controllers enabled by default in EKS:

**Default EKS Admission Controllers (as of EKS 1.28+):**
```
NamespaceLifecycle
LimitRanger
ServiceAccount
DefaultStorageClass
DefaultTolerationSeconds
MutatingAdmissionWebhook
ValidatingAdmissionWebhook
ResourceQuota
PodSecurity
Priority
TaintNodesByCondition
StorageObjectInUseProtection
PersistentVolumeClaimResize
RuntimeClass
CertificateApproval
CertificateSigning
CertificateSubjectRestriction
```

**EKS-Specific Note:**
- EKS automatically enables Pod Security Standards
- You cannot modify built-in admission controllers
- You can add custom webhooks (Mutating/Validating)

#### Practical Example: Verify EKS Admission Controllers

**Create a test script:**
```bash
#!/bin/bash
# test-eks-admission-controllers.sh

echo "Testing EKS Admission Controllers..."
echo "======================================"

# Test 1: NamespaceLifecycle
echo -e "\n1. Testing NamespaceLifecycle..."
kubectl run test-ns --image=nginx -n fake-namespace 2>&1 | grep -q "not found"
if [ $? -eq 0 ]; then
    echo "✅ NamespaceLifecycle: ENABLED"
else
    echo "❌ NamespaceLifecycle: DISABLED"
fi

# Test 2: ServiceAccount
echo -e "\n2. Testing ServiceAccount..."
kubectl run test-sa --image=nginx --dry-run=client -o yaml | kubectl apply -f - > /dev/null 2>&1
SA=$(kubectl get pod test-sa -o jsonpath='{.spec.serviceAccountName}' 2>/dev/null)
if [ "$SA" == "default" ]; then
    echo "✅ ServiceAccount: ENABLED (auto-injected: $SA)"
    kubectl delete pod test-sa > /dev/null 2>&1
else
    echo "❌ ServiceAccount: DISABLED"
fi

# Test 3: MutatingAdmissionWebhook
echo -e "\n3. Testing MutatingAdmissionWebhook..."
COUNT=$(kubectl get mutatingwebhookconfigurations 2>/dev/null | wc -l)
if [ $COUNT -gt 1 ]; then
    echo "✅ MutatingAdmissionWebhook: ENABLED"
    kubectl get mutatingwebhookconfigurations --no-headers | awk '{print "   - " $1}'
else
    echo "⚠️  MutatingAdmissionWebhook: ENABLED (no webhooks configured)"
fi

# Test 4: ValidatingAdmissionWebhook
echo -e "\n4. Testing ValidatingAdmissionWebhook..."
COUNT=$(kubectl get validatingwebhookconfigurations 2>/dev/null | wc -l)
if [ $COUNT -gt 1 ]; then
    echo "✅ ValidatingAdmissionWebhook: ENABLED"
    kubectl get validatingwebhookconfigurations --no-headers | awk '{print "   - " $1}'
else
    echo "⚠️  ValidatingAdmissionWebhook: ENABLED (no webhooks configured)"
fi

# Test 5: PodSecurity
echo -e "\n5. Testing PodSecurity..."
kubectl create namespace test-pod-security > /dev/null 2>&1
kubectl label namespace test-pod-security pod-security.kubernetes.io/enforce=restricted > /dev/null 2>&1
kubectl run privileged-test --image=nginx --privileged -n test-pod-security 2>&1 | grep -q "violates PodSecurity"
if [ $? -eq 0 ]; then
    echo "✅ PodSecurity: ENABLED"
    kubectl delete namespace test-pod-security > /dev/null 2>&1
else
    echo "❌ PodSecurity: DISABLED"
    kubectl delete namespace test-pod-security > /dev/null 2>&1
fi

# Test 6: DefaultStorageClass
echo -e "\n6. Testing DefaultStorageClass..."
DEFAULT_SC=$(kubectl get storageclass -o jsonpath='{.items[?(@.metadata.annotations.storageclass\.kubernetes\.io/is-default-class=="true")].metadata.name}')
if [ ! -z "$DEFAULT_SC" ]; then
    echo "✅ DefaultStorageClass: ENABLED (default: $DEFAULT_SC)"
else
    echo "⚠️  DefaultStorageClass: No default storage class configured"
fi

echo -e "\n======================================"
echo "Test completed!"
```

**Run the test:**
```bash
chmod +x test-eks-admission-controllers.sh
./test-eks-admission-controllers.sh
```

**Expected Output:**
```
Testing EKS Admission Controllers...
======================================

1. Testing NamespaceLifecycle...
✅ NamespaceLifecycle: ENABLED

2. Testing ServiceAccount...
✅ ServiceAccount: ENABLED (auto-injected: default)

3. Testing MutatingAdmissionWebhook...
✅ MutatingAdmissionWebhook: ENABLED
   - vpc-resource-mutating-webhook
   - pod-identity-mutating-webhook

4. Testing ValidatingAdmissionWebhook...
⚠️  ValidatingAdmissionWebhook: ENABLED (no webhooks configured)

5. Testing PodSecurity...
✅ PodSecurity: ENABLED

6. Testing DefaultStorageClass...
✅ DefaultStorageClass: ENABLED (default: gp2)

======================================
Test completed!
```

### GKE (Google Kubernetes Engine)

**View admission controllers in GKE:**

```bash
# GKE doesn't expose kube-apiserver config directly
# But you can check cluster configuration

# Get cluster details
gcloud container clusters describe my-cluster --zone us-central1-a

# Check enabled addons (some use admission controllers)
gcloud container clusters describe my-cluster --zone us-central1-a \
    --format="value(addonsConfig)"

# Test specific controllers (same methods as EKS)
kubectl get mutatingwebhookconfigurations
kubectl get validatingwebhookconfigurations
```

### AKS (Azure Kubernetes Service)

**View admission controllers in AKS:**

```bash
# Get cluster configuration
az aks show --resource-group myResourceGroup --name myAKSCluster

# Check API server access
az aks show --resource-group myResourceGroup --name myAKSCluster \
    --query "apiServerAccessProfile"

# Test admission controllers (same methods as EKS)
kubectl get mutatingwebhookconfigurations
kubectl get validatingwebhookconfigurations
```

### Summary Table: Managed K8s Admission Controllers Access

| Provider | Direct Access | Control Plane Logs | Test via API | Add Custom Webhooks |
|----------|---------------|-------------------|--------------|---------------------|
| **EKS**  | ❌ No         | ✅ CloudWatch     | ✅ Yes       | ✅ Yes              |
| **GKE**  | ❌ No         | ✅ Cloud Logging  | ✅ Yes       | ✅ Yes              |
| **AKS**  | ❌ No         | ✅ Azure Monitor  | ✅ Yes       | ✅ Yes              |
| **Self-Hosted** | ✅ Yes | ✅ Direct        | ✅ Yes       | ✅ Yes              |

### Key Takeaways for Managed Kubernetes

1. 🔒 **No direct control plane access** - You cannot modify built-in admission controllers
2. 🧪 **Test behavior** - Best way to verify which controllers are enabled
3. 📝 **Check documentation** - Each provider documents their defaults
4. ✅ **Custom webhooks work** - You can add MutatingWebhook and ValidatingWebhook
5. 📊 **Enable logging** - CloudWatch/Cloud Logging helps troubleshoot admission issues

---

## Practice Exercises

### Exercise 1: Basic Validation
Create a ValidatingWebhook that rejects pods without the label `owner`.

### Exercise 2: Resource Management
Create a LimitRange that sets default CPU limit to 500m and memory to 512Mi.

### Exercise 3: Security Hardening
Configure PodSecurity admission controller in `restricted` mode for a namespace.

### Exercise 4: Custom Mutation
Create a MutatingWebhook that automatically adds a sidecar logging container.

### Exercise 5: Namespace Protection
Prevent deletion of namespaces with label `protected: "true"`.

### Exercise 6: EKS Admission Controller Testing (NEW!)
Run the provided test script on your EKS cluster and document which admission controllers are enabled.

---

## Additional Resources

- 📚 [Official Kubernetes Admission Controllers Reference](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)
- 🔗 [Dynamic Admission Control](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/)
- 🛠️ [Webhook Development Guide](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#write-an-admission-webhook-server)
- 🎯 [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)

---

## Summary

**Key Takeaways:**

1. ✅ Admission controllers are gatekeepers between API requests and etcd
2. 🔄 Mutating controllers run first, validating controllers run second
3. 🎯 Use built-in controllers when possible (LimitRanger, ResourceQuota, PodSecurity)
4. 🌐 Use webhooks for custom business logic
5. ⚡ ValidatingAdmissionPolicy is easier than webhooks (no server needed!)
6. 🛡️ Always set proper `failurePolicy` and `timeoutSeconds`
7. 📊 Scope webhooks carefully to avoid performance issues
8. 🔒 In managed K8s (EKS/GKE/AKS), test admission controllers via behavior rather than direct inspection
9. 📝 Enable control plane logging in managed clusters for troubleshooting

**Remember:** Admission controllers are critical for cluster security, stability, and compliance. Use them wisely!

---

*Happy Learning! 🚀*

*Last Updated: January 2026*
