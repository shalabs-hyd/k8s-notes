# Kubernetes Mutating Admission Controllers: A Comprehensive Guide

## 📚 Table of Contents
- [Introduction](#introduction)
- [What Are Admission Controllers?](#what-are-admission-controllers)
- [Request Flow](#request-flow)
- [Mutating vs Validating](#mutating-vs-validating)
- [Detailed Controller Documentation](#detailed-controller-documentation)
- [Best Practices](#best-practices)
- [Troubleshooting](#troubleshooting)

---

## Introduction

Admission controllers are powerful plugins in the Kubernetes API server that intercept requests **before** objects are persisted to etcd. They can modify (mutate) or validate requests, ensuring cluster security, governance, and resource management.

This document covers **13 mutating admission controllers** active in your cluster.

---

## What Are Admission Controllers?

Admission controllers act as **gatekeepers** for the Kubernetes API server:

- ✅ **Validate** requests (reject invalid configurations)
- ✏️ **Mutate** requests (modify objects before storage)
- 🛡️ **Enforce** policies (security, quotas, best practices)

```
┌─────────────┐
│   kubectl   │
│   or API    │
│   Client    │
└──────┬──────┘
       │
       ▼
┌─────────────────────────┐
│   Authentication        │
│   (Who are you?)        │
└──────────┬──────────────┘
           │
           ▼
┌─────────────────────────┐
│   Authorization         │
│   (What can you do?)    │
└──────────┬──────────────┘
           │
           ▼
┌─────────────────────────┐
│  MUTATING ADMISSION     │  ◄── Modify the request
│  CONTROLLERS            │
└──────────┬──────────────┘
           │
           ▼
┌─────────────────────────┐
│  VALIDATING ADMISSION   │  ◄── Validate the request
│  CONTROLLERS            │
└──────────┬──────────────┘
           │
           ▼
┌─────────────────────────┐
│   Persist to etcd       │
└─────────────────────────┘
```

---

## Request Flow

### The Admission Controller Pipeline

```
User Request
     │
     ├──► [1] NamespaceAutoProvision
     │         ↓ Auto-create namespace if missing
     │
     ├──► [2] NamespaceLifecycle
     │         ↓ Block operations on terminating namespaces
     │
     ├──► [3] LimitRanger
     │         ↓ Apply default resource limits
     │
     ├──► [4] ServiceAccount
     │         ↓ Auto-assign ServiceAccount
     │
     ├──► [5] NodeRestriction
     │         ↓ Restrict kubelet modifications
     │
     ├──► [6] TaintNodesByCondition
     │         ↓ Add taints based on node conditions
     │
     ├──► [7] Priority
     │         ↓ Apply PriorityClass
     │
     ├──► [8] DefaultTolerationSeconds
     │         ↓ Add default tolerations
     │
     ├──► [9] DefaultStorageClass
     │         ↓ Apply default StorageClass
     │
     ├──► [10] StorageObjectInUseProtection
     │         ↓ Add finalizers to prevent deletion
     │
     ├──► [11] RuntimeClass
     │         ↓ Validate and apply RuntimeClass
     │
     ├──► [12] DefaultIngressClass
     │         ↓ Apply default IngressClass
     │
     └──► [13] MutatingAdmissionWebhook
               ↓ Custom mutation logic via webhooks
     
Object persisted to etcd ✓
```

---

## Mutating vs Validating

| Aspect | Mutating Controllers | Validating Controllers |
|--------|---------------------|------------------------|
| **Purpose** | Modify/enhance objects | Accept or reject objects |
| **Execution Order** | First | After mutating |
| **Can Change Object** | ✅ Yes | ❌ No |
| **Example** | Add default values | Enforce naming conventions |

---

## Detailed Controller Documentation

### 1️⃣ NamespaceAutoProvision

**Purpose:** Automatically creates namespaces if they don't exist.

**Status:** ⚠️ Deprecated (since v1.10)

**How It Works:**
- Intercepts requests for non-existent namespaces
- Automatically creates the namespace
- Allows the original request to proceed

**Example:**
```yaml
# You create a pod in namespace "dev"
apiVersion: v1
kind: Pod
metadata:
  name: app
  namespace: dev  # Namespace doesn't exist yet
  
# Controller auto-creates "dev" namespace
# Pod creation succeeds
```

**Note:** Modern clusters should use explicit namespace creation or tools like Helm.

---

### 2️⃣ NamespaceLifecycle

**Purpose:** Prevents operations in terminating namespaces and enforces namespace existence.

**Key Functions:**
- ✋ Blocks object creation in namespaces being deleted
- 🚫 Prevents deletion of `default`, `kube-system`, `kube-public` namespaces
- ✅ Ensures namespace exists before creating objects

**Lifecycle States:**
```
Active  ──delete──►  Terminating  ──cleanup──►  Deleted
  ✅                      ❌                      ×
Create OK           Create BLOCKED          Doesn't exist
```

**Example:**
```bash
# Try to create pod in terminating namespace
kubectl create -f pod.yaml -n my-namespace

# Error: namespace "my-namespace" is terminating
```

**Why It Matters:** Prevents resource leaks and ensures clean namespace deletion.

---

### 3️⃣ LimitRanger

**Purpose:** Applies default resource limits and validates resource requests/limits.

**What It Does:**
- 📏 Applies default CPU/memory requests and limits
- ✅ Validates resource constraints
- 🎯 Enforces min/max resource boundaries per namespace

**LimitRange Configuration:**
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: mem-cpu-limits
  namespace: production
spec:
  limits:
  - default:           # Default limits
      memory: 512Mi
      cpu: 500m
    defaultRequest:    # Default requests
      memory: 256Mi
      cpu: 250m
    max:              # Maximum allowed
      memory: 1Gi
      cpu: 1000m
    min:              # Minimum allowed
      memory: 128Mi
      cpu: 100m
    type: Container
```

**Before & After:**
```yaml
# User submits (no resources specified)
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: nginx

# LimitRanger mutates to:
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        memory: "256Mi"
        cpu: "250m"
      limits:
        memory: "512Mi"
        cpu: "500m"
```

---

### 4️⃣ ServiceAccount

**Purpose:** Automatically assigns ServiceAccounts to Pods and injects authentication tokens.

**Automatic Behavior:**
- 🤖 Assigns `default` ServiceAccount if none specified
- 🔑 Mounts ServiceAccount token as a volume
- 📂 Injects token at `/var/run/secrets/kubernetes.io/serviceaccount/`

**Process Flow:**
```
Pod without SA  ──►  ServiceAccount Controller  ──►  Pod with SA + Token
```

**Before Mutation:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - name: app
    image: nginx
```

**After Mutation:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  serviceAccountName: default  # ◄── Added
  containers:
  - name: app
    image: nginx
  volumes:  # ◄── Added
  - name: kube-api-access-xxxxx
    projected:
      sources:
      - serviceAccountToken:
          path: token
      - configMap:
          name: kube-root-ca.crt
          items:
          - key: ca.crt
            path: ca.crt
```

**Token Contents:**
- `token`: JWT for API authentication
- `ca.crt`: Cluster CA certificate
- `namespace`: Current namespace

---

### 5️⃣ NodeRestriction

**Purpose:** Restricts kubelets to only modify their own Node and Pod objects.

**Security Model:**
```
Kubelet on node-1    Kubelet on node-2
      │                    │
      ├─ ✅ Can modify     ├─ ✅ Can modify
      │   node-1           │   node-2
      │   pods on node-1   │   pods on node-2
      │                    │
      └─ ❌ Cannot modify  └─ ❌ Cannot modify
          node-2               node-1
          pods on node-2       pods on node-1
```

**What It Blocks:**
- ❌ Modifying other nodes' objects
- ❌ Setting node labels with `node-restriction.kubernetes.io/` prefix
- ❌ Modifying pods not bound to the kubelet's node

**Why It Matters:** Prevents compromised nodes from affecting other nodes.

**Allowed Operations:**
```bash
# On node-1, kubelet CAN:
kubectl label node node-1 disk=ssd           # ✅
kubectl taint node node-1 dedicated=gpu:NoSchedule  # ✅

# On node-1, kubelet CANNOT:
kubectl label node node-2 disk=ssd           # ❌
kubectl delete node node-2                   # ❌
```

---

### 6️⃣ TaintNodesByCondition

**Purpose:** Automatically adds taints to nodes based on their conditions.

**Node Condition → Taint Mapping:**

| Node Condition | Taint Applied | Effect |
|---------------|---------------|--------|
| `NotReady` | `node.kubernetes.io/not-ready` | NoExecute |
| `Unreachable` | `node.kubernetes.io/unreachable` | NoExecute |
| `OutOfDisk` | `node.kubernetes.io/out-of-disk` | NoSchedule |
| `MemoryPressure` | `node.kubernetes.io/memory-pressure` | NoSchedule |
| `DiskPressure` | `node.kubernetes.io/disk-pressure` | NoSchedule |
| `NetworkUnavailable` | `node.kubernetes.io/network-unavailable` | NoSchedule |

**Workflow:**
```
Node becomes NotReady
        ↓
Controller adds taint: node.kubernetes.io/not-ready:NoExecute
        ↓
Pods without matching toleration are evicted
        ↓
New pods won't schedule until node is Ready
```

**Example:**
```bash
# Node status
kubectl describe node worker-1
Conditions:
  Ready    False   # ◄── Node is not ready
  
Taints:
  node.kubernetes.io/not-ready:NoExecute  # ◄── Auto-added
```

**Toleration to Survive:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: critical-app
spec:
  tolerations:
  - key: "node.kubernetes.io/not-ready"
    operator: "Exists"
    effect: "NoExecute"
    tolerationSeconds: 300  # Stay for 5 minutes
  containers:
  - name: app
    image: nginx
```

---

### 7️⃣ Priority

**Purpose:** Resolves PriorityClass names to priority values and sets default priority.

**Priority Hierarchy:**
```
System Critical (1000000000+)
     ▲
     │
High Priority (1000-999999)
     ▲
     │
Default Priority (0)
     ▲
     │
Low Priority (-1000 to -1)
```

**PriorityClass Definition:**
```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000
globalDefault: false
description: "High priority for critical workloads"
```

**Before Mutation:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  priorityClassName: high-priority  # Name reference
  containers:
  - name: app
    image: nginx
```

**After Mutation:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  priorityClassName: high-priority
  priority: 1000  # ◄── Controller resolves to numeric value
  containers:
  - name: app
    image: nginx
```

**Preemption Example:**
```
Low priority pod running (priority: 0)
        ↓
High priority pod arrives (priority: 1000)
        ↓
Scheduler evicts low priority pod
        ↓
High priority pod scheduled
```

---

### 8️⃣ DefaultTolerationSeconds

**Purpose:** Adds default tolerations with time limits for node taints.

**Default Tolerations Added:**
```yaml
tolerations:
- key: "node.kubernetes.io/not-ready"
  operator: "Exists"
  effect: "NoExecute"
  tolerationSeconds: 300  # 5 minutes

- key: "node.kubernetes.io/unreachable"
  operator: "Exists"
  effect: "NoExecute"
  tolerationSeconds: 300  # 5 minutes
```

**Timeline:**
```
t=0s:  Node becomes NotReady
       ↓
t=0s:  Taint applied by TaintNodesByCondition
       ↓
t=0s:  Pod tolerates taint (stays running)
       ↓
t=300s: Toleration expires
       ↓
t=300s: Pod is evicted and rescheduled
```

**Visual:**
```
Node Healthy          Node Fails          Grace Period          Eviction
    ✅         ──►        ⚠️        ──►    [300 seconds]   ──►    🚫
Pod Running         Pod Tolerating        Waiting...          Pod Evicted
```

**Custom Override:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  tolerations:
  - key: "node.kubernetes.io/not-ready"
    operator: "Exists"
    effect: "NoExecute"
    tolerationSeconds: 0  # Evict immediately
  containers:
  - name: app
    image: nginx
```

---

### 9️⃣ DefaultStorageClass

**Purpose:** Assigns default StorageClass to PVCs without one specified.

**Selection Logic:**
```
PVC without storageClassName
        ↓
    Is there a StorageClass
    with annotation:
    storageclass.kubernetes.io/is-default-class: "true"?
        ↓
    Yes → Assign it
        ↓
    No → Leave empty
```

**Default StorageClass:**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"  # ◄── Default
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
```

**Before Mutation:**
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
  # No storageClassName specified
```

**After Mutation:**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  storageClassName: standard  # ◄── Added by controller
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

**Multiple Default Classes:**
⚠️ If multiple StorageClasses have `is-default-class: "true"`, behavior is undefined. Only mark one as default!

---

### 🔟 StorageObjectInUseProtection

**Purpose:** Adds finalizers to prevent deletion of PVs/PVCs while in use.

**Protection Mechanism:**
```yaml
# PVC with finalizer
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
  finalizers:
  - kubernetes.io/pvc-protection  # ◄── Added automatically
spec:
  # ... spec details
```

**Deletion Flow:**
```
User deletes PVC
      ↓
Is PVC bound to a Pod?
      ↓
  Yes → PVC marked for deletion (DeletionTimestamp set)
  │     PVC remains until Pod is deleted
  │
  └──► Pod deleted
        ↓
      PVC finalizer removed
        ↓
      PVC actually deleted
      
  No → PVC deleted immediately
```

**Visual Timeline:**
```
Pod using PVC          Delete PVC         Delete Pod           PVC Removed
     ✅         ──►    🕐 Pending   ──►      ❌        ──►        ×
PVC Status: Bound     Status: Terminating  Pod Gone          PVC Gone
```

**Check Finalizers:**
```bash
kubectl get pvc my-pvc -o yaml | grep -A 2 finalizers
finalizers:
- kubernetes.io/pvc-protection
```

**Why It Matters:** Prevents data loss by ensuring volumes aren't deleted while actively used.

---

### 1️⃣1️⃣ RuntimeClass

**Purpose:** Validates and mutates pods to use specified container runtimes.

**RuntimeClass Definition:**
```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc  # Runtime handler on the node
scheduling:
  nodeSelector:
    runtime: gvisor
  tolerations:
  - key: "runtime"
    value: "gvisor"
    effect: "NoSchedule"
```

**Pod Requesting RuntimeClass:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
spec:
  runtimeClassName: gvisor  # ◄── Requests RuntimeClass
  containers:
  - name: app
    image: nginx
```

**After Mutation:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
spec:
  runtimeClassName: gvisor
  nodeSelector:  # ◄── Added from RuntimeClass
    runtime: gvisor
  tolerations:   # ◄── Added from RuntimeClass
  - key: "runtime"
    value: "gvisor"
    effect: "NoSchedule"
  overhead:      # ◄── Added if defined
    memory: "50Mi"
    cpu: "100m"
  containers:
  - name: app
    image: nginx
```

**Common Runtime Handlers:**
- `runc` - Default container runtime
- `runsc` - gVisor (sandboxed)
- `kata` - Kata Containers (VM-based)
- `nvidia` - NVIDIA GPU runtime

---

### 1️⃣2️⃣ DefaultIngressClass

**Purpose:** Assigns default IngressClass to Ingress resources without one specified.

**IngressClass with Default:**
```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"  # ◄── Default
spec:
  controller: k8s.io/ingress-nginx
```

**Before Mutation:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web
            port:
              number: 80
  # No ingressClassName specified
```

**After Mutation:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  ingressClassName: nginx  # ◄── Added by controller
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web
            port:
              number: 80
```

**Multiple Controllers:**
```
┌──────────────┐
│   Ingress    │
└──────┬───────┘
       │
   No IngressClass
   specified?
       │
       ├──► Use Default IngressClass (nginx)
       │
       ├──► Or specify explicitly:
       │     ingressClassName: traefik
       │     ingressClassName: haproxy
       │
       └──► Routes to correct controller
```

---

### 1️⃣3️⃣ MutatingAdmissionWebhook

**Purpose:** Enables custom mutation logic via external HTTPS webhooks.

**Architecture:**
```
API Server
    │
    ├──► Built-in Mutating Controllers (1-12)
    │
    └──► MutatingAdmissionWebhook Controller
              │
              └──► Webhook 1 (Istio Sidecar Injection)
              └──► Webhook 2 (Secret Encryption)
              └──► Webhook 3 (Custom Policy)
              └──► Webhook N (Your custom logic)
```

**Webhook Configuration:**
```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: istio-sidecar-injector
webhooks:
- name: sidecar-injector.istio.io
  clientConfig:
    service:
      name: istio-sidecar-injector
      namespace: istio-system
      path: "/inject"
    caBundle: LS0tLS...  # Base64 CA cert
  rules:
  - operations: ["CREATE"]
    apiGroups: [""]
    apiVersions: ["v1"]
    resources: ["pods"]
  namespaceSelector:
    matchLabels:
      istio-injection: enabled
  admissionReviewVersions: ["v1"]
  sideEffects: None
  timeoutSeconds: 10
```

**Request/Response Flow:**
```
1. User creates Pod
        ↓
2. API Server sends AdmissionReview to webhook:
   {
     "request": {
       "uid": "abc123",
       "kind": {"kind": "Pod"},
       "object": { /* Pod spec */ },
       "operation": "CREATE"
     }
   }
        ↓
3. Webhook processes and returns:
   {
     "response": {
       "uid": "abc123",
       "allowed": true,
       "patchType": "JSONPatch",
       "patch": "W3sib3A....==",  # Base64 JSONPatch
     }
   }
        ↓
4. API Server applies patch to Pod
        ↓
5. Pod persisted with modifications
```

**JSONPatch Example:**
```json
[
  {
    "op": "add",
    "path": "/spec/containers/-",
    "value": {
      "name": "istio-proxy",
      "image": "istio/proxyv2:1.20.0"
    }
  },
  {
    "op": "add",
    "path": "/metadata/labels/version",
    "value": "v1.20.0"
  }
]
```

**Common Use Cases:**
- 🔄 Sidecar injection (Istio, Linkerd)
- 🔐 Secret encryption/decryption
- 🏷️ Automatic labeling
- 📊 Resource quota adjustment
- ✅ Custom validation + mutation
- 🔧 Configuration standardization

**Failure Modes:**
```yaml
failurePolicy: Fail    # Reject request if webhook fails (safe)
failurePolicy: Ignore  # Allow request if webhook fails (dangerous)
```

---

## Best Practices

### ✅ DO's

1. **Understand the Order**: Mutating controllers run in the order listed; design accordingly
2. **Use LimitRanger**: Define namespace-level defaults for better resource management
3. **Leverage ServiceAccounts**: Create custom ServiceAccounts with minimal RBAC permissions
4. **Monitor Webhooks**: Set appropriate timeouts and failure policies
5. **Test Mutations**: Verify webhook logic in staging before production

### ❌ DON'Ts

1. **Don't Disable Critical Controllers**: `ServiceAccount`, `NodeRestriction` are essential for security
2. **Don't Create Circular Dependencies**: Webhook dependencies can deadlock the API server
3. **Don't Ignore Failures**: Monitor webhook failures - they can block critical operations
4. **Don't Overuse Webhooks**: Each webhook adds latency; consolidate logic where possible
5. **Don't Mark Multiple StorageClasses as Default**: Causes undefined behavior

---

## Troubleshooting

### Check Enabled Admission Controllers
```bash
kubectl -n kube-system logs kube-apiserver-<name> | grep 'admission'
```

### Debug Webhook Issues
```bash
# Check webhook configuration
kubectl get mutatingwebhookconfigurations

# View webhook details
kubectl describe mutatingwebhookconfigurations <name>

# Check webhook service
kubectl get svc -n <namespace>

# Test connectivity
kubectl run test --rm -it --image=curlimages/curl -- \
  curl -k https://<webhook-service>.<namespace>.svc:443/health
```

### Common Errors

**Error:** `admission webhook denied the request`
- **Cause**: Webhook validation failed or timed out
- **Fix**: Check webhook logs, verify network connectivity, increase timeout

**Error:** `no matches for kind "PriorityClass"`
- **Cause**: PriorityClass CRD not registered (old cluster)
- **Fix**: Upgrade cluster or disable Priority admission controller

**Error:** `storageclass.storage.k8s.io "standard" not found`
- **Cause**: Default StorageClass doesn't exist
- **Fix**: Create StorageClass with `is-default-class: "true"` annotation

---

## Quick Reference Card

| Controller | Primary Function | Key Mutation |
|-----------|------------------|--------------|
| NamespaceAutoProvision | Auto-create namespaces | Creates missing namespaces |
| NamespaceLifecycle | Prevent ops in terminating NS | Blocks requests |
| LimitRanger | Apply resource defaults | Adds resource limits |
| ServiceAccount | Auto-assign SA | Adds SA + token volume |
| NodeRestriction | Restrict kubelet scope | Enforces node-level isolation |
| TaintNodesByCondition | Auto-taint nodes | Adds condition taints |
| Priority | Resolve priority | Adds numeric priority |
| DefaultTolerationSeconds | Add taint tolerations | Adds 300s tolerations |
| DefaultStorageClass | Assign default SC | Adds storageClassName |
| StorageObjectInUseProtection | Prevent PV/PVC deletion | Adds finalizers |
| RuntimeClass | Apply runtime config | Adds nodeSelector/tolerations |
| DefaultIngressClass | Assign default ingress | Adds ingressClassName |
| MutatingAdmissionWebhook | Custom mutations | Applies webhook patches |

---

## Additional Resources

- [Official Kubernetes Docs - Admission Controllers](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)
- [Dynamic Admission Control](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/)
- [Admission Webhooks Guide](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#what-are-they)

---

**Document Version**: 1.0  
**Last Updated**: January 2026  
**Author**: Kubernetes Expert Documentation Team

---

## Summary

Mutating admission controllers are essential for:
- 🔒 **Security**: ServiceAccount, NodeRestriction
- ⚙️ **Automation**: Default values for storage, ingress, tolerations
- 📊 **Resource Management**: LimitRanger, Priority
- 🛡️ **Protection**: StorageObjectInUseProtection, NamespaceLifecycle
- 🔧 **Extensibility**: MutatingAdmissionWebhook for custom logic

Understanding these controllers helps you build secure, efficient, and well-governed Kubernetes clusters!