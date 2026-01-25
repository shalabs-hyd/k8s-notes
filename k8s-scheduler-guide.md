# Kubernetes Scheduler and Custom Profiles: A Comprehensive Guide for CKA Candidates

## Table of Contents
1. [Introduction to the Kubernetes Scheduler](#introduction)
2. [The Scheduling Lifecycle](#scheduling-lifecycle)
3. [Understanding Scheduler Plugins](#scheduler-plugins)
4. [Priority-Based Scheduling](#priority-scheduling)
5. [Multiple Scheduler Profiles](#scheduler-profiles)
6. [Deploying Custom Schedulers](#deploying-schedulers)
7. [Observability and Troubleshooting](#troubleshooting)
8. [Best Practices](#best-practices)
9. [CKA Exam Focus Areas](#cka-exam-focus)
10. [Hands-On Lab Exercises](#hands-on-labs)

---

## Introduction to the Kubernetes Scheduler {#introduction}

### What is the Scheduler?

The **kube-scheduler** is a critical control plane component responsible for assigning Pods to Nodes. Without a functioning scheduler, all Pods would remain in a `Pending` state indefinitely, making the cluster completely non-functional.

**Core Responsibilities:**
- Watch for newly created Pods with no assigned node
- Find the best node for each Pod based on resource requirements and constraints
- Bind the Pod to the selected node
- Handle priority and preemption logic

**Key Concept**: The scheduler doesn't actually run the Pod—it only makes the placement decision. The kubelet on the assigned node is responsible for actually starting the containers.

### Default Scheduler vs Custom Schedulers

**Default Scheduler (`default-scheduler`):**
- Comes pre-configured with Kubernetes
- Handles 95% of standard workloads
- Uses a comprehensive set of built-in plugins
- Managed as a static Pod in `/etc/kubernetes/manifests/kube-scheduler.yaml`

**Custom Schedulers:**
- Implement specialized business logic
- Required for advanced multi-tenancy scenarios
- Can run alongside the default scheduler
- Enable fine-grained control over pod placement

### Why Multiple Schedulers?

**Real-World Scenarios:**

1. **GPU Workloads**: AI/ML teams need specialized scheduling for expensive GPU nodes
2. **Multi-Tenancy**: Different teams require isolated scheduling policies
3. **Compliance**: Regulatory requirements demand specific node placement rules
4. **Performance**: High-frequency trading applications need latency-aware scheduling

---

## The Scheduling Lifecycle {#scheduling-lifecycle}

### Overview: From Pod Creation to Node Assignment

The scheduling process consists of four distinct phases:

```
Pod Created → Queueing → Filtering → Scoring → Binding → Pod Assigned
```

Let's walk through what happens when you run `kubectl create -f pod.yaml`:

**Step 1: Pod Creation**
- API server receives the Pod specification
- Pod is stored in etcd with `spec.nodeName: ""` (empty)
- Pod status: `Pending`

**Step 2: Scheduler Detects Unscheduled Pod**
- Scheduler watches API server for Pods with empty `nodeName`
- Pod enters the scheduling queue

**Step 3: Scheduling Cycle Begins**
- Pod moves through filtering, scoring, and binding phases
- Best node is selected based on constraints and optimization

**Step 4: Binding**
- Scheduler updates the Pod's `spec.nodeName` field via API server
- Pod is now "bound" to a node

**Step 5: Kubelet Takes Over**
- Kubelet on the assigned node detects the Pod
- Kubelet pulls images and starts containers
- Pod status: `Running`

### The Four Scheduling Phases Explained

#### Phase 1: Queueing (Scheduling Queue)

**Purpose**: Determine the order in which Pods are processed

**How it Works:**
- All unscheduled Pods enter a priority queue
- Pods are sorted based on their priority value (higher = first)
- Equal-priority Pods are processed in creation-time order (FIFO)

**Active Plugin:**
- **PrioritySort Plugin**: Sorts queue by `spec.priority` field

**Example:**
```
Queue State:
1. database-pod (priority: 1000000) ← Processed first
2. api-pod (priority: 500000)
3. batch-job (priority: -1000) ← Processed last
```

#### Phase 2: Filtering (Feasibility Check)

**Purpose**: Eliminate nodes that cannot run the Pod

**How it Works:**
- Scheduler evaluates each node against a set of predicates
- A node must pass ALL filters to be considered viable
- If zero nodes pass, the Pod remains `Pending`

**Key Filtering Plugins:**

| Plugin | What It Checks | Example |
|--------|---------------|---------|
| **NodeResourcesFit** | CPU/Memory availability | Pod requests 4 CPU, node has 2 CPU → FAIL |
| **NodeName** | Specific node requirement | Pod specifies `nodeName: worker-1` → Only worker-1 passes |
| **NodeUnschedulable** | Node cordoned status | `kubectl cordon worker-2` → worker-2 filtered out |
| **NodeAffinity** | Node label requirements | Pod requires `disktype=ssd` → Only SSD nodes pass |
| **TaintToleration** | Taints and tolerations | Node has `key=gpu:NoSchedule` → Only Pods with matching toleration pass |
| **PodTopologySpread** | Even distribution | Ensures Pods spread across zones |

**Example Scenario:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: database
spec:
  containers:
  - name: postgres
    image: postgres:15
    resources:
      requests:
        cpu: "4"
        memory: "8Gi"
```

**Cluster State:**
- worker-1: 8 CPU, 16Gi RAM → ✅ PASS
- worker-2: 2 CPU, 8Gi RAM → ❌ FAIL (insufficient CPU)
- worker-3: 6 CPU, 32Gi RAM, cordoned → ❌ FAIL (unschedulable)

**Result**: Only worker-1 proceeds to scoring phase

#### Phase 3: Scoring (Optimization)

**Purpose**: Rank remaining nodes to find the best fit

**How it Works:**
- Each viable node receives a score (0-100) from each scoring plugin
- Scores are weighted and summed
- Node with highest total score wins

**Key Scoring Plugins:**

| Plugin | Scoring Logic | Why It Matters |
|--------|--------------|----------------|
| **NodeResourcesFit** | Prefers balanced resource utilization | Prevents fragmentation |
| **ImageLocality** | Higher score if image already cached | Faster Pod startup |
| **InterPodAffinity** | Scores based on Pod affinity rules | Enables colocation strategies |
| **NodeAffinity** | Scores based on node label preferences | Soft constraints for placement |

**Example Scoring:**

Pod requests: 2 CPU, 4Gi RAM

| Node | Available Resources | ImageLocality | NodeResourcesFit | Total Score |
|------|-------------------|---------------|-----------------|-------------|
| worker-1 | 6 CPU, 12Gi RAM | 10 (image cached) | 75 (moderate fit) | 85 |
| worker-4 | 14 CPU, 28Gi RAM | 0 (no image) | 90 (excellent fit) | 90 ← Winner |

**Result**: worker-4 selected (highest score)

#### Phase 4: Binding (Finalization)

**Purpose**: Commit the scheduling decision to the API server

**How it Works:**
- Scheduler sends a binding request to API server
- API server updates the Pod's `spec.nodeName` field
- Pod transitions from scheduling cycle to kubelet management

**Active Plugin:**
- **DefaultBinder Plugin**: Performs the actual API call

**Critical Exam Point**: Binding is asynchronous. The scheduler doesn't wait for the kubelet to start the Pod—it immediately moves on to schedule the next Pod in the queue.

---

## Understanding Scheduler Plugins {#scheduler-plugins}

### The Plugin Architecture

Modern Kubernetes schedulers are built on a **plugin framework** that allows customization without modifying core code.

**Extension Points** (where plugins can hook into the lifecycle):

```
Queue → Filter → PreScore → Score → Reserve → Permit → PreBind → Bind → PostBind
  ↓        ↓         ↓         ↓        ↓        ↓        ↓       ↓        ↓
Plugin  Plugin    Plugin    Plugin   Plugin   Plugin   Plugin  Plugin   Plugin
```

### Built-in Plugins Reference

#### Queueing Extension Point

**PrioritySort**
- **Phase**: QueueSort
- **Purpose**: Orders Pods by priority
- **Enabled by default**: Yes
- **Configuration**: Cannot be disabled in default scheduler

#### Filtering Extension Point

**NodeResourcesFit**
- **Phase**: Filter
- **Purpose**: Checks if node has sufficient CPU/memory
- **Logic**: Compares Pod `requests` vs node allocatable resources
- **Enabled by default**: Yes

**NodeName**
- **Phase**: Filter
- **Purpose**: Filters when `spec.nodeName` is explicitly set
- **Enabled by default**: Yes
- **Exam Tip**: This bypasses the entire scheduling cycle

**NodeUnschedulable**
- **Phase**: Filter
- **Purpose**: Filters cordoned nodes
- **Check Command**: `kubectl get nodes` (look for `SchedulingDisabled`)
- **Enabled by default**: Yes

**NodeAffinity**
- **Phase**: Filter
- **Purpose**: Enforces required node selector terms
- **Enabled by default**: Yes

**TaintToleration**
- **Phase**: Filter
- **Purpose**: Filters nodes based on taints
- **Enabled by default**: Yes

#### Scoring Extension Point

**NodeResourcesFit** (Scoring Mode)
- **Phase**: Score
- **Purpose**: Ranks nodes by resource efficiency
- **Strategy**: Configurable (LeastAllocated, MostAllocated, RequestedToCapacityRatio)
- **Default**: LeastAllocated (spreads Pods evenly)

**ImageLocality**
- **Phase**: Score
- **Purpose**: Prefers nodes with images already pulled
- **Impact**: Can save 30+ seconds on Pod startup
- **Enabled by default**: Yes

**InterPodAffinity**
- **Phase**: Score
- **Purpose**: Implements Pod affinity/anti-affinity rules
- **Enabled by default**: Yes

**NodeAffinity** (Scoring Mode)
- **Phase**: Score
- **Purpose**: Scores nodes matching preferred terms
- **Enabled by default**: Yes

#### Binding Extension Point

**DefaultBinder**
- **Phase**: Bind
- **Purpose**: Performs the actual binding operation
- **Enabled by default**: Yes
- **Critical**: Without this, Pods cannot be assigned

### Plugin Configuration Example

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
- schedulerName: custom-scheduler
  plugins:
    # Disable specific plugins
    filter:
      disabled:
      - name: NodeAffinity
      - name: TaintToleration
    
    # Enable custom plugins
    score:
      enabled:
      - name: MyCustomScoringPlugin
        weight: 10
      disabled:
      - name: ImageLocality  # Don't consider image caching
```

---

## Priority-Based Scheduling {#priority-scheduling}

### How Priority Affects Scheduling

Priority influences TWO critical behaviors:

1. **Queue Ordering**: Higher priority Pods are scheduled first
2. **Preemption**: High priority Pods can evict lower priority Pods

### Implementing Priority Scheduling

**Step 1: Create a PriorityClass**

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
preemptionPolicy: PreemptLowerPriority
description: "Mission-critical production workloads"
```

**Step 2: Assign to Pod**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: critical-database
spec:
  priorityClassName: high-priority
  containers:
  - name: postgres
    image: postgres:15
    resources:
      requests:
        cpu: "10"
        memory: "16Gi"
```

**Step 3: Verify Priority Assignment**

```bash
kubectl get pod critical-database -o yaml | grep priority
```

**Output:**
```yaml
priority: 1000000
priorityClassName: high-priority
```

### Resource Requests and Filtering

**Critical Concept**: Priority does NOT bypass resource requirements.

**Example:**

```yaml
resources:
  requests:
    cpu: "10"  # Pod MUST find a node with 10 available CPUs
    memory: "1Gi"
```

**What Happens:**
1. Pod enters queue with priority 1000000 (at front of queue)
2. Scheduler filters nodes using NodeResourcesFit plugin
3. Only nodes with ≥10 CPUs pass filtering
4. If NO nodes have 10 CPUs, Pod remains `Pending` despite high priority
5. If preemption is enabled, scheduler may evict lower-priority Pods to free resources

**Key Exam Point**: A high-priority Pod with `requests.cpu: 100` will still be `Pending` if the largest node only has 64 CPUs. Priority cannot create resources that don't exist.

### Priority vs Preemption

**Priority without Preemption:**
```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority-no-preempt
value: 1000000
preemptionPolicy: Never  # Will NOT evict other Pods
```

**Behavior:**
- Pod goes to front of queue
- Will NOT kick off running Pods
- Waits for resources to free naturally

**Use Case**: Batch jobs that need priority but shouldn't disrupt production

---

## Multiple Scheduler Profiles {#scheduler-profiles}

### Why Multiple Profiles?

**The Old Way (Pre-1.18):**
- Run completely separate scheduler binaries
- Each scheduler manages its own queue
- **Problem**: Race conditions when multiple schedulers try to bind different Pods to the same node resources

**The Modern Way (1.18+):**
- Single scheduler binary with multiple profiles
- Shared resource awareness
- No race conditions
- Lower operational overhead

### Creating Scheduler Profiles

**Configuration File: KubeSchedulerConfiguration**

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  # Profile 1: Default scheduler (unchanged)
  - schedulerName: default-scheduler
  
  # Profile 2: Custom scheduler for GPU workloads
  - schedulerName: gpu-scheduler
    plugins:
      filter:
        enabled:
        - name: NodeResourcesFit
      score:
        enabled:
        - name: NodeResourcesFit
          weight: 5
        - name: ImageLocality
          weight: 3
        disabled:
        - name: InterPodAffinity  # Don't care about Pod colocation
  
  # Profile 3: Fast scheduler (minimal plugins for speed)
  - schedulerName: fast-scheduler
    plugins:
      preScore:
        disabled:
        - name: '*'  # Disable all preScore plugins
      score:
        disabled:
        - name: '*'  # Disable all scoring (use first viable node)
```

### Profile Configuration Options

#### Disabling Plugins

**Disable specific plugin:**
```yaml
plugins:
  filter:
    disabled:
    - name: TaintToleration
```

**Disable all plugins in a phase:**
```yaml
plugins:
  score:
    disabled:
    - name: '*'
```

#### Enabling Custom Plugins

```yaml
plugins:
  score:
    enabled:
    - name: MyCustomPlugin
      weight: 10
```

#### Plugin Weights

Adjust the importance of scoring plugins:

```yaml
plugins:
  score:
    enabled:
    - name: NodeResourcesFit
      weight: 5  # 5x more important than weight-1 plugins
    - name: ImageLocality
      weight: 1
```

**Score Calculation:**
```
Total Score = (NodeResourcesFit × 5) + (ImageLocality × 1)
```

### Using Custom Scheduler Profiles

**Assign a Pod to a specific scheduler profile:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-workload
spec:
  schedulerName: gpu-scheduler  # Use the gpu-scheduler profile
  containers:
  - name: tensorflow
    image: tensorflow/tensorflow:latest-gpu
    resources:
      limits:
        nvidia.com/gpu: 1
```

**Default Behavior:**
- If `schedulerName` is omitted, the default-scheduler profile is used
- Profile name MUST match exactly (case-sensitive)

---

## Deploying Custom Schedulers {#deploying-schedulers}

### Deployment Methods Overview

| Method | Use Case | Complexity | HA Support |
|--------|----------|------------|-----------|
| **Static Pod** | CKA exam, kubeadm clusters | Low | No |
| **Binary/Service** | Kubernetes the Hard Way | Medium | Manual |
| **Deployment** | Production, multi-replica | High | Yes |

### Method 1: Static Pod Deployment (CKA Exam Focus)

**Why Static Pods?**
- Managed by kubelet (not the API server)
- Automatically restarted if they crash
- Standard approach for control plane components
- No need for Deployments or ReplicaSets

#### Step-by-Step Implementation

**Step 1: Create KubeSchedulerConfiguration**

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
- schedulerName: my-custom-scheduler
  plugins:
    score:
      disabled:
      - name: TaintToleration
leaderElection:
  leaderElect: false  # Single instance, no HA
```

Save as `/etc/kubernetes/my-scheduler-config.yaml`

**Step 2: Create Static Pod Manifest**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-custom-scheduler
  namespace: kube-system
spec:
  containers:
  - name: kube-scheduler
    image: registry.k8s.io/kube-scheduler:v1.28.0
    command:
    - kube-scheduler
    - --config=/etc/kubernetes/my-scheduler-config.yaml
    - --v=2
    volumeMounts:
    - name: config-volume
      mountPath: /etc/kubernetes
      readOnly: true
  volumes:
  - name: config-volume
    hostPath:
      path: /etc/kubernetes
      type: Directory
  hostNetwork: true
```

Save as `/etc/kubernetes/manifests/my-custom-scheduler.yaml`

**Step 3: Verify Deployment**

```bash
# Check if scheduler Pod is running
kubectl get pods -n kube-system | grep my-custom-scheduler

# Check scheduler logs
kubectl logs -n kube-system my-custom-scheduler

# Look for successful startup
# Expected: "Successfully acquired lease kube-system/my-custom-scheduler"
```

**Step 4: Test with a Pod**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  schedulerName: my-custom-scheduler
  containers:
  - name: nginx
    image: nginx
```

```bash
kubectl apply -f test-pod.yaml

# Verify which scheduler handled it
kubectl get events | grep test-pod | grep Scheduled
```

**Expected Output:**
```
Normal  Scheduled  pod/test-pod  my-custom-scheduler  Successfully assigned default/test-pod to worker-1
```

### Method 2: Deployment with RBAC (Production)

#### Step 1: Create ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-scheduler
  namespace: kube-system
```

#### Step 2: Create RBAC Bindings

The scheduler needs extensive permissions to function:

```yaml
---
# Binding 1: Core scheduler permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: my-scheduler-as-kube-scheduler
subjects:
- kind: ServiceAccount
  name: my-scheduler
  namespace: kube-system
roleRef:
  kind: ClusterRole
  name: system:kube-scheduler
  apiGroup: rbac.authorization.k8s.io

---
# Binding 2: Volume scheduling permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: my-scheduler-as-volume-scheduler
subjects:
- kind: ServiceAccount
  name: my-scheduler
  namespace: kube-system
roleRef:
  kind: ClusterRole
  name: system:volume-scheduler
  apiGroup: rbac.authorization.k8s.io
```

**What these roles provide:**

**system:kube-scheduler:**
- Get/List/Watch Pods
- Bind Pods to Nodes
- Update Pod status
- Create events
- Manage leases (for leader election)

**system:volume-scheduler:**
- Get/List/Watch PersistentVolumeClaims
- Update PV/PVC bindings

#### Step 3: Create ConfigMap for Configuration

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-scheduler-config
  namespace: kube-system
data:
  scheduler-config.yaml: |
    apiVersion: kubescheduler.config.k8s.io/v1
    kind: KubeSchedulerConfiguration
    profiles:
    - schedulerName: my-custom-scheduler
      plugins:
        score:
          disabled:
          - name: ImageLocality
    leaderElection:
      leaderElect: true  # Enable for multi-replica HA
      resourceNamespace: kube-system
      resourceName: my-custom-scheduler
```

#### Step 4: Create Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-custom-scheduler
  namespace: kube-system
spec:
  replicas: 2  # High availability
  selector:
    matchLabels:
      component: my-scheduler
  template:
    metadata:
      labels:
        component: my-scheduler
    spec:
      serviceAccountName: my-scheduler
      containers:
      - name: kube-scheduler
        image: registry.k8s.io/kube-scheduler:v1.28.0
        command:
        - kube-scheduler
        - --config=/etc/kubernetes/scheduler-config.yaml
        - --v=2
        volumeMounts:
        - name: config-volume
          mountPath: /etc/kubernetes
          readOnly: true
      volumes:
      - name: config-volume
        configMap:
          name: my-scheduler-config
```

#### Step 5: Verify Leader Election

```bash
# Check which replica is the leader
kubectl get lease -n kube-system my-custom-scheduler -o yaml

# Look for holderIdentity field
# Only the leader actively schedules Pods
```

**Leader Election Output:**
```yaml
spec:
  holderIdentity: my-custom-scheduler-7d4f9c8b5-xk2p9  # Current leader
  leaseDurationSeconds: 15
  renewTime: "2024-01-25T10:30:45.123456Z"
```

### Method 3: Binary Deployment (Advanced)

**Use Case**: Custom built schedulers, non-kubeadm clusters

**Steps:**

1. **Download Scheduler Binary**
```bash
wget https://storage.googleapis.com/kubernetes-release/release/v1.28.0/bin/linux/amd64/kube-scheduler
chmod +x kube-scheduler
sudo mv kube-scheduler /usr/local/bin/
```

2. **Create Systemd Service**
```ini
[Unit]
Description=Kubernetes Scheduler
Documentation=https://kubernetes.io/docs/

[Service]
ExecStart=/usr/local/bin/kube-scheduler \
  --config=/etc/kubernetes/scheduler-config.yaml \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

3. **Start Service**
```bash
sudo systemctl daemon-reload
sudo systemctl enable kube-scheduler
sudo systemctl start kube-scheduler
sudo systemctl status kube-scheduler
```

---

## Observability and Troubleshooting {#troubleshooting}

### Verifying Scheduler Assignment

#### Method 1: Events

```bash
# View all events
kubectl get events -o wide

# Filter for scheduling events
kubectl get events --field-selector involvedObject.name=<pod-name>

# Look for Scheduled event
kubectl get events | grep Scheduled
```

**Example Output:**
```
LAST SEEN   TYPE     REASON      OBJECT        SOURCE                MESSAGE
2m          Normal   Scheduled   pod/nginx     my-custom-scheduler   Successfully assigned default/nginx to worker-1
```

**Key Fields:**
- **REASON**: `Scheduled` indicates successful placement
- **SOURCE**: Shows which scheduler made the decision
- **MESSAGE**: Shows the node assignment

#### Method 2: Pod Spec

```bash
kubectl get pod <pod-name> -o yaml | grep schedulerName
```

**Output:**
```yaml
schedulerName: my-custom-scheduler
```

#### Method 3: Describe Pod

```bash
kubectl describe pod <pod-name>
```

**Look for Events section:**
```
Events:
  Type    Reason     Age   From                  Message
  ----    ------     ----  ----                  -------
  Normal  Scheduled  30s   my-custom-scheduler   Successfully assigned default/nginx to worker-1
```

### Common Issues and Solutions

#### Issue 1: Pod Stuck in Pending

**Symptoms:**
```bash
kubectl get pods
NAME      READY   STATUS    RESTARTS   AGE
my-pod    0/1     Pending   0          5m
```

**Diagnosis:**

```bash
kubectl describe pod my-pod
```

**Look for:**

**Scenario A: Scheduler Not Found**
```
Events:
  Warning  FailedScheduling  5m  default-scheduler  0/3 nodes are available: 
  3 Insufficient cpu.
```

**Cause**: Wrong `schedulerName` or scheduler not running

**Solution:**
```bash
# Check if scheduler Pod exists
kubectl get pods -n kube-system | grep scheduler

# Check scheduler name spelling
kubectl get pod my-pod -o jsonpath='{.spec.schedulerName}'

# Fix scheduler name
kubectl delete pod my-pod
# Edit YAML and reapply
```

**Scenario B: No Suitable Nodes**
```
Events:
  Warning  FailedScheduling  my-custom-scheduler  0/3 nodes available: 
  insufficient cpu (3), node(s) had taint that the pod didn't tolerate (2).
```

**Cause**: Resource constraints or taints

**Solution:**
```bash
# Check node resources
kubectl describe nodes

# Check Pod resource requests
kubectl get pod my-pod -o yaml | grep -A 5 resources

# Reduce requests or add nodes
```

**Scenario C: Scheduler Configuration Error**
```
Events:
  Warning  FailedScheduling  No events (scheduler may not be running)
```

**Solution:**
```bash
# Check scheduler logs
kubectl logs -n kube-system my-custom-scheduler

# Look for configuration errors
```

#### Issue 2: Scheduler Pod Not Starting

**Check scheduler status:**
```bash
kubectl get pods -n kube-system | grep scheduler
kubectl describe pod -n kube-system my-custom-scheduler
```

**Common Causes:**

**Cause A: ConfigMap Not Mounted**

**Error in logs:**
```
Error: failed to read configuration: open /etc/kubernetes/scheduler-config.yaml: no such file or directory
```

**Solution:**
```yaml
# Verify volume mount in scheduler Pod
volumeMounts:
- name: config-volume
  mountPath: /etc/kubernetes
volumes:
- name: config-volume
  configMap:
    name: my-scheduler-config  # Must match ConfigMap name
```

**Cause B: Invalid Configuration**

**Error in logs:**
```
Error: failed to decode configuration: error unmarshaling JSON: unknown field "schedulerNam"
```

**Solution:**
- Fix typos in KubeSchedulerConfiguration
- Validate YAML syntax
- Check API version matches cluster version

**Cause C: RBAC Permissions Missing**

**Error in logs:**
```
Failed to acquire lease: pods is forbidden: User "system:serviceaccount:kube-system:my-scheduler" 
cannot create resource "leases" in API group "coordination.k8s.io"
```

**Solution:**
```bash
# Verify ServiceAccount
kubectl get sa -n kube-system my-scheduler

# Verify ClusterRoleBindings
kubectl get clusterrolebinding | grep my-scheduler

# Create missing RBAC (see Deployment section)
```

#### Issue 3: Multiple Schedulers Conflict

**Symptoms:**
- Pods scheduled by wrong scheduler
- Race conditions in bindings

**Diagnosis:**
```bash
# List all scheduler Pods
kubectl get pods -n kube-system | grep scheduler

# Check each scheduler's configuration
kubectl get pod -n kube-system <scheduler-pod> -o yaml | grep schedulerName
```

**Solution:**
- Ensure each scheduler has a unique `schedulerName`
- Verify Pods specify the correct `schedulerName`
- Use scheduler profiles instead of separate binaries

#### Issue 4: Leader Election Failures

**Symptoms:**
- Multiple replicas all trying to schedule
- Binding conflicts

**Check leader:**
```bash
kubectl get lease -n kube-system my-custom-scheduler -o yaml
```

**Expected:**
```yaml
spec:
  holderIdentity: my-custom-scheduler-abc123  # One replica should be leader
  leaseDurationSeconds: 15
```

**If no leader:**
```bash
# Check scheduler logs
kubectl logs -n kube-system my-custom-scheduler-abc123

# Look for:
# "successfully acquired lease"  ← Good
# "failed to acquire lease"      ← Problem
```

**Solution:**
- Verify `leaderElection.leaderElect: true` in config
- Check RBAC permissions for lease management
- Ensure only one scheduler has a specific name

### Debugging Commands Reference

```bash
# Check scheduler is running
kubectl get pods -n kube-system -l component=kube-scheduler

# View scheduler logs
kubectl logs -n kube-system <scheduler-pod-name>

# Follow logs in real-time
kubectl logs -n kube-system <scheduler-pod-name> -f

# Check scheduler configuration
kubectl get cm -n kube-system my-scheduler-config -o yaml

# List all schedulers in cluster
kubectl get pods -n kube-system -o custom-columns=NAME:.metadata.name,SCHEDULER:.spec.schedulerName | grep scheduler

# Check which Pods use custom scheduler
kubectl get pods --all-namespaces -o custom-columns=NAMESPACE:.metadata.namespace,NAME:.metadata.name,SCHEDULER:.spec.schedulerName | grep -v default-scheduler

# Validate RBAC
kubectl auth can-i create leases --as=system:serviceaccount:kube-system:my-scheduler -n kube-system

# Check events for scheduling failures
kubectl get events --all-namespaces --field-selector reason=FailedScheduling
```

---

## Best Practices {#best-practices}

### Production Scheduler Guidelines

**1. Use Profiles Over Separate Binaries**

❌ **Old approach:**
```
kube-scheduler-default (separate binary)
kube-scheduler-gpu (separate binary)
kube-scheduler-batch (separate binary)
```

✅ **Modern approach:**
```yaml
# Single scheduler with multiple profiles
profiles:
- schedulerName: default-scheduler
- schedulerName: gpu-scheduler
- schedulerName: batch-scheduler
```

**Benefits:**
- No race conditions
- Lower resource overhead
- Easier to manage

**2. Standardize Naming Conventions**

```yaml
# Good naming pattern
profiles:
- schedulerName: default-scheduler
- schedulerName: gpu-scheduler
- schedulerName: ml-training-scheduler
- schedulerName: batch-scheduler

# Bad naming (confusing)
profiles:
- schedulerName: scheduler1
- schedulerName: my-sched
- schedulerName: custom
```

**3. Minimize Plugin Overhead**

```yaml
# Only enable plugins you actually need
profiles:
- schedulerName: fast-scheduler
  plugins:
    preScore:
      disabled:
      - name: '*'  # Disable all preScore for speed
    score:
      enabled:
      - name: NodeResourcesFit  # Only essential plugins
      disabled:
      - name: '*'
```

**Performance Impact:**
- Each plugin adds latency to scheduling decisions
- In large clusters (1000+ nodes), excessive plugins can delay Pod startup by seconds
- Benchmark: Default scheduler with all plugins ~50ms, minimal plugins ~10ms per Pod

**4. Configure Leader Election Correctly**

**Single replica (development/testing):**
```yaml
leaderElection:
  leaderElect: false  # No leader election needed
```

**Multiple replicas (production HA):**
```yaml
leaderElection:
  leaderElect: true
  leaseDuration: 15s
  renewDeadline: 10s
  retryPeriod: 2s
  resourceNamespace: kube-system
  resourceName: my-custom-scheduler
```

**5. Always Include Resource Requests**

Pods without resource requests can cause scheduling inefficiencies:

❌ **Bad:**
```yaml
spec:
  containers:
  - name: app
    image: myapp
    # No resource requests
```

✅ **Good:**
```yaml
spec:
  containers:
  - name: app
    image: myapp
    resources:
      requests:
        cpu: "500m"
        memory: "512Mi"
      limits:
        cpu: "1000m"
        memory: "1Gi"
```

**6. Monitor Scheduler Performance**

```bash
# Check scheduling latency
kubectl get --raw /metrics | grep scheduler_scheduling_duration_seconds

# Monitor pending Pods
kubectl get pods --all-namespaces --field-selector=status.phase=Pending

# Set up alerts for sustained pending Pods
```

**7. Version Alignment**

**Critical**: Scheduler version must match API server version

```bash
# Check versions
kubectl version

# Scheduler binary should match
kubectl get pod -n kube-system kube-scheduler-controlplane -o yaml | grep image:
```

**8. Document Custom Schedulers**

Create clear documentation for your team:

```yaml
# Example documentation in scheduler ConfigMap
metadata:
  annotations:
    description: "GPU scheduler for ML workloads"
    owner: "ml-platform-team@company.com"
    usage: |
      Use this scheduler for Pods that require GPU resources.
      Specify in Pod: schedulerName: gpu-scheduler
      Requires: nvidia.com/gpu resource limits
```

---

## CKA Exam Focus Areas {#cka-exam-focus}

### What You Must Know

**1. Scheduling Phases (High Priority)**

Memorize the four phases:
1. **Queueing** - PrioritySort plugin
2. **Filtering** - NodeResourcesFit, NodeName, NodeUnschedulable
3. **Scoring** - NodeResourcesFit, ImageLocality
4. **Binding** - DefaultBinder

**Expected Question Pattern:**
"Explain what happens when a Pod is created and how it gets assigned to a node."

**2. Static Pod Deployment (Critical for Exam)**

You must be able to create a custom scheduler as a static Pod from memory:

**Key locations:**
- Configuration file: `/etc/kubernetes/scheduler-config.yaml`
- Static Pod manifest: `/etc/kubernetes/manifests/my-scheduler.yaml`

**Template to memorize:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-scheduler
  namespace: kube-system
spec:
  containers:
  - name: kube-scheduler
    image: registry.k8s.io/kube-scheduler:v1.28.0
    command:
    - kube-scheduler
    - --config=/etc/kubernetes/my-scheduler-config.yaml
    volumeMounts:
    - name: config
      mountPath: /etc/kubernetes
  volumes:
  - name: config
    hostPath:
      path: /etc/kubernetes
  hostNetwork: true
```

**3. KubeSchedulerConfiguration (Medium Priority)**

Basic structure to know:

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
- schedulerName: my-scheduler
  plugins:
    score:
      disabled:
      - name: TaintToleration
leaderElection:
  leaderElect: false
```

**4. Assigning Pods to Custom Scheduler (High Priority)**

Simple but critical:

```yaml
spec:
  schedulerName: my-custom-scheduler  # Just add this line
```

**Common mistake**: Spelling errors - `schedulerName` not `schedulername`

**5. Verification Commands (High Priority)**

Must know these by heart:

```bash
# Check which scheduler assigned a Pod
kubectl get events -o wide | grep <pod-name>

# View scheduler name in Pod
kubectl get pod <pod-name> -o yaml | grep schedulerName

# List all scheduler Pods
kubectl get pods -n kube-system | grep scheduler

# Check scheduler logs
kubectl logs -n kube-system <scheduler-pod>
```

**6. Common Troubleshooting (Medium Priority)**

**Issue**: Pod stuck in Pending
**First check**: `kubectl describe pod <pod-name>`
**Look for**: Events section showing why scheduling failed

**Issue**: Custom scheduler not working
**First check**: `kubectl get pods -n kube-system | grep scheduler`
**Second check**: `kubectl logs -n kube-system <scheduler-pod>`

### Exam Time-Savers

**Tip 1: Use kubectl explain**
```bash
kubectl explain pod.spec.schedulerName
kubectl explain kubeschedulerconfiguration
```

**Tip 2: Copy from existing scheduler**
```bash
# Get default scheduler config
kubectl get pod -n kube-system kube-scheduler-controlplane -o yaml > scheduler.yaml

# Modify as needed
vim scheduler.yaml
```

**Tip 3: Quick validation**
```bash
# Dry-run to validate YAML
kubectl apply -f scheduler.yaml --dry-run=client

# Check if scheduler Pod started
kubectl get pods -n kube-system -w
```

**Tip 4: Remember the source column**
```bash
kubectl get events -o wide
# SOURCE column shows which scheduler was used
```

### Common Exam Question Patterns

**Pattern 1: "Create a custom scheduler named X"**
- Create KubeSchedulerConfiguration
- Create static Pod manifest
- Verify it's running

**Pattern 2: "Configure this Pod to use scheduler Y"**
- Add `schedulerName: Y` to Pod spec
- Apply and verify with events

**Pattern 3: "Why is this Pod not scheduling?"**
- Check if scheduler exists
- Check schedulerName spelling
- Review describe output for errors

**Pattern 4: "Disable plugin X in scheduler Y"**
- Edit KubeSchedulerConfiguration
- Add plugin to disabled list
- Restart scheduler Pod

---

## Hands-On Lab Exercises {#hands-on-labs}

### Lab 1: Creating Your First Custom Scheduler

**Objective:** Deploy a basic custom scheduler as a static Pod

**Prerequisites:** Access to a control plane node

**Tasks:**

1. Create a custom scheduler named `my-first-scheduler`
2. Deploy it as a static Pod
3. Test with a sample Pod
4. Verify using events

**Solution:**

```bash
# Step 1: SSH to control plane node
ssh controlplane

# Step 2: Create scheduler configuration
sudo cat > /etc/kubernetes/my-scheduler-config.yaml <<EOF
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
- schedulerName: my-first-scheduler
leaderElection:
  leaderElect: false
EOF

# Step 3: Create static Pod manifest
sudo cat > /etc/kubernetes/manifests/my-first-scheduler.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: my-first-scheduler
  namespace: kube-system
spec:
  containers:
  - name: kube-scheduler
    image: registry.k8s.io/kube-scheduler:v1.28.0
    command:
    - kube-scheduler
    - --config=/etc/kubernetes/my-scheduler-config.yaml
    - --v=2
    volumeMounts:
    - name: config
      mountPath: /etc/kubernetes
      readOnly: true
  volumes:
  - name: config
    hostPath:
      path: /etc/kubernetes
  hostNetwork: true
EOF

# Step 4: Wait for scheduler to start (auto-managed by kubelet)
sleep 10
kubectl get pods -n kube-system | grep my-first-scheduler

# Step 5: Create test Pod
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-custom-scheduler
spec:
  schedulerName: my-first-scheduler
  containers:
  - name: nginx
    image: nginx
EOF

# Step 6: Verify
kubectl get events -o wide | grep test-custom-scheduler | grep Scheduled

# Expected output should show "my-first-scheduler" in SOURCE column
```

### Lab 2: Multiple Scheduler Profiles

**Objective:** Configure a single scheduler with multiple profiles

**Scenario:** Create profiles for production, development, and batch workloads

**Solution:**

```yaml
# Create advanced configuration
cat > /etc/kubernetes/multi-profile-config.yaml <<EOF
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  # Production scheduler - balanced scoring
  - schedulerName: production-scheduler
    plugins:
      score:
        enabled:
        - name: NodeResourcesFit
          weight: 5
        - name: ImageLocality
          weight: 3
  
  # Development scheduler - prioritize image locality
  - schedulerName: dev-scheduler
    plugins:
      score:
        enabled:
        - name: ImageLocality
          weight: 10
        - name: NodeResourcesFit
          weight: 1
  
  # Batch scheduler - minimal scoring for speed
  - schedulerName: batch-scheduler
    plugins:
      preScore:
        disabled:
        - name: '*'
      score:
        disabled:
        - name: '*'
leaderElection:
  leaderElect: false
EOF

# Update scheduler Pod to use new config
# Edit /etc/kubernetes/manifests/kube-scheduler.yaml
# Change --config path to /etc/kubernetes/multi-profile-config.yaml

# Test each profile
kubectl run prod-pod --image=nginx --overrides='{"spec":{"schedulerName":"production-scheduler"}}'
kubectl run dev-pod --image=nginx --overrides='{"spec":{"schedulerName":"dev-scheduler"}}'
kubectl run batch-pod --image=busybox --command sleep 3600 --overrides='{"spec":{"schedulerName":"batch-scheduler"}}'

# Verify
kubectl get events -o custom-columns=NAME:.involvedObject.name,SCHEDULER:.source.component | grep -E "prod-pod|dev-pod|batch-pod"
```

### Lab 3: Troubleshooting Scheduler Issues

**Objective:** Practice diagnosing common scheduler problems

**Scenario 1: Typo in scheduler name**

```bash
# Create Pod with typo
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: broken-pod
spec:
  schedulerName: my-first-schedulr  # Typo: missing 'e'
  containers:
  - name: nginx
    image: nginx
EOF

# Diagnosis
kubectl get pod broken-pod  # Will show Pending
kubectl describe pod broken-pod

# Fix
kubectl delete pod broken-pod
kubectl run broken-pod --image=nginx --overrides='{"spec":{"schedulerName":"my-first-scheduler"}}'
```

**Scenario 2: Missing configuration file**

```bash
# Simulate by renaming config
sudo mv /etc/kubernetes/my-scheduler-config.yaml /etc/kubernetes/my-scheduler-config.yaml.bak

# Scheduler Pod will crashloop
kubectl get pods -n kube-system | grep my-first-scheduler

# Check logs
kubectl logs -n kube-system my-first-scheduler

# Fix
sudo mv /etc/kubernetes/my-scheduler-config.yaml.bak /etc/kubernetes/my-scheduler-config.yaml

# Pod will auto-restart
```

**Scenario 3: Insufficient resources**

```bash
# Create Pod with impossible resource request
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: huge-pod
spec:
  schedulerName: my-first-scheduler
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:
        cpu: "1000"  # Requesting 1000 CPUs (impossible)
        memory: "1Ti"
EOF

# Diagnosis
kubectl describe pod huge-pod
# Look for "0/X nodes are available: insufficient cpu"

# Fix: Reduce resource requests to realistic values
```

### Lab 4: Scheduler with Priority Classes

**Objective:** Combine custom scheduler with priority-based scheduling

**Solution:**

```bash
# Step 1: Create priority classes
cat <<EOF | kubectl apply -f -
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
preemptionPolicy: PreemptLowerPriority
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority
value: 100
preemptionPolicy: Never
EOF

# Step 2: Create Pods with different priorities using custom scheduler
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: high-priority-pod
spec:
  schedulerName: my-first-scheduler
  priorityClassName: high-priority
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:
        cpu: "500m"
---
apiVersion: v1
kind: Pod
metadata:
  name: low-priority-pod
spec:
  schedulerName: my-first-scheduler
  priorityClassName: low-priority
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:
        cpu: "500m"
EOF

# Step 3: Verify scheduling order
kubectl get events -o custom-columns=TIME:.lastTimestamp,POD:.involvedObject.name,REASON:.reason | grep Scheduled

# High priority Pod should be scheduled first
```

### Lab 5: Production HA Scheduler Deployment

**Objective:** Deploy a highly available custom scheduler using Deployments

**Solution:**

```bash
# Step 1: Create namespace
kubectl create namespace scheduler-system

# Step 2: Create ServiceAccount
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: custom-scheduler
  namespace: scheduler-system
EOF

# Step 3: Create RBAC
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: custom-scheduler-as-kube-scheduler
subjects:
- kind: ServiceAccount
  name: custom-scheduler
  namespace: scheduler-system
roleRef:
  kind: ClusterRole
  name: system:kube-scheduler
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: custom-scheduler-as-volume-scheduler
subjects:
- kind: ServiceAccount
  name: custom-scheduler
  namespace: scheduler-system
roleRef:
  kind: ClusterRole
  name: system:volume-scheduler
  apiGroup: rbac.authorization.k8s.io
EOF

# Step 4: Create ConfigMap
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: scheduler-config
  namespace: scheduler-system
data:
  config.yaml: |
    apiVersion: kubescheduler.config.k8s.io/v1
    kind: KubeSchedulerConfiguration
    profiles:
    - schedulerName: ha-custom-scheduler
    leaderElection:
      leaderElect: true
      resourceNamespace: scheduler-system
      resourceName: ha-custom-scheduler
EOF

# Step 5: Create Deployment
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: custom-scheduler
  namespace: scheduler-system
spec:
  replicas: 3  # HA with 3 replicas
  selector:
    matchLabels:
      app: custom-scheduler
  template:
    metadata:
      labels:
        app: custom-scheduler
    spec:
      serviceAccountName: custom-scheduler
      containers:
      - name: kube-scheduler
        image: registry.k8s.io/kube-scheduler:v1.28.0
        command:
        - kube-scheduler
        - --config=/etc/kubernetes/config.yaml
        - --v=2
        volumeMounts:
        - name: config
          mountPath: /etc/kubernetes
      volumes:
      - name: config
        configMap:
          name: scheduler-config
EOF

# Step 6: Verify deployment
kubectl get pods -n scheduler-system
kubectl get lease -n scheduler-system ha-custom-scheduler -o yaml

# Step 7: Test
kubectl run ha-test --image=nginx --overrides='{"spec":{"schedulerName":"ha-custom-scheduler"}}'
kubectl get events | grep ha-test | grep Scheduled
```

---

## Quick Reference Card

### Essential Commands

```bash
# List all schedulers
kubectl get pods -n kube-system | grep scheduler

# Check scheduler logs
kubectl logs -n kube-system <scheduler-pod-name>

# Verify Pod's assigned scheduler
kubectl get pod <pod-name> -o yaml | grep schedulerName

# View scheduling events
kubectl get events -o wide | grep Scheduled

# Check which scheduler assigned a Pod
kubectl get events --field-selector involvedObject.name=<pod-name>

# List Pods using custom scheduler
kubectl get pods --all-namespaces -o custom-columns=NAMESPACE:.metadata.namespace,NAME:.metadata.name,SCHEDULER:.spec.schedulerName | grep -v default-scheduler

# Check leader election
kubectl get lease -n kube-system <scheduler-name> -o yaml
```

### KubeSchedulerConfiguration Template

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
- schedulerName: <SCHEDULER_NAME>
  plugins:
    score:
      enabled:
      - name: <PLUGIN_NAME>
        weight: <INTEGER>
      disabled:
      - name: <PLUGIN_NAME>
leaderElection:
  leaderElect: <true|false>
  resourceNamespace: <NAMESPACE>
  resourceName: <LEASE_NAME>
```

### Static Pod Scheduler Template

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: <SCHEDULER_NAME>
  namespace: kube-system
spec:
  containers:
  - name: kube-scheduler
    image: registry.k8s.io/kube-scheduler:<VERSION>
    command:
    - kube-scheduler
    - --config=/etc/kubernetes/<CONFIG_FILE>.yaml
    volumeMounts:
    - name: config
      mountPath: /etc/kubernetes
  volumes:
  - name: config
    hostPath:
      path: /etc/kubernetes
  hostNetwork: true
```

### Pod Assignment Template

```yaml
spec:
  schedulerName: <SCHEDULER_NAME>
  priorityClassName: <PRIORITY_CLASS>  # Optional
  containers:
  - name: <CONTAINER_NAME>
    image: <IMAGE>
```

---

## Additional Resources

**Official Kubernetes Documentation:**
- [Scheduler Configuration](https://kubernetes.io/docs/reference/scheduling/config/)
- [Scheduling Framework](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/)
- [Multiple Schedulers](https://kubernetes.io/docs/tasks/extend-kubernetes/configure-multiple-schedulers/)

**CKA Exam Preparation:**
- Practice creating schedulers as static Pods (most common exam scenario)
- Understand the four scheduling phases deeply
- Know how to verify scheduler assignment using events
- Master troubleshooting pending Pods

**Key Takeaways for CKA:**
1. Scheduler is a control plane component that assigns Pods to Nodes
2. Scheduling has 4 phases: Queueing, Filtering, Scoring, Binding
3. Custom schedulers are typically deployed as static Pods
4. Use `schedulerName` in Pod spec to assign to custom scheduler
5. Verify using `kubectl get events -o wide` and check SOURCE column
6. KubeSchedulerConfiguration defines profiles and plugins
7. Multiple profiles can run in a single scheduler binary
8. Static Pod manifests go in `/etc/kubernetes/manifests/`
9. Configuration files typically in `/etc/kubernetes/`
10. Always check scheduler logs when troubleshooting

---

**Good luck with your CKA certification! Practice deploying custom schedulers in various configurations, and you'll master this crucial aspect of Kubernetes administration.**