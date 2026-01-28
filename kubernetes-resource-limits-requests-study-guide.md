# Kubernetes Resource Limits & Requests - Student Study Guide

> **A practical guide for mastering resource management in Kubernetes with hands-on examples**

---

## Table of Contents
1. [What are Resource Requests and Limits?](#what-are-resource-requests-and-limits)
2. [How Resource Management Works](#how-resource-management-works)
3. [Resource Types](#resource-types)
4. [Requests vs Limits - Understanding the Difference](#requests-vs-limits---understanding-the-difference)
5. [Practical Examples](#practical-examples)
6. [LimitRange - Setting Defaults](#limitrange---setting-defaults)
7. [ResourceQuota - Namespace-Level Limits](#resourcequota---namespace-level-limits)
8. [QoS Classes](#qos-classes)
9. [Pod-Level Resources (New!)](#pod-level-resources-new)
10. [Best Practices](#best-practices)
11. [Troubleshooting](#troubleshooting)
12. [Quick Reference](#quick-reference)

---

## What are Resource Requests and Limits?

In Kubernetes, **resources** refer to compute resources like CPU and Memory that containers need to run. Think of them as:

- 🎫 **Requests** = Minimum guaranteed resources (like a reservation)
- 🚧 **Limits** = Maximum allowed resources (like a speed limit)

### Why Do We Need Them?

Without resource management:
- ❌ One pod could consume all node resources
- ❌ Applications might crash due to lack of resources
- ❌ Cluster becomes unstable and unpredictable
- ❌ Can't plan capacity effectively

With resource management:
- ✅ Fair resource allocation
- ✅ Predictable performance
- ✅ Better cluster utilization
- ✅ Prevents resource starvation

---

## How Resource Management Works

### The Complete Flow

```
┌─────────────────────────────────────────────────────┐
│  1. Pod Creation Request                            │
│     resources:                                       │
│       requests: { cpu: 500m, memory: 512Mi }       │
│       limits: { cpu: 1, memory: 1Gi }              │
└───────────────────┬─────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────┐
│  2. Scheduler Decision                              │
│     - Finds nodes with available capacity           │
│     - Checks: Sum of requests ≤ Node capacity      │
│     - Schedules to best-fit node                    │
└───────────────────┬─────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────┐
│  3. Kubelet Enforcement                             │
│     - Configures cgroups with limits                │
│     - Guarantees request amount                     │
│     - Enforces limit constraints                    │
└───────────────────┬─────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────┐
│  4. Runtime Behavior                                │
│     CPU:  Throttled if exceeds limit               │
│     Memory: OOMKilled if exceeds limit             │
└─────────────────────────────────────────────────────┘
```

---

## Resource Types

### 1. CPU Resources

**Units:**
- 1 CPU = 1 physical core or 1 vCPU
- Can use millicores: `1000m` = 1 CPU
- Minimum precision: `1m` (0.001 CPU)

**Examples:**
```yaml
cpu: "1"        # 1 full CPU core
cpu: "0.5"      # Half a CPU core
cpu: "500m"     # 500 millicores (same as 0.5)
cpu: "100m"     # 0.1 CPU
cpu: "2000m"    # 2 CPUs
```

**Important Notes:**
- CPU is **compressible** - can be throttled without killing the pod
- Throttling = container has to wait longer for CPU time
- No pod termination for CPU limit violations

### 2. Memory Resources

**Units:**
- Bytes: plain integer
- Decimal: K, M, G, T, P, E
- Binary: Ki, Mi, Gi, Ti, Pi, Ei

**Examples:**
```yaml
memory: "128974848"     # 128974848 bytes
memory: "129e6"         # 129000000 bytes (129MB)
memory: "129M"          # 129 megabytes
memory: "123Mi"         # 123 mebibytes
memory: "1Gi"           # 1 gibibyte
```

**Important Notes:**
- Memory is **incompressible** - can't be squeezed like CPU
- Exceeding limit = OOMKilled (Out Of Memory Kill)
- Pod will be terminated and potentially restarted

**Common Mistake:**
```yaml
memory: "400m"  # ❌ This means 0.0004 bytes, NOT 400 mebibytes!
memory: "400Mi" # ✅ Correct: 400 mebibytes
memory: "400M"  # ✅ Also correct: 400 megabytes
```

### 3. Huge Pages (Advanced)

```yaml
resources:
  limits:
    hugepages-2Mi: 100Mi  # 50 x 2Mi pages
    hugepages-1Gi: 2Gi    # 2 x 1Gi pages
```

---

## Requests vs Limits - Understanding the Difference

### Visual Comparison

```
Node Capacity: 4 CPU, 16Gi Memory

┌─────────────────────────────────────────────┐
│  Node Resources                              │
│                                              │
│  ┌──────────┐ ← Limit (max can use)        │
│  │          │                                │
│  │  BURST   │   (Can use if available)      │
│  │          │                                │
│  ├──────────┤ ← Request (guaranteed)        │
│  │          │                                │
│  │ RESERVED │   (Always available)           │
│  │          │                                │
│  └──────────┘                                │
│                                              │
└─────────────────────────────────────────────┘
```

### Key Differences

| Aspect | Requests | Limits |
|--------|----------|--------|
| **Purpose** | Minimum guarantee | Maximum ceiling |
| **Scheduling** | Used by scheduler | Not used for scheduling |
| **Enforcement** | Guaranteed available | Hard limit enforced |
| **Exceeding** | Can use more if available | Cannot exceed (throttled/killed) |
| **Default** | No default (optional) | No default (optional) |
| **CPU Behavior** | Scheduling weight | Throttling |
| **Memory Behavior** | Scheduling check | OOM kill if exceeded |

### Behavior Matrix

#### Scenario 1: No Requests or Limits
```yaml
resources: {}
```
- ❌ No guarantees
- ❌ Can consume all node resources
- ❌ Can starve other pods
- ❌ Unpredictable performance

#### Scenario 2: Only Limits (No Requests)
```yaml
resources:
  limits:
    cpu: "1"
    memory: "1Gi"
```
- ✅ Kubernetes sets: requests = limits
- ✅ Guaranteed QoS class
- ❌ Less flexible resource sharing

#### Scenario 3: Only Requests (No Limits)
```yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
```
- ✅ Guaranteed minimum
- ✅ Can burst to use more
- ⚠️ Can potentially overuse and get evicted
- ✅ Better resource utilization

#### Scenario 4: Both Requests and Limits
```yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "1"
    memory: "1Gi"
```
- ✅ Guaranteed minimum (500m, 512Mi)
- ✅ Can burst up to limit (1 CPU, 1Gi)
- ✅ Protected from excessive usage
- ✅ Best balance (RECOMMENDED)

---

## Practical Examples

### Example 1: Simple Pod with Resources

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-with-resources
spec:
  containers:
  - name: nginx
    image: nginx:latest
    resources:
      requests:
        memory: "256Mi"
        cpu: "250m"
      limits:
        memory: "512Mi"
        cpu: "500m"
```

**What this means:**
- Scheduler ensures node has ≥ 256Mi memory and ≥ 250m CPU available
- Container is guaranteed 256Mi memory and 250m CPU
- Can burst up to 512Mi memory and 500m CPU
- If exceeds 512Mi → OOMKilled
- If exceeds 500m CPU → Throttled (slowed down)

**Test it:**
```bash
kubectl apply -f nginx-with-resources.yaml

# Check assigned node
kubectl get pod nginx-with-resources -o wide

# Check actual resource usage
kubectl top pod nginx-with-resources
```

---

### Example 2: Multi-Container Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-app
spec:
  containers:
  - name: app
    image: myapp:v1
    resources:
      requests:
        cpu: "500m"
        memory: "512Mi"
      limits:
        cpu: "1"
        memory: "1Gi"
  
  - name: sidecar-logger
    image: fluentd:latest
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "200m"
        memory: "256Mi"
```

**Pod-level totals:**
- **Requests:** 600m CPU, 640Mi memory
- **Limits:** 1.2 CPU, 1280Mi memory

The scheduler looks for a node with at least 600m CPU and 640Mi memory available.

---

### Example 3: Deployment with Resources

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: nginx:latest
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "250m"
            memory: "256Mi"
```

**Total cluster requirements:**
- **Requests:** 3 × (100m CPU + 128Mi memory) = 300m CPU, 384Mi memory
- **Limits:** 3 × (250m CPU + 256Mi memory) = 750m CPU, 768Mi memory

---

### Example 4: Stress Testing Resources

**Create a memory-hungry pod:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo
spec:
  containers:
  - name: memory-demo-ctr
    image: polinux/stress
    resources:
      requests:
        memory: "100Mi"
      limits:
        memory: "200Mi"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "250M", "--vm-hang", "1"]
```

**What happens:**
```bash
kubectl apply -f memory-demo.yaml

# Watch the pod - it will be OOMKilled!
kubectl get pod memory-demo --watch

# Check events
kubectl describe pod memory-demo
```

**Output:**
```
Last State:     Terminated
  Reason:       OOMKilled
  Exit Code:    137
```

**Fix it:**
```yaml
resources:
  limits:
    memory: "300Mi"  # Increase limit to accommodate 250M + overhead
```

---

## LimitRange - Setting Defaults

LimitRange automatically sets default resource values for pods that don't specify them.

### Example 1: CPU LimitRange

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-limit-range
  namespace: development
spec:
  limits:
  - default:          # Default limits
      cpu: "1"
    defaultRequest:   # Default requests
      cpu: "500m"
    max:              # Maximum allowed
      cpu: "2"
    min:              # Minimum required
      cpu: "100m"
    type: Container
```

**Apply it:**
```bash
kubectl create namespace development
kubectl apply -f cpu-limit-range.yaml
```

**Test it:**
```yaml
# Pod without resource specifications
apiVersion: v1
kind: Pod
metadata:
  name: test-defaults
  namespace: development
spec:
  containers:
  - name: nginx
    image: nginx
    # No resources specified!
```

**Check what happened:**
```bash
kubectl apply -f test-defaults.yaml
kubectl get pod test-defaults -n development -o yaml | grep -A 10 resources
```

**Output:**
```yaml
resources:
  limits:
    cpu: "1"        # ← Automatically added!
  requests:
    cpu: 500m       # ← Automatically added!
```

---

### Example 2: Memory LimitRange

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: memory-limit-range
  namespace: development
spec:
  limits:
  - default:
      memory: "512Mi"
    defaultRequest:
      memory: "256Mi"
    max:
      memory: "1Gi"
    min:
      memory: "128Mi"
    type: Container
```

---

### Example 3: Combined CPU and Memory LimitRange

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: combined-limit-range
  namespace: production
spec:
  limits:
  # Container limits
  - type: Container
    max:
      cpu: "2"
      memory: "2Gi"
    min:
      cpu: "100m"
      memory: "128Mi"
    default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "250m"
      memory: "256Mi"
  
  # Pod limits (sum of all containers)
  - type: Pod
    max:
      cpu: "4"
      memory: "4Gi"
```

**Test violation:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: too-big
  namespace: production
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        cpu: "3"      # ❌ Exceeds container max of 2
```

**Result:**
```
Error: Pod "too-big" is invalid: spec.containers[0].resources.requests: 
Invalid value: "3": must be less than or equal to cpu limit
```

---

### Example 4: Testing LimitRange Enforcement

```bash
# Create namespace and apply LimitRange
kubectl create namespace limit-test
kubectl apply -f - <<EOF
apiVersion: v1
kind: LimitRange
metadata:
  name: test-limits
  namespace: limit-test
spec:
  limits:
  - max:
      cpu: "1"
      memory: "1Gi"
    min:
      cpu: "50m"
      memory: "64Mi"
    type: Container
EOF

# Try to create pod that violates limits
kubectl run test --image=nginx --namespace=limit-test \
  --requests='cpu=2'

# Expected error:
# Error: CPU request exceeds maximum allowed: 2 > 1
```

---

## ResourceQuota - Namespace-Level Limits

ResourceQuota limits the aggregate resource consumption in a namespace.

### Example 1: Basic ResourceQuota

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: development
spec:
  hard:
    requests.cpu: "10"        # Total CPU requests
    requests.memory: "20Gi"   # Total memory requests
    limits.cpu: "20"          # Total CPU limits
    limits.memory: "40Gi"     # Total memory limits
    pods: "50"                # Maximum number of pods
```

**Apply and test:**
```bash
kubectl apply -f compute-quota.yaml

# Check quota status
kubectl get resourcequota compute-quota -n development
kubectl describe resourcequota compute-quota -n development
```

**Output:**
```
Name:            compute-quota
Namespace:       development
Resource         Used   Hard
--------         ----   ----
limits.cpu       0      20
limits.memory    0      40Gi
pods             0      50
requests.cpu     0      10
requests.memory  0      20Gi
```

---

### Example 2: Comprehensive ResourceQuota

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: comprehensive-quota
  namespace: production
spec:
  hard:
    # Compute resources
    requests.cpu: "50"
    requests.memory: "100Gi"
    limits.cpu: "100"
    limits.memory: "200Gi"
    
    # Storage
    requests.storage: "500Gi"
    persistentvolumeclaims: "20"
    
    # Objects
    pods: "100"
    services: "50"
    configmaps: "50"
    secrets: "50"
    replicationcontrollers: "20"
    
    # Services
    services.loadbalancers: "5"
    services.nodeports: "10"
```

---

### Example 3: Testing ResourceQuota

**Create pods until quota is exhausted:**
```bash
# Create namespace with quota
kubectl create namespace quota-test
kubectl apply -f - <<EOF
apiVersion: v1
kind: ResourceQuota
metadata:
  name: test-quota
  namespace: quota-test
spec:
  hard:
    requests.cpu: "2"
    requests.memory: "2Gi"
    pods: "3"
EOF

# Create first pod (should succeed)
kubectl run pod1 --image=nginx --namespace=quota-test \
  --requests='cpu=500m,memory=512Mi'

# Create second pod (should succeed)
kubectl run pod2 --image=nginx --namespace=quota-test \
  --requests='cpu=500m,memory=512Mi'

# Create third pod (should succeed)
kubectl run pod3 --image=nginx --namespace=quota-test \
  --requests='cpu=500m,memory=512Mi'

# Create fourth pod (should FAIL - exceeds pod count)
kubectl run pod4 --image=nginx --namespace=quota-test \
  --requests='cpu=500m,memory=512Mi'

# Error: exceeded quota: test-quota, requested: pods=1, 
# used: pods=3, limited: pods=3
```

---

### Example 4: Quota Per Priority Class

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: priority-quota
  namespace: production
spec:
  hard:
    pods: "10"
  scopeSelector:
    matchExpressions:
    - operator: In
      scopeName: PriorityClass
      values: ["high-priority"]
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: best-effort-quota
  namespace: production
spec:
  hard:
    pods: "50"
  scopes:
  - BestEffort  # Only applies to pods with no resource requests/limits
```

---

## QoS Classes

Kubernetes assigns a Quality of Service (QoS) class to each pod based on resource specifications.

### The Three QoS Classes

#### 1. Guaranteed (Highest Priority)

**Requirements:**
- Every container has CPU and memory limits
- Every container has CPU and memory requests
- Limits == Requests for both CPU and memory

**Example:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: guaranteed-pod
spec:
  containers:
  - name: app
    image: nginx
    resources:
      limits:
        cpu: "500m"
        memory: "512Mi"
      requests:
        cpu: "500m"
        memory: "512Mi"
```

**Characteristics:**
- ✅ Highest priority during eviction
- ✅ Last to be evicted
- ✅ Most predictable performance
- ✅ Best for production workloads

#### 2. Burstable (Medium Priority)

**Requirements:**
- At least one container has CPU or memory request/limit
- Doesn't meet Guaranteed criteria

**Example:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: burstable-pod
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        cpu: "250m"
        memory: "256Mi"
      limits:
        cpu: "500m"
        memory: "512Mi"
```

**Characteristics:**
- ⚠️ Medium priority during eviction
- ⚠️ Evicted before Guaranteed, after BestEffort
- ✅ Good balance between flexibility and stability
- ✅ Recommended for most workloads

#### 3. BestEffort (Lowest Priority)

**Requirements:**
- No containers have any requests or limits

**Example:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: besteffort-pod
spec:
  containers:
  - name: app
    image: nginx
    # No resources specified!
```

**Characteristics:**
- ❌ Lowest priority during eviction
- ❌ First to be evicted
- ❌ Unpredictable performance
- ❌ Only use for non-critical workloads

### QoS Class Comparison Table

| QoS Class | Eviction Priority | Use Case | Resource Guarantee |
|-----------|------------------|----------|-------------------|
| **Guaranteed** | 1 (Last) | Production apps, databases | ✅ Full |
| **Burstable** | 2 (Middle) | Most applications | ⚠️ Partial |
| **BestEffort** | 3 (First) | Batch jobs, dev/test | ❌ None |

### Check QoS Class

```bash
kubectl get pod <pod-name> -o jsonpath='{.status.qosClass}'

# Or use describe
kubectl describe pod <pod-name> | grep "QoS Class"
```

---

## Pod-Level Resources (New!)

**Feature State:** Beta in Kubernetes v1.34+

Pod-level resources allow setting resource budgets at the pod level rather than per-container.

### Example 1: Basic Pod-Level Resources

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-level-demo
spec:
  resources:
    requests:
      cpu: "1"
      memory: "1Gi"
    limits:
      cpu: "2"
      memory: "2Gi"
  containers:
  - name: app
    image: nginx
  - name: sidecar
    image: fluentd
    # Containers share pod resources!
```

**Benefits:**
- ✅ Easier management with many containers
- ✅ Containers can share idle resources
- ✅ Better resource utilization

---

### Example 2: Mixed Pod and Container Resources

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mixed-resources
spec:
  resources:
    requests:
      cpu: "2"
      memory: "2Gi"
    limits:
      cpu: "4"
      memory: "4Gi"
  containers:
  - name: main-app
    image: myapp
    resources:
      requests:
        cpu: "1"      # Explicit container request
        memory: "1Gi"
      limits:
        cpu: "2"
        memory: "2Gi"
  
  - name: sidecar
    image: logging
    # No explicit resources - shares pod budget
```

**Note:** Enable with feature gate `PodLevelResources`

---

## Best Practices

### 1. ✅ Always Set Requests

```yaml
# ❌ Bad
resources: {}

# ✅ Good
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
```

**Why:** Ensures predictable scheduling and prevents resource starvation.

---

### 2. ✅ Set Appropriate Limits

```yaml
# ⚠️ OK but risky
resources:
  requests:
    cpu: "500m"
  # No limit - can use unlimited CPU

# ✅ Better
resources:
  requests:
    cpu: "500m"
  limits:
    cpu: "1"
```

**Why:** Prevents runaway processes from affecting other pods.

---

### 3. ✅ Use Realistic Values

```bash
# Find actual usage
kubectl top pod <pod-name>

# Monitor over time
kubectl top pod <pod-name> --watch

# Set requests based on average + buffer
# Set limits based on peak + 20-30% buffer
```

---

### 4. ✅ Memory Limits = Requests (Usually)

```yaml
# ✅ Recommended for memory
resources:
  requests:
    memory: "512Mi"
  limits:
    memory: "512Mi"  # Same as request
```

**Why:** Prevents OOM kills and makes memory behavior predictable.

---

### 5. ✅ Use LimitRange for Defaults

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: defaults
spec:
  limits:
  - default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "250m"
      memory: "256Mi"
    type: Container
```

**Why:** Ensures all pods have sensible defaults.

---

### 6. ✅ Use ResourceQuota for Namespaces

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
spec:
  hard:
    requests.cpu: "20"
    requests.memory: "40Gi"
    pods: "100"
```

**Why:** Prevents one team from consuming all cluster resources.

---

### 7. ✅ Monitor Resource Usage

```bash
# Install metrics-server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Check node resources
kubectl top nodes

# Check pod resources
kubectl top pods --all-namespaces

# Sort by memory usage
kubectl top pods -A --sort-by=memory

# Sort by CPU usage
kubectl top pods -A --sort-by=cpu
```

---

### 8. ⚠️ Avoid These Anti-Patterns

```yaml
# ❌ No resources at all
resources: {}

# ❌ Limits without requests (sets request = limit)
resources:
  limits:
    cpu: "2"

# ❌ Unrealistic values
resources:
  requests:
    cpu: "10m"      # Too low - will be throttled
    memory: "10Mi"  # Too low - will be OOMKilled

# ❌ Huge gaps
resources:
  requests:
    cpu: "100m"
  limits:
    cpu: "10"       # 100x difference - wasteful
```

---

## Troubleshooting

### Problem 1: Pod Stuck in Pending

**Symptom:**
```bash
kubectl get pods
NAME    READY   STATUS    RESTARTS   AGE
mypod   0/1     Pending   0          2m
```

**Check events:**
```bash
kubectl describe pod mypod
```

**Common causes:**

#### Insufficient CPU
```
Events:
  Type     Reason            Message
  ----     ------            -------
  Warning  FailedScheduling  0/3 nodes available: insufficient cpu
```

**Solution:**
```bash
# Check node capacity
kubectl describe nodes | grep -A 5 "Allocated resources"

# Options:
# 1. Reduce pod requests
# 2. Add more nodes
# 3. Delete unused pods
```

#### Insufficient Memory
```
Events:
  Type     Reason            Message
  ----     ------            -------
  Warning  FailedScheduling  0/3 nodes available: insufficient memory
```

---

### Problem 2: Pod Getting OOMKilled

**Symptom:**
```bash
kubectl get pods
NAME    READY   STATUS      RESTARTS   AGE
mypod   0/1     OOMKilled   5          10m
```

**Check details:**
```bash
kubectl describe pod mypod
```

**Output:**
```
Last State:   Terminated
  Reason:     OOMKilled
  Exit Code:  137
```

**Solutions:**

#### Solution 1: Increase Memory Limit
```yaml
resources:
  limits:
    memory: "1Gi"  # Increased from 512Mi
```

#### Solution 2: Fix Memory Leak
```bash
# Check actual usage
kubectl top pod mypod

# Check logs for issues
kubectl logs mypod --previous
```

---

### Problem 3: CPU Throttling

**Symptom:** Application is slow, but CPU usage seems low.

**Check throttling:**
```bash
# On the node where pod runs
cat /sys/fs/cgroup/cpu/kubepods/pod<uid>/<container-id>/cpu.stat
```

**Look for:**
```
nr_throttled: 1000    # Number of times throttled
throttled_time: 50000 # Time throttled (in nanoseconds)
```

**Solutions:**

#### Increase CPU Limit
```yaml
resources:
  limits:
    cpu: "2"  # Increased from 1
```

#### Remove CPU Limit (Carefully!)
```yaml
resources:
  requests:
    cpu: "500m"
  # No CPU limit - can use all available
```

---

### Problem 4: ResourceQuota Exceeded

**Symptom:**
```bash
kubectl run test --image=nginx
Error: exceeded quota: compute-quota
```

**Check quota:**
```bash
kubectl get resourcequota
kubectl describe resourcequota compute-quota
```

**Output:**
```
Name:         compute-quota
Resource      Used   Hard
--------      ----   ----
requests.cpu  10     10    # ← At limit!
pods          50     50    # ← At limit!
```

**Solutions:**

#### Solution 1: Delete Unused Pods
```bash
kubectl delete pod unused-pod
```

#### Solution 2: Increase Quota
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
spec:
  hard:
    requests.cpu: "20"  # Increased
    pods: "100"         # Increased
```

---

### Problem 5: LimitRange Violation

**Symptom:**
```bash
kubectl apply -f pod.yaml
Error: Pod "mypod" is invalid: spec.containers[0].resources.requests: 
Invalid value: "4": must be less than or equal to cpu limit of 2
```

**Check LimitRange:**
```bash
kubectl get limitrange
kubectl describe limitrange cpu-limit-range
```

**Solutions:**

#### Solution 1: Adjust Pod Resources
```yaml
resources:
  requests:
    cpu: "1"  # Within limit of 2
```

#### Solution 2: Modify LimitRange
```yaml
spec:
  limits:
  - max:
      cpu: "4"  # Increased
```

---

## Quick Reference

### Common Commands

```bash
# View resource usage
kubectl top nodes
kubectl top pods
kubectl top pods --sort-by=memory
kubectl top pods --sort-by=cpu

# Describe pod resources
kubectl describe pod <pod-name> | grep -A 5 "Limits\|Requests"

# Get QoS class
kubectl get pod <pod-name> -o jsonpath='{.status.qosClass}'

# Check LimitRange
kubectl get limitrange
kubectl describe limitrange <name>

# Check ResourceQuota
kubectl get resourcequota
kubectl describe resourcequota <name>

# Check node allocatable resources
kubectl describe nodes | grep -A 5 "Allocated resources"
```

---

### Resource Calculation Examples

#### Example 1: Can this pod be scheduled?

**Node Capacity:**
```
CPU: 4 cores (4000m)
Memory: 16Gi
```

**Already Running:**
```
Pod A: requests: cpu=1000m, memory=4Gi
Pod B: requests: cpu=1500m, memory=6Gi
Total: cpu=2500m, memory=10Gi
```

**New Pod:**
```yaml
resources:
  requests:
    cpu: "2"        # 2000m
    memory: "8Gi"
```

**Calculation:**
```
Available CPU: 4000m - 2500m = 1500m
Available Memory: 16Gi - 10Gi = 6Gi

Pod needs: 2000m CPU, 8Gi memory

CPU: 2000m > 1500m ❌ NO
Memory: 8Gi > 6Gi ❌ NO

Result: Pod CANNOT be scheduled!
```

---

### Resource Units Cheat Sheet

#### CPU
```
1 CPU = 1000m
0.5 CPU = 500m
0.1 CPU = 100m
0.001 CPU = 1m (minimum)

Common values:
100m  = 0.1 CPU (light workload)
250m  = 0.25 CPU
500m  = 0.5 CPU (moderate workload)
1     = 1 CPU (heavy workload)
2     = 2 CPUs (very heavy workload)
```

#### Memory
```
Decimal (1000-based):
1K  = 1,000 bytes
1M  = 1,000,000 bytes
1G  = 1,000,000,000 bytes

Binary (1024-based):
1Ki = 1,024 bytes
1Mi = 1,048,576 bytes
1Gi = 1,073,741,824 bytes

Common values:
128Mi = 134 MB (small app)
256Mi = 268 MB
512Mi = 537 MB (medium app)
1Gi   = 1.07 GB (large app)
2Gi   = 2.15 GB (database)
```

---

### Decision Matrix: Setting Resources

| Workload Type | CPU Request | CPU Limit | Memory Request | Memory Limit |
|---------------|-------------|-----------|----------------|--------------|
| **Nginx** | 100m | 250m | 128Mi | 256Mi |
| **Node.js App** | 250m | 500m | 256Mi | 512Mi |
| **Java App** | 500m | 1 | 512Mi | 1Gi |
| **Database** | 1 | 2 | 2Gi | 4Gi |
| **Batch Job** | 100m | No limit | 256Mi | 512Mi |
| **ML Training** | 4 | 8 | 8Gi | 16Gi |

---

## Practice Exercises

### Exercise 1: Basic Resources
Create a pod with:
- CPU request: 200m, limit: 500m
- Memory request: 256Mi, limit: 512Mi

### Exercise 2: Test OOMKill
Create a pod that tries to use more memory than its limit and observe the OOMKill.

### Exercise 3: LimitRange
Create a LimitRange with sensible defaults and test it with a pod without resources.

### Exercise 4: ResourceQuota
Create a ResourceQuota, then try to exceed it by creating multiple pods.

### Exercise 5: QoS Classes
Create three pods (Guaranteed, Burstable, BestEffort) and verify their QoS classes.

### Exercise 6: Resource Calculation
Given a 4-core, 16Gi node with existing pods, determine if a new pod can be scheduled.

---

## Additional Resources

- 📚 [Official Kubernetes Resource Management](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- 📊 [LimitRange Documentation](https://kubernetes.io/docs/concepts/policy/limit-range/)
- 🎯 [ResourceQuota Documentation](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
- 🔍 [QoS Classes](https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/)

---

## Summary

**Key Takeaways:**

1. ✅ **Always set requests** - ensures predictable scheduling
2. 🎯 **Set realistic limits** - based on actual usage + buffer
3. 💾 **Memory limits = requests** - prevents OOMKills
4. 🔄 **CPU is compressible** - throttled but not killed
5. ❌ **Memory is incompressible** - OOMKilled if exceeded
6. 📊 **Use LimitRange** - for namespace-level defaults
7. 🎫 **Use ResourceQuota** - for namespace-level limits
8. 🏆 **Understand QoS classes** - affects eviction priority
9. 📈 **Monitor usage** - kubectl top and metrics-server
10. 🔧 **Test thoroughly** - before production deployment

**Remember:** Proper resource management is critical for:
- Cluster stability
- Cost optimization
- Application performance
- Fair resource allocation

---

*Happy Resource Managing! 🚀*

*Last Updated: January 2026*
