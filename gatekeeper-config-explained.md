# OPA Gatekeeper Configuration: Complete Breakdown

## 📚 Table of Contents
- [What is OPA Gatekeeper?](#what-is-opa-gatekeeper)
- [Architecture Overview](#architecture-overview)
- [Relationship to Admission Controllers](#relationship-to-admission-controllers)
- [Component Breakdown](#component-breakdown)
- [Webhook Configurations](#webhook-configurations)
- [Custom Resource Definitions (CRDs)](#custom-resource-definitions-crds)
- [Deployments](#deployments)
- [RBAC Configuration](#rbac-configuration)
- [Practical Examples](#practical-examples)
- [Troubleshooting](#troubleshooting)

---

## What is OPA Gatekeeper?

**OPA Gatekeeper** is a policy engine for Kubernetes that:
- Enforces custom policies using the Open Policy Agent (OPA)
- Validates and mutates Kubernetes resources
- Uses the **MutatingAdmissionWebhook** and **ValidatingAdmissionWebhook** admission controllers
- Allows you to write policies as code using Rego language

### Key Capabilities

```
┌─────────────────────────────────────────┐
│     OPA Gatekeeper Capabilities         │
├─────────────────────────────────────────┤
│  ✅ Policy Enforcement                  │
│     - Block disallowed resources        │
│     - Enforce naming conventions        │
│     - Require labels/annotations        │
│                                         │
│  🔄 Resource Mutation                   │
│     - Auto-add labels                   │
│     - Inject sidecars                   │
│     - Set default values                │
│                                         │
│  📊 Audit & Compliance                  │
│     - Scan existing resources           │
│     - Generate violation reports        │
│     - Track policy violations           │
│                                         │
│  🎯 Custom Constraints                  │
│     - Define custom policies            │
│     - Reusable constraint templates     │
│     - Parameterized rules               │
└─────────────────────────────────────────┘
```

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    Kubernetes API Server                        │
└────────────┬────────────────────────────────────┬───────────────┘
             │                                    │
             │ Admission Request                  │ Admission Request
             │ (CREATE/UPDATE)                    │ (CREATE/UPDATE)
             │                                    │
             ▼                                    ▼
┌────────────────────────────┐    ┌─────────────────────────────┐
│  Mutating Webhook          │    │  Validating Webhook         │
│  (mutation.gatekeeper.sh)  │    │  (validation.gatekeeper.sh) │
│                            │    │                             │
│  • Modify resources        │    │  • Allow/Deny requests      │
│  • Add labels/annotations  │    │  • Enforce policies         │
│  • Inject values           │    │  • Check constraints        │
│                            │    │                             │
│  Timeout: 1 second         │    │  Timeout: 3 seconds         │
│  FailurePolicy: Ignore     │    │  FailurePolicy: Ignore/Fail │
└────────────┬───────────────┘    └────────────┬────────────────┘
             │                                  │
             │                                  │
             └──────────────┬───────────────────┘
                            │
                            ▼
              ┌─────────────────────────────┐
              │  Gatekeeper Service         │
              │  Port: 443                  │
              │  Target: webhook-server     │
              └─────────────┬───────────────┘
                            │
                            ▼
              ┌─────────────────────────────┐
              │  Controller Manager Pods    │
              │  (3 replicas)               │
              │                             │
              │  • Process webhooks         │
              │  • Evaluate policies        │
              │  • Apply mutations          │
              │  • Validate constraints     │
              └─────────────────────────────┘

              ┌─────────────────────────────┐
              │  Audit Controller Pod       │
              │  (1 replica)                │
              │                             │
              │  • Scan existing resources  │
              │  • Generate reports         │
              │  • Track violations         │
              └─────────────────────────────┘
```

---

## Relationship to Admission Controllers

Gatekeeper extends Kubernetes admission controllers (specifically #13 from the main documentation):

### Integration with Built-in Controllers

```
Request Flow with Gatekeeper:

kubectl apply -f pod.yaml
        ↓
API Server Authentication/Authorization
        ↓
┌───────────────────────────────────────────┐
│  Built-in Mutating Admission Controllers  │
│  1. NamespaceLifecycle                    │
│  2. LimitRanger                           │
│  3. ServiceAccount                        │
│  ... (controllers 1-12)                   │
└───────────────┬───────────────────────────┘
                ↓
┌───────────────────────────────────────────┐
│  13. MutatingAdmissionWebhook             │
│      ↓                                    │
│  ┌──────────────────────────────────┐    │
│  │  Gatekeeper Mutating Webhook     │    │
│  │  - Assign mutations              │    │
│  │  - AssignMetadata mutations      │    │
│  │  - ModifySet mutations           │    │
│  └──────────────────────────────────┘    │
└───────────────┬───────────────────────────┘
                ↓
┌───────────────────────────────────────────┐
│  Validating Admission Controllers         │
│      ↓                                    │
│  ┌──────────────────────────────────┐    │
│  │  Gatekeeper Validating Webhook   │    │
│  │  - ConstraintTemplates           │    │
│  │  - Custom Constraints            │    │
│  │  - Policy enforcement            │    │
│  └──────────────────────────────────┘    │
└───────────────┬───────────────────────────┘
                ↓
        Persist to etcd ✅
```

---

## Component Breakdown

### 1️⃣ ResourceQuota

**Purpose:** Ensures Gatekeeper critical pods can always be scheduled.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: gatekeeper-critical-pods
  namespace: gatekeeper-system
spec:
  hard:
    pods: 100  # Allow up to 100 critical pods
  scopeSelector:
    matchExpressions:
    - operator: In
      scopeName: PriorityClass
      values:
      - system-cluster-critical  # Only applies to critical pods
```

**Why It Matters:**
- Gatekeeper pods use `system-cluster-critical` priority
- Ensures they can start even under resource pressure
- Prevents cluster-wide policy enforcement failures

---

### 2️⃣ Secret & Service

#### Webhook Server Certificate Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: gatekeeper-webhook-server-cert
  namespace: gatekeeper-system
# Certificate data populated by Gatekeeper
```

**Purpose:**
- Stores TLS certificates for webhook HTTPS communication
- Auto-generated by Gatekeeper controller
- Used to secure API Server → Gatekeeper communication

#### Webhook Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: gatekeeper-webhook-service
  namespace: gatekeeper-system
spec:
  ports:
  - name: https-webhook-server
    port: 443
    targetPort: webhook-server  # Maps to container port 8473
  selector:
    control-plane: controller-manager
    gatekeeper.sh/operation: webhook
```

**Purpose:**
- Provides stable endpoint for API Server webhook calls
- Routes to controller-manager pods (3 replicas)
- Enables high availability

---

## Webhook Configurations

### 3️⃣ MutatingWebhookConfiguration

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: gatekeeper-mutating-webhook-configuration
webhooks:
- name: mutation.gatekeeper.sh
  clientConfig:
    service:
      name: gatekeeper-webhook-service
      namespace: gatekeeper-system
      path: /v1/mutate  # ◄── Mutation endpoint
  failurePolicy: Ignore  # ◄── Continue if webhook fails
  timeoutSeconds: 1      # ◄── Fast timeout
  rules:
  - apiGroups: ["*"]
    apiVersions: ["*"]
    operations: ["CREATE", "UPDATE"]
    resources: ["*"]
  namespaceSelector:
    matchExpressions:
    - key: admission.gatekeeper.sh/ignore
      operator: DoesNotExist  # Skip if namespace has ignore label
    - key: kubernetes.io/metadata.name
      operator: NotIn
      values:
      - gatekeeper-system  # Don't mutate Gatekeeper itself
```

**Configuration Analysis:**

| Setting | Value | Meaning |
|---------|-------|---------|
| `failurePolicy` | Ignore | If webhook times out/fails, allow request (safe default) |
| `timeoutSeconds` | 1 | Quick timeout to avoid blocking deployments |
| `matchPolicy` | Exact | Only exact matches to rules |
| `reinvocationPolicy` | Never | Don't call again after other mutations |
| `sideEffects` | None | Webhook doesn't cause external changes |

**Namespace Exclusions:**

```
┌──────────────────────────────────┐
│  Namespaces Mutated by Gatekeeper│
├──────────────────────────────────┤
│  ✅ default                       │
│  ✅ production                    │
│  ✅ staging                       │
│  ✅ custom-namespaces             │
├──────────────────────────────────┤
│  ❌ gatekeeper-system (excluded) │
│  ❌ Namespaces with label:       │
│     admission.gatekeeper.sh/     │
│     ignore=true                  │
└──────────────────────────────────┘
```

---

### 4️⃣ ValidatingWebhookConfiguration

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: gatekeeper-validating-webhook-configuration
webhooks:
- name: validation.gatekeeper.sh
  clientConfig:
    service:
      name: gatekeeper-webhook-service
      namespace: gatekeeper-system
      path: /v1/admit  # ◄── Validation endpoint
  failurePolicy: Ignore  # Safe default
  timeoutSeconds: 3      # Longer timeout than mutation
  rules:
  - apiGroups: ["*"]
    apiVersions: ["*"]
    operations: ["CREATE", "UPDATE"]
    resources: ["*", "pods/ephemeralcontainers", ...]
```

**Additional Webhook: Label Validation**

```yaml
- name: check-ignore-label.gatekeeper.sh
  path: /v1/admitlabel
  failurePolicy: Fail  # ◄── Critical: must succeed
  rules:
  - apiGroups: [""]
    apiVersions: ["*"]
    operations: ["CREATE", "UPDATE"]
    resources: ["namespaces"]
```

**Purpose:** Prevents users from adding `admission.gatekeeper.sh/ignore` label to bypass policies.

**Comparison:**

| Aspect | Mutating Webhook | Validating Webhook |
|--------|------------------|-------------------|
| **Endpoint** | `/v1/mutate` | `/v1/admit` |
| **Timeout** | 1 second | 3 seconds |
| **Purpose** | Modify resources | Accept/reject resources |
| **Failure** | Ignore (safe) | Ignore (namespace check is Fail) |
| **Order** | First | After mutations |

---

## Custom Resource Definitions (CRDs)

Gatekeeper defines several CRDs for policy management:

### 5️⃣ Mutation CRDs

#### A. Assign CRD

**Purpose:** Assigns values to specific fields in Kubernetes resources.

```yaml
apiVersion: mutations.gatekeeper.sh/v1
kind: Assign
metadata:
  name: assign-example
spec:
  applyTo:
  - groups: [""]
    kinds: ["Pod"]
    versions: ["v1"]
  location: "metadata.labels.managed-by"
  parameters:
    assign:
      value: "gatekeeper"
```

**Real-World Example:**

```yaml
apiVersion: mutations.gatekeeper.sh/v1
kind: Assign
metadata:
  name: add-security-label
spec:
  applyTo:
  - groups: ["apps"]
    kinds: ["Deployment"]
    versions: ["v1"]
  match:
    namespaces: ["production"]
  location: "spec.template.metadata.labels.security-scan"
  parameters:
    assign:
      value: "required"
```

**Before Mutation:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: production
spec:
  template:
    metadata:
      labels:
        app: my-app
```

**After Mutation:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: production
spec:
  template:
    metadata:
      labels:
        app: my-app
        security-scan: "required"  # ◄── Added by Gatekeeper
```

---

#### B. AssignMetadata CRD

**Purpose:** Assigns values to metadata fields (labels, annotations).

```yaml
apiVersion: mutations.gatekeeper.sh/v1
kind: AssignMetadata
metadata:
  name: assign-metadata-example
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
  location: "metadata.labels.injected"
  parameters:
    assign:
      value: "true"
```

**Use Case: Auto-labeling**

```yaml
apiVersion: mutations.gatekeeper.sh/v1
kind: AssignMetadata
metadata:
  name: add-cost-center
spec:
  match:
    namespaces: ["*"]
    kinds:
    - apiGroups: [""]
      kinds: ["Pod", "Service"]
  location: "metadata.labels.cost-center"
  parameters:
    assign:
      fromMetadata:
        field: namespace  # Use namespace name as label value
```

---

#### C. ModifySet CRD

**Purpose:** Modifies non-keyed lists (like container args).

```yaml
apiVersion: mutations.gatekeeper.sh/v1
kind: ModifySet
metadata:
  name: add-log-flags
spec:
  applyTo:
  - groups: ["apps"]
    kinds: ["Deployment"]
    versions: ["v1"]
  location: "spec.template.spec.containers[name:*].args"
  parameters:
    operation: merge  # or "prune"
    values:
      fromList:
      - "--log-level=info"
      - "--enable-metrics"
```

**Before:**
```yaml
containers:
- name: app
  args:
  - "--port=8080"
```

**After:**
```yaml
containers:
- name: app
  args:
  - "--port=8080"
  - "--log-level=info"    # ◄── Added
  - "--enable-metrics"    # ◄── Added
```

---

### 6️⃣ Validation CRDs

#### ConstraintTemplate

**Purpose:** Defines reusable policy templates using Rego.

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items:
                type: string
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8srequiredlabels

      violation[{"msg": msg, "details": {"missing_labels": missing}}] {
        provided := {label | input.review.object.metadata.labels[label]}
        required := {label | label := input.parameters.labels[_]}
        missing := required - provided
        count(missing) > 0
        msg := sprintf("Missing required labels: %v", [missing])
      }
```

**What This Does:**
1. Defines a new CRD: `K8sRequiredLabels`
2. Accepts parameter: `labels` (array)
3. Validates that resources have required labels
4. Returns violation if labels are missing

---

#### Using the ConstraintTemplate

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels  # ◄── Created by template
metadata:
  name: all-must-have-owner
spec:
  match:
    kinds:
    - apiGroups: ["apps"]
      kinds: ["Deployment", "StatefulSet"]
  parameters:
    labels:
    - "owner"
    - "environment"
```

**Enforcement:**

```yaml
# This will be REJECTED ❌
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bad-deployment
  labels:
    app: my-app
    # Missing: owner, environment

# This will be ACCEPTED ✅
apiVersion: apps/v1
kind: Deployment
metadata:
  name: good-deployment
  labels:
    app: my-app
    owner: "team-platform"
    environment: "production"
```

---

### 7️⃣ Config CRD

**Purpose:** Configure Gatekeeper behavior.

```yaml
apiVersion: config.gatekeeper.sh/v1alpha1
kind: Config
metadata:
  name: config
  namespace: gatekeeper-system
spec:
  # Exclude namespaces from policy enforcement
  match:
  - excludedNamespaces:
    - "kube-system"
    - "kube-public"
    - "gatekeeper-system"
    processes:
    - "audit"
    - "webhook"
  
  # Sync Kubernetes resources to OPA cache
  sync:
    syncOnly:
    - group: ""
      version: "v1"
      kind: "Namespace"
    - group: ""
      version: "v1"
      kind: "Pod"
```

---

## Deployments

### 8️⃣ Controller Manager Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gatekeeper-controller-manager
  namespace: gatekeeper-system
spec:
  replicas: 3  # ◄── High availability
  template:
    spec:
      priorityClassName: system-cluster-critical
      hostNetwork: true  # ◄── Uses host network
      containers:
      - name: manager
        image: openpolicyagent/gatekeeper:v3.11.0
        args:
        - --port=8473
        - --health-addr=:9191
        - --prometheus-port=8899
        - --exempt-namespace=gatekeeper-system
        - --operation=webhook
        - --operation=mutation-webhook
        - --enable-external-data=true
        - --tls-min-version=1.3
        - --disable-opa-builtin={http.send}
        ports:
        - containerPort: 8473
          name: webhook-server
        - containerPort: 8899
          name: metrics
        - containerPort: 9191
          name: healthz
        resources:
          requests:
            cpu: 100m
            memory: 512Mi
          limits:
            memory: 512Mi
```

**Key Features:**

| Feature | Purpose |
|---------|---------|
| `replicas: 3` | High availability, load balancing |
| `hostNetwork: true` | Direct access to node network |
| `priorityClassName: system-cluster-critical` | Won't be evicted under pressure |
| `--exempt-namespace` | Gatekeeper won't validate itself |
| `--tls-min-version=1.3` | Strong encryption |
| `--disable-opa-builtin={http.send}` | Security: prevent external calls |

**Pod Anti-Affinity:**

```yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchExpressions:
          - key: gatekeeper.sh/operation
            operator: In
            values:
            - webhook
        topologyKey: kubernetes.io/hostname
```

**Effect:** Spreads 3 replicas across different nodes for resilience.

---

### 9️⃣ Audit Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gatekeeper-audit
  namespace: gatekeeper-system
spec:
  replicas: 1  # ◄── Single instance
  template:
    spec:
      containers:
      - name: manager
        image: openpolicyagent/gatekeeper:v3.11.0
        args:
        - --audit-interval=60  # Scan every 60 seconds
        - --constraint-violations-limit=20
        - --audit-from-cache=false
        - --audit-chunk-size=500
        - --operation=audit
        - --operation=status
        - --operation=mutation-status
        ports:
        - containerPort: 8474
          name: metrics
        - containerPort: 9192
          name: healthz
```

**Purpose:**
- Periodically scans **existing** resources in the cluster
- Generates violation reports for resources that don't comply
- Doesn't block resource creation (runs asynchronously)

**Audit vs. Webhook:**

```
Webhook (Controller Manager)       Audit Controller
         ▼                                ▼
  Real-time enforcement          Periodic compliance scan
         │                                │
    Block CREATE/                   Scan existing
    UPDATE if policy               resources every
    violations                      60 seconds
         │                                │
         ▼                                ▼
  Prevent bad resources           Report violations
  from being created              for existing resources
```

---

### 🔟 PodDisruptionBudget

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: gatekeeper-controller-manager
  namespace: gatekeeper-system
spec:
  minAvailable: 1  # ◄── At least 1 pod must be running
  selector:
    matchLabels:
      control-plane: controller-manager
```

**Purpose:**
- During node maintenance/upgrades, ensures at least 1 webhook pod is always available
- Prevents "downtime" where all webhook pods are unavailable
- Critical for cluster operations (webhooks must always respond)

---

## RBAC Configuration

### 1️⃣1️⃣ ClusterRole

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: gatekeeper-manager-role
rules:
# Read all resources (for audit)
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]

# Manage webhook configurations
- apiGroups: ["admissionregistration.k8s.io"]
  resources: ["mutatingwebhookconfigurations", "validatingwebhookconfigurations"]
  resourceNames:
  - "gatekeeper-mutating-webhook-configuration"
  - "gatekeeper-validating-webhook-configuration"
  verbs: ["get", "list", "patch", "update", "watch"]

# Manage CRDs
- apiGroups: ["apiextensions.k8s.io"]
  resources: ["customresourcedefinitions"]
  verbs: ["create", "delete", "get", "list", "patch", "update", "watch"]

# Manage Gatekeeper resources
- apiGroups:
  - "config.gatekeeper.sh"
  - "constraints.gatekeeper.sh"
  - "mutations.gatekeeper.sh"
  - "templates.gatekeeper.sh"
  - "status.gatekeeper.sh"
  resources: ["*"]
  verbs: ["create", "delete", "get", "list", "patch", "update", "watch"]
```

**Permission Breakdown:**

| Permission | Purpose |
|------------|---------|
| Read `*/*` | Audit existing resources |
| Manage webhooks | Self-configure admission webhooks |
| Manage CRDs | Create/update ConstraintTemplate CRDs |
| Manage Gatekeeper CRs | Handle policies, constraints, mutations |

---

### 1️⃣2️⃣ Role (Namespace-scoped)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: gatekeeper-manager-role
  namespace: gatekeeper-system
rules:
- apiGroups: [""]
  resources: ["events"]
  verbs: ["create", "patch"]  # Emit audit events

- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["create", "delete", "get", "list", "patch", "update", "watch"]
  # Manage TLS certificates
```

---

## Practical Examples

### Example 1: Require Labels

**Step 1: Create ConstraintTemplate**

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items:
                type: string
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8srequiredlabels

      violation[{"msg": msg}] {
        provided := {label | input.review.object.metadata.labels[label]}
        required := {label | label := input.parameters.labels[_]}
        missing := required - provided
        count(missing) > 0
        msg := sprintf("Missing required labels: %v", [missing])
      }
```

**Step 2: Create Constraint**

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: ns-must-have-env
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Namespace"]
  parameters:
    labels: ["environment", "owner"]
```

**Step 3: Test**

```bash
# This will FAIL ❌
kubectl create namespace test-ns

# Error from server: Missing required labels: {"environment" "owner"}

# This will SUCCEED ✅
kubectl create namespace test-ns --labels environment=dev,owner=team-platform
```

---

### Example 2: Auto-Add Security Context

**Create Assign Mutation:**

```yaml
apiVersion: mutations.gatekeeper.sh/v1
kind: Assign
metadata:
  name: add-security-context
spec:
  applyTo:
  - groups: [""]
    kinds: ["Pod"]
    versions: ["v1"]
  match:
    namespaces: ["production"]
  location: "spec.securityContext.runAsNonRoot"
  parameters:
    assign:
      value: true
```

**Test:**

```yaml
# Create pod without securityContext
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
  namespace: production
spec:
  containers:
  - name: app
    image: nginx

# Gatekeeper automatically adds:
spec:
  securityContext:
    runAsNonRoot: true  # ◄── Added by mutation
  containers:
  - name: app
    image: nginx
```

---

### Example 3: Block Privileged Containers

**ConstraintTemplate:**

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sblockprivileged
spec:
  crd:
    spec:
      names:
        kind: K8sBlockPrivileged
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8sblockprivileged

      violation[{"msg": msg}] {
        container := input.review.object.spec.containers[_]
        container.securityContext.privileged
        msg := sprintf("Privileged container not allowed: %v", [container.name])
      }
```

**Constraint:**

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sBlockPrivileged
metadata:
  name: no-privileged-containers
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
```

**Result:**

```yaml
# This will be REJECTED ❌
apiVersion: v1
kind: Pod
metadata:
  name: bad-pod
spec:
  containers:
  - name: app
    image: nginx
    securityContext:
      privileged: true  # ◄── Violation!

# Error: Privileged container not allowed: app
```

---

## Troubleshooting

### Check Gatekeeper Status

```bash
# Check controller pods
kubectl get pods -n gatekeeper-system

# Check webhook configurations
kubectl get mutatingwebhookconfigurations
kubectl get validatingwebhookconfigurations

# View audit results
kubectl get constraints -A

# Check specific constraint violations
kubectl describe k8srequiredlabels ns-must-have-env
```

### Common Issues

#### 1. Webhook Timeout

**Symptom:**
```
Error from server: Internal error occurred: failed calling webhook...
context deadline exceeded
```

**Solution:**
```yaml
# Increase timeout in webhook config
webhooks:
- name: validation.gatekeeper.sh
  timeoutSeconds: 10  # Increase from 3
```

#### 2. Certificate Issues

**Symptom:**
```
unable to dial webhook: x509: certificate signed by unknown authority
```

**Solution:**
```bash
# Delete and recreate certificates
kubectl delete secret gatekeeper-webhook-server-cert -n gatekeeper-system

# Restart Gatekeeper
kubectl rollout restart deployment gatekeeper-controller-manager -n gatekeeper-system
```

#### 3. Policies Not Enforced

**Check:**
```bash
# Verify constraint is enforced
kubectl get constraints -A

# Check if namespace is excluded
kubectl get namespace <ns> -o yaml | grep admission.gatekeeper.sh

# View Gatekeeper logs
kubectl logs -n gatekeeper-system deployment/gatekeeper-controller-manager
```

---

## Monitoring & Metrics

### Prometheus Metrics

Gatekeeper exposes metrics on:
- **Controller Manager:** Port `8899`
- **Audit:** Port `8474`

**Key Metrics:**

| Metric | Description |
|--------|-------------|
| `gatekeeper_violations` | Current policy violations |
| `gatekeeper_constraint_templates` | Number of active templates |
| `gatekeeper_constraints` | Number of active constraints |
| `gatekeeper_webhook_request_duration` | Webhook latency |
| `gatekeeper_audit_duration` | Audit scan time |

**Grafana Dashboard Query:**

```promql
# Webhook request latency (p95)
histogram_quantile(0.95, 
  rate(gatekeeper_webhook_request_duration_seconds_bucket[5m])
)

# Total violations by kind
sum by (kind) (gatekeeper_violations)
```

---

## Best Practices

### ✅ DO's

1. **Start with Audit Mode**
   - Use audit controller first to identify violations
   - Don't enforce policies immediately in production

2. **Use Namespace Exclusions**
   ```yaml
   namespaceSelector:
     matchExpressions:
     - key: admission.gatekeeper.sh/ignore
       operator: DoesNotExist
   ```

3. **Set Reasonable Timeouts**
   - Mutation: 1-2 seconds
   - Validation: 3-5 seconds

4. **Test Policies in Staging**
   - Apply constraints to dev/staging first
   - Monitor violation reports before enforcing

5. **Use Descriptive Error Messages**
   ```rego
   msg := sprintf("Pod %v must have owner label", [input.review.object.metadata.name])
   ```

### ❌ DON'Ts

1. **Don't Block System Namespaces**
   - Always exclude `kube-system`, `kube-public`
   - Gatekeeper can break cluster-critical workloads

2. **Don't Use `failurePolicy: Fail` Initially**
   - Start with `Ignore` to avoid blocking deployments
   - Switch to `Fail` after thorough testing

3. **Don't Create Overlapping Constraints**
   - Can cause confusing errors
   - Consolidate related policies

4. **Don't Ignore Performance**
   - Complex Rego policies can slow webhooks
   - Monitor webhook latency metrics

---

## Summary

Your Gatekeeper configuration provides:

✅ **Policy Enforcement:** Custom constraints via ConstraintTemplates  
✅ **Resource Mutation:** Auto-add labels, security contexts, etc.  
✅ **Audit Compliance:** Periodic scans of existing resources  
✅ **High Availability:** 3 webhook replicas with anti-affinity  
✅ **Security:** TLS 1.3, non-root containers, disabled external calls  
✅ **Observability:** Prometheus metrics, health checks

**Integration with Admission Controllers:**
- Uses **MutatingAdmissionWebhook** (#13) for mutations
- Uses **ValidatingAdmissionWebhook** for constraint enforcement
- Extends Kubernetes native admission control with custom policies

**Next Steps:**
1. Deploy ConstraintTemplates for your use cases
2. Create Constraints to enforce policies
3. Monitor audit reports for existing violations
4. Gradually enforce policies in production

---

**Document Version:** 1.0  
**Compatible with:** Gatekeeper v3.11.0  
**Last Updated:** January 2026