# Kubernetes Priority Classes: A Comprehensive Guide for CKA Candidates

## Table of Contents
1. [Introduction to Priority Classes](#introduction)
2. [Why Priority Classes Matter](#why-priority-classes-matter)
3. [Understanding the Priority System](#understanding-the-priority-system)
4. [Creating Priority Classes](#creating-priority-classes)
5. [Assigning Priorities to Pods](#assigning-priorities-to-pods)
6. [Preemption Mechanics](#preemption-mechanics)
7. [Best Practices and Common Pitfalls](#best-practices)
8. [CKA Exam Focus Areas](#cka-exam-focus)
9. [Hands-On Lab Exercises](#hands-on-labs)

---

## Introduction to Priority Classes {#introduction}

**What is a PriorityClass?**

A PriorityClass is a cluster-scoped Kubernetes resource that assigns a numerical priority value to Pods. This value determines:
- The order in which Pods are scheduled from the queue
- Which Pods can evict others when resources are scarce
- How the scheduler makes placement decisions during resource contention

**Key Concept**: Think of priority as a "ticket number" in a queue. Higher priority Pods get served first, and in some cases, can even ask lower-priority Pods to give up their seats (nodes).

---

## Why Priority Classes Matter {#why-priority-classes-matter}

### Real-World Scenario: The Production Crisis

Imagine this situation at 2 AM:
- Your production database crashes and needs to restart
- All cluster nodes are fully utilized
- A developer's test workload is consuming resources on every node
- Without priority classes: Your database waits in line behind test jobs
- With priority classes: Your database immediately evicts test workloads and restarts

### The Three-Tier Workload Model

Modern Kubernetes clusters typically run three categories of workloads:

**1. System Critical (Infrastructure)**
- API Server, etcd, CoreDNS, kube-proxy
- CNI plugins (Calico, Flannel, Cilium)
- Node-level monitoring agents
- **Priority Range**: 2,000,000,000+

**2. Business Critical (Production Applications)**
- Customer-facing APIs and web services
- Production databases (PostgreSQL, MySQL, MongoDB)
- Payment processing systems
- Real-time data pipelines
- **Priority Range**: 100,000 to 1,000,000,000

**3. Best-Effort (Non-Critical)**
- Batch processing jobs
- Machine learning training
- Development and testing workloads
- Log processing and analytics
- **Priority Range**: -2,000,000,000 to 100,000

---

## Understanding the Priority System {#understanding-the-priority-system}

### Priority Value Ranges Explained

| Category | Priority Range | Examples | Can Be Evicted? |
|----------|---------------|----------|-----------------|
| **System Node Critical** | 2,000,010,000 | kube-proxy, CNI | Never |
| **System Cluster Critical** | 2,000,000,000 | CoreDNS, Metrics Server | Rarely |
| **User High Priority** | 100,000 - 1,000,000,000 | Production DBs, APIs | By higher priority only |
| **Default/Standard** | 0 (default) | Regular apps | Yes |
| **Low Priority** | -1 to -2,000,000,000 | Batch jobs, testing | Always |

### Built-in System Priority Classes

Kubernetes provides two pre-configured PriorityClasses:

```bash
kubectl get priorityclass
```

**Expected Output:**
```
NAME                      VALUE        GLOBAL-DEFAULT   AGE
system-cluster-critical   2000000000   false            50d
system-node-critical      2000010000   false            50d
```

**Critical Exam Point**: These system classes are reserved. Users **cannot** create PriorityClasses with values above 1,000,000,000. Attempting to do so will result in an API validation error.

### How Priority Affects Scheduling

The scheduler uses priority in two ways:

**1. Queue Ordering (Pending Pods)**
When multiple Pods are waiting to be scheduled, they are sorted by priority value in descending order. Higher values = earlier scheduling.

**2. Preemption (Running Pods)**
When a node has no available resources, high-priority Pods can trigger eviction of lower-priority Pods to claim the resources.

---

## Creating Priority Classes {#creating-priority-classes}

### Basic PriorityClass Structure

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority-production
value: 1000000
description: "For mission-critical production workloads"
preemptionPolicy: PreemptLowerPriority
globalDefault: false
```

### Field-by-Field Breakdown

**apiVersion: scheduling.k8s.io/v1**
- Uses the scheduling API group (not core v1)
- This is stable since Kubernetes 1.14

**kind: PriorityClass**
- Declares this as a PriorityClass resource
- Always cluster-scoped (no namespace)

**metadata.name**
- Unique identifier across the cluster
- Naming convention: `<tier>-priority` (e.g., `production-critical-priority`)

**value**
- Integer from -2,147,483,648 to 1,000,000,000
- Higher = more important
- Cannot exceed 1 billion for user-defined classes

**description**
- Human-readable explanation
- Helps administrators understand intended use
- Best practice: Include team/purpose info

**preemptionPolicy**
- `PreemptLowerPriority` (default): Can evict lower-priority Pods
- `Never`: Cannot evict, only gets queue priority

**globalDefault**
- `true`: Applied to all Pods without explicit priorityClassName
- `false`: Only applies when explicitly referenced
- **Only one** PriorityClass can be global default

### Practical Examples

**Example 1: Production Database Priority**

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: production-database-priority
value: 500000
description: "Production databases - critical for business operations"
preemptionPolicy: PreemptLowerPriority
globalDefault: false
```

**Example 2: Batch Job Priority**

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: batch-job-priority
value: -100
description: "Low-priority batch processing - can be evicted anytime"
preemptionPolicy: Never
globalDefault: false
```

**Example 3: Setting a Global Default**

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: standard-priority
value: 0
description: "Default priority for all workloads unless specified"
globalDefault: true
preemptionPolicy: PreemptLowerPriority
```

### Creating Priority Classes

```bash
# Apply from YAML file
kubectl apply -f priority-class.yaml

# Verify creation
kubectl get priorityclass production-database-priority

# View details
kubectl describe priorityclass production-database-priority
```

---

## Assigning Priorities to Pods {#assigning-priorities-to-pods}

### Pod Specification with Priority

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: postgres-production
  namespace: production
spec:
  priorityClassName: production-database-priority
  containers:
  - name: postgres
    image: postgres:15
    resources:
      requests:
        memory: "2Gi"
        cpu: "1000m"
      limits:
        memory: "4Gi"
        cpu: "2000m"
```

### Using Priority with Deployments

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-server
  template:
    metadata:
      labels:
        app: api-server
    spec:
      priorityClassName: production-database-priority
      containers:
      - name: api
        image: myapp/api:v2.1
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
```

### Checking Pod Priority

```bash
# View priority value assigned to a pod
kubectl get pod postgres-production -o yaml | grep -A 5 priority

# Expected output:
#   priority: 500000
#   priorityClassName: production-database-priority
```

**Important**: The priority value is automatically populated by the Kubernetes API server when you specify a priorityClassName. You never set the numeric value directly on the Pod.

---

## Preemption Mechanics {#preemption-mechanics}

### Understanding Preemption

**What is Preemption?**
Preemption is the process where the scheduler evicts (removes) lower-priority Pods from a node to make room for a higher-priority Pod that cannot be scheduled anywhere else.

### The Preemption Decision Process

**Step 1: Scheduling Attempt**
- Scheduler tries to place the Pod on available nodes
- Filters nodes based on resource requests, node selectors, taints, etc.

**Step 2: No Suitable Node Found**
- If all nodes lack sufficient resources, scheduler checks if preemption is allowed
- Looks at the Pod's `preemptionPolicy`

**Step 3: Victim Selection**
- Scheduler identifies nodes where evicting Pods would free enough resources
- Selects Pods with lower priority values
- Prefers evicting the minimum number of Pods necessary

**Step 4: Graceful Eviction**
- Selected Pods receive a termination signal (SIGTERM)
- Grace period allows cleanup (default: 30 seconds)
- Pods are forcibly killed (SIGKILL) if they don't exit gracefully

**Step 5: Scheduling the Preemptor**
- Once resources are freed, the high-priority Pod is scheduled
- Status changes from Pending to Running

### Preemption Policy Options

**PreemptLowerPriority (Default Behavior)**

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: can-preempt
value: 100000
preemptionPolicy: PreemptLowerPriority
```

**Behavior:**
- Can evict Pods with lower priority values
- Will wait for graceful termination before scheduling
- Most common for production workloads

**Never (No Preemption)**

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: cannot-preempt
value: 100000
preemptionPolicy: Never
```

**Behavior:**
- Gets priority in the scheduling queue
- Will NOT evict any running Pods
- Waits until resources naturally become available
- Useful for batch jobs that should not disrupt production

### Worked Example: The 5-6-7 Scenario

**Initial State:**
- Node "worker-1" has 4 CPU cores total
- Running Pods:
  - `critical-app` (Priority: 7, Using: 2 CPUs)
  - `background-job` (Priority: 5, Using: 2 CPUs)
- Available: 0 CPUs

**Scenario A: New Pod with PreemptLowerPriority**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secondary-job
spec:
  priorityClassName: medium-priority  # value: 6
  containers:
  - name: worker
    image: processor:v1
    resources:
      requests:
        cpu: "2000m"
```

**What Happens:**
1. Scheduler cannot place `secondary-job` (needs 2 CPUs, none available)
2. Checks preemptionPolicy: `PreemptLowerPriority`
3. Identifies `background-job` (priority 5 < 6)
4. Evicts `background-job`
5. Waits for termination to complete
6. Schedules `secondary-job` on worker-1
7. `critical-app` (priority 7) remains untouched

**Final State:**
- `critical-app` (Priority 7) - Running
- `secondary-job` (Priority 6) - Running
- `background-job` (Priority 5) - Evicted, back to Pending

**Scenario B: New Pod with Never Policy**

Same setup, but `secondary-job` uses:

```yaml
spec:
  priorityClassName: medium-priority-no-preempt  # value: 6, preemptionPolicy: Never
```

**What Happens:**
1. Scheduler cannot place `secondary-job`
2. Checks preemptionPolicy: `Never`
3. Places Pod at front of scheduling queue (ahead of priority 5 or lower)
4. Does NOT evict `background-job`
5. Waits for natural resource availability
6. Remains in Pending state

**Final State (Immediate):**
- `critical-app` (Priority 7) - Running
- `background-job` (Priority 5) - Running
- `secondary-job` (Priority 6) - Pending (waiting)

### Preemption and Pod Disruption Budgets (PDBs)

**Critical Exam Point**: Preemption respects PodDisruptionBudgets.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: db-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: database
```

If preempting a Pod would violate a PDB, the scheduler will:
1. Try to find alternative victims
2. Wait if no safe alternative exists
3. Eventually schedule elsewhere or wait for natural availability

---

## Best Practices and Common Pitfalls {#best-practices}

### Best Practices

**1. Establish a Clear Priority Hierarchy**

Create a documented priority scheme for your organization:

```yaml
# System Infrastructure (Use built-in classes)
# - system-node-critical: 2,000,010,000
# - system-cluster-critical: 2,000,000,000

# Production Tier 1 (Revenue Critical)
- name: production-tier1-priority
  value: 900000

# Production Tier 2 (Important but not critical)
- name: production-tier2-priority
  value: 500000

# Staging/QA
- name: staging-priority
  value: 100000

# Development
- name: development-priority
  value: 0

# Batch/Background
- name: batch-priority
  value: -1000
```

**2. Use Resource Quotas with Priority Classes**

Prevent priority abuse by limiting resource consumption per priority:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: high-priority-quota
  namespace: production
spec:
  hard:
    requests.cpu: "100"
    requests.memory: "200Gi"
  scopeSelector:
    matchExpressions:
    - operator: In
      scopeName: PriorityClass
      values: ["production-tier1-priority"]
```

**3. Never Set High Priority as Global Default**

❌ **Wrong:**
```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: production-tier1-priority
value: 900000
globalDefault: true  # DANGER!
```

This causes every Pod (including test Pods) to get high priority, defeating the purpose.

✅ **Correct:**
```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: standard-priority
value: 0
globalDefault: true  # Safe default
```

**4. Document Priority Class Usage**

Include usage guidelines in the description:

```yaml
description: "Production Tier 1 - Revenue-critical services only. Requires approval from Platform Team. Maximum 10 replicas per deployment."
```

**5. Monitor Preemption Events**

```bash
# Watch for preemption events
kubectl get events --field-selector reason=Preempted --all-namespaces

# Check specific namespace
kubectl get events -n production | grep -i preempt
```

**6. Combine with Node Affinity/Taints**

Prevent low-priority workloads from even attempting to schedule on production nodes:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: batch-job
spec:
  priorityClassName: batch-priority
  tolerations:
  - key: "workload-type"
    operator: "Equal"
    value: "batch"
    effect: "NoSchedule"
```

### Common Pitfalls

**Pitfall 1: Forgetting PriorityClass is Cluster-Scoped**

❌ **This will fail:**
```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: my-priority
  namespace: production  # ERROR: PriorityClass doesn't support namespaces
value: 100000
```

**Pitfall 2: Using Wrong API Version**

❌ **This will fail:**
```yaml
apiVersion: v1  # Wrong! This is for core resources
kind: PriorityClass
```

✅ **Correct:**
```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
```

**Pitfall 3: Setting Priority Value Directly on Pods**

❌ **This will be rejected:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  priority: 100000  # Cannot set directly
  containers:
  - name: app
    image: myapp
```

✅ **Correct:**
```yaml
spec:
  priorityClassName: production-tier1-priority  # Reference the class
```

**Pitfall 4: Preemption Spirals**

Without resource limits, high-priority Pods can repeatedly evict each other:

**Scenario:**
1. High-priority Pod A evicts low-priority Pod B
2. Pod B rescheduled with even higher priority
3. Pod B evicts Pod A
4. Repeat infinitely

**Solution:** Enforce resource quotas and limit replica counts.

**Pitfall 5: Ignoring System Reserved Classes**

❌ **This will be rejected by API server:**
```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: my-critical-class
value: 2000000000  # Above 1 billion - reserved for system
```

Error: `value must be less than or equal to 1000000000`

---

## CKA Exam Focus Areas {#cka-exam-focus}

### What You Must Know

**1. API Details (High Likelihood)**
- API group: `scheduling.k8s.io/v1`
- Scope: Cluster-scoped (no namespace)
- Fields: `value`, `description`, `preemptionPolicy`, `globalDefault`

**2. Creating Priority Classes (High Likelihood)**

You should be able to write this from memory:
```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: exam-priority
value: 100000
description: "CKA exam priority class"
preemptionPolicy: PreemptLowerPriority
globalDefault: false
```

**3. Pod Assignment (High Likelihood)**

Know how to assign priority to different resource types:
```yaml
# Pod
spec:
  priorityClassName: exam-priority

# Deployment template
spec:
  template:
    spec:
      priorityClassName: exam-priority

# StatefulSet template
spec:
  template:
    spec:
      priorityClassName: exam-priority
```

**4. Inspection Commands (Medium Likelihood)**

```bash
# List all priority classes
kubectl get priorityclass

# Describe specific class
kubectl describe priorityclass production-tier1-priority

# Check which class is global default
kubectl get priorityclass -o custom-columns=NAME:.metadata.name,VALUE:.value,GLOBAL:.globalDefault

# View Pod's assigned priority
kubectl get pod mypod -o jsonpath='{.spec.priorityClassName}'
```

**5. Understanding Preemption (Medium Likelihood)**

Be able to explain:
- When preemption occurs (resource exhaustion)
- Victim selection criteria (lower priority, fewest Pods)
- Difference between `PreemptLowerPriority` and `Never`

**6. Priority Value Constraints (Low Likelihood but Critical)**

- User range: -2,147,483,648 to 1,000,000,000
- System reserved: Above 1,000,000,000
- Default (no class): 0

### Time-Saving Exam Tips

**Tip 1: Use kubectl explain**

```bash
kubectl explain priorityclass
kubectl explain priorityclass.spec
```

**Tip 2: Generate and Modify**

```bash
# Create quickly with dry-run
kubectl create priorityclass exam-priority --value=100000 --dry-run=client -o yaml > priority.yaml

# Edit as needed
vim priority.yaml
```

**Tip 3: Verify Before Moving On**

```bash
# Quick verification
kubectl get priorityclass exam-priority -o yaml
kubectl get pod testpod -o yaml | grep priority
```

**Tip 4: Know the Built-in Classes**

Memorize these for quick reference:
- `system-node-critical`: 2,000,010,000
- `system-cluster-critical`: 2,000,000,000

### Exam Question Patterns

**Pattern 1: "Create a priority class..."**
- Write the YAML from scratch
- Apply it
- Verify with `kubectl get`

**Pattern 2: "Configure this Pod/Deployment to use..."**
- Edit existing resource
- Add `priorityClassName` field
- Apply changes

**Pattern 3: "Which Pod will be evicted when..."**
- Analyze priorities
- Understand preemption policy
- Explain the outcome

**Pattern 4: "Troubleshoot why Pod is not scheduling..."**
- Check priority class exists
- Verify spelling of `priorityClassName`
- Check for preemption events

---

## Hands-On Lab Exercises {#hands-on-labs}

### Lab 1: Basic Priority Class Creation

**Objective:** Create three priority classes representing different tiers.

**Tasks:**
1. Create a high-priority class named `production-critical` with value 900000
2. Create a medium-priority class named `production-standard` with value 500000
3. Create a low-priority class named `batch-processing` with value -1000
4. Set `production-standard` as the global default
5. Verify all classes are created correctly

**Solution:**

```bash
# Create high priority
cat <<EOF | kubectl apply -f -
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: production-critical
value: 900000
description: "Critical production workloads"
preemptionPolicy: PreemptLowerPriority
globalDefault: false
EOF

# Create medium priority (global default)
cat <<EOF | kubectl apply -f -
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: production-standard
value: 500000
description: "Standard production workloads"
preemptionPolicy: PreemptLowerPriority
globalDefault: true
EOF

# Create low priority
cat <<EOF | kubectl apply -f -
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: batch-processing
value: -1000
description: "Batch processing jobs"
preemptionPolicy: Never
globalDefault: false
EOF

# Verify
kubectl get priorityclass
```

### Lab 2: Pod Priority Assignment

**Objective:** Deploy Pods with different priorities and observe scheduling behavior.

**Setup:**
```bash
# Create namespace
kubectl create namespace priority-lab

# Create priority classes from Lab 1 if not already present
```

**Tasks:**
1. Deploy a database Pod with `production-critical` priority
2. Deploy an API Pod with `production-standard` priority
3. Deploy a batch job Pod with `batch-processing` priority
4. Deploy a Pod with no priority specified (should get global default)
5. Verify each Pod's assigned priority value

**Solution:**

```bash
# Database with critical priority
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: database
  namespace: priority-lab
spec:
  priorityClassName: production-critical
  containers:
  - name: db
    image: postgres:15
    resources:
      requests:
        cpu: "500m"
        memory: "1Gi"
EOF

# API with standard priority
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: api-server
  namespace: priority-lab
spec:
  priorityClassName: production-standard
  containers:
  - name: api
    image: nginx:latest
    resources:
      requests:
        cpu: "250m"
        memory: "512Mi"
EOF

# Batch job with low priority
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: batch-job
  namespace: priority-lab
spec:
  priorityClassName: batch-processing
  containers:
  - name: worker
    image: busybox
    command: ["sleep", "3600"]
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
EOF

# Pod with no priority (gets global default)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: default-pod
  namespace: priority-lab
spec:
  containers:
  - name: app
    image: nginx:latest
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
EOF

# Verify priorities
kubectl get pods -n priority-lab -o custom-columns=NAME:.metadata.name,PRIORITY:.spec.priority,CLASS:.spec.priorityClassName
```

### Lab 3: Observing Preemption

**Objective:** Create a resource-constrained scenario and observe preemption in action.

**Prerequisites:** A cluster with limited resources or use resource quotas

**Tasks:**
1. Create a single-node cluster or identify a node with limited resources
2. Fill the node with low-priority Pods
3. Deploy a high-priority Pod
4. Observe the preemption event
5. Check which Pod was evicted

**Solution:**

```bash
# Create low-priority Pods to fill node
for i in {1..5}; do
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: low-priority-pod-$i
  namespace: priority-lab
spec:
  priorityClassName: batch-processing
  containers:
  - name: worker
    image: nginx:latest
    resources:
      requests:
        cpu: "500m"
        memory: "512Mi"
EOF
done

# Wait for all to be running
kubectl wait --for=condition=Ready pod -l app=low-priority -n priority-lab --timeout=60s

# Deploy high-priority Pod that requires resources
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: high-priority-pod
  namespace: priority-lab
spec:
  priorityClassName: production-critical
  containers:
  - name: critical-app
    image: nginx:latest
    resources:
      requests:
        cpu: "1000m"
        memory: "1Gi"
EOF

# Watch for preemption events
kubectl get events -n priority-lab --sort-by='.lastTimestamp' | grep -i preempt

# Check Pod status
kubectl get pods -n priority-lab -o wide
```

### Lab 4: Troubleshooting Priority Issues

**Objective:** Practice diagnosing common priority-related problems.

**Scenario 1: Pod Not Using Expected Priority**

```bash
# Create a Pod with typo in priorityClassName
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: broken-pod
  namespace: priority-lab
spec:
  priorityClassName: production-critial  # Typo: "critial" instead of "critical"
  containers:
  - name: app
    image: nginx:latest
EOF

# Task: Find and fix the issue
# Solution:
kubectl describe pod broken-pod -n priority-lab
# Look for events showing priority class not found
# Fix: Edit and correct the priorityClassName
kubectl edit pod broken-pod -n priority-lab
```

**Scenario 2: Cannot Create High-Priority Class**

```bash
# Try to create a priority class in reserved range
cat <<EOF | kubectl apply -f -
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: super-priority
value: 2000000001  # Above 1 billion
description: "Attempting reserved range"
EOF

# Task: Understand why this fails
# Solution: Check error message, adjust value to <= 1000000000
```

### Lab 5: Real-World Production Scenario

**Objective:** Implement a complete priority strategy for a microservices application.

**Application Components:**
- Frontend (3 replicas) - Medium priority
- Backend API (3 replicas) - High priority
- Database (1 replica) - Critical priority
- Cache (2 replicas) - High priority
- Background worker (5 replicas) - Low priority
- ML training job (1 replica) - Very low priority

**Tasks:**
1. Design appropriate priority classes
2. Create ResourceQuotas for each priority tier
3. Deploy all components
4. Simulate node failure and observe rescheduling behavior
5. Verify critical services recover first

**Solution:** (Implementation left as exercise - apply concepts from previous labs)

---

## Quick Reference Card

### Essential Commands

```bash
# List all priority classes
kubectl get priorityclass

# Create priority class
kubectl create priorityclass <name> --value=<number> --dry-run=client -o yaml

# Describe priority class
kubectl describe priorityclass <name>

# Delete priority class
kubectl delete priorityclass <name>

# Check Pod's priority
kubectl get pod <name> -o jsonpath='{.spec.priority}'

# Check which is global default
kubectl get priorityclass -o json | jq '.items[] | select(.globalDefault==true)'

# Monitor preemption events
kubectl get events --field-selector reason=Preempted -A
```

### Priority Class Template

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: <CLASS_NAME>
value: <INTEGER_VALUE>  # -2147483648 to 1000000000
description: "<DESCRIPTION>"
preemptionPolicy: PreemptLowerPriority  # or Never
globalDefault: false  # or true (only one can be true)
```

### Pod Priority Assignment Template

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: <POD_NAME>
spec:
  priorityClassName: <CLASS_NAME>
  containers:
  - name: <CONTAINER_NAME>
    image: <IMAGE>
```

---

## Additional Resources

**Official Kubernetes Documentation:**
- [Pod Priority and Preemption](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/)
- [Configure Quality of Service for Pods](https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/)

**CKA Exam Preparation:**
- Practice creating priority classes in different scenarios
- Understand the relationship between priorities, QoS, and resource requests/limits
- Know how to troubleshoot scheduling issues related to priority

**Key Takeaways for CKA:**
1. PriorityClass is cluster-scoped and uses `scheduling.k8s.io/v1` API
2. Priority is assigned via `spec.priorityClassName` in Pod spec
3. Default priority is 0 unless a global default is configured
4. User priority values must be ≤ 1,000,000,000
5. Preemption respects PodDisruptionBudgets
6. Only one PriorityClass can be the global default

---

**Good luck with your CKA certification! Master these concepts through hands-on practice, and you'll be well-prepared for both the exam and real-world cluster management.**