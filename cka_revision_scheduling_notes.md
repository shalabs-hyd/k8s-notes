# CKA Revision Notes - Pod Scheduling & Resource Management

## 1. Labels & Selectors

### Labels
Labels are key-value pairs attached to Kubernetes objects (pods, services, deployments, etc.) for identification and organization.

**Key Points:**
- Used to organize and select subsets of objects
- Can be added at creation time or modified later
- Multiple labels can be added to a single object

**Example:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
    environment: production
    tier: frontend
spec:
  containers:
  - name: nginx
    image: nginx
```

### Selectors
Selectors are used to filter Kubernetes objects based on their labels.

**Equality-based selectors:**
```bash
kubectl get pods -l environment=production
kubectl get pods -l environment=production,tier=frontend
kubectl get pods -l environment!=production
```

**Set-based selectors:**
```bash
kubectl get pods -l 'environment in (production,qa)'
kubectl get pods -l 'tier notin (frontend,backend)'
```

**In YAML (for Services, Deployments):**
```yaml
selector:
  matchLabels:
    app: myapp
  matchExpressions:
  - key: environment
    operator: In
    values:
    - production
    - qa
```

---

## 2. Taints & Tolerations

Taints and tolerations work together to ensure pods are not scheduled on inappropriate nodes.

### Taints (Applied to Nodes)
Taints repel pods from being scheduled on a node unless the pod has a matching toleration.

**Taint Effects:**
- `NoSchedule`: Pods will not be scheduled on the node
- `PreferNoSchedule`: System tries to avoid scheduling pods on the node (soft version)
- `NoExecute`: New pods won't be scheduled, and existing pods will be evicted if they don't tolerate the taint

**Commands:**
```bash
# Add taint to node
kubectl taint nodes node01 key=value:NoSchedule

# Remove taint from node
kubectl taint nodes node01 key=value:NoSchedule-

# View taints on a node
kubectl describe node node01 | grep Taint
```

**Example:**
```bash
kubectl taint nodes node01 app=blue:NoSchedule
```

### Tolerations (Applied to Pods)
Tolerations allow pods to be scheduled on nodes with matching taints.

**Pod YAML with Toleration:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
  - name: nginx
    image: nginx
  tolerations:
  - key: "app"
    operator: "Equal"
    value: "blue"
    effect: "NoSchedule"
```

**Important Notes:**
- Taints and tolerations do NOT guarantee that pods will be placed on specific nodes
- They only restrict which nodes can accept which pods
- Master nodes have a taint by default to prevent user pods from being scheduled on them

---

## 3. Node Selector

Node Selector is the simplest way to constrain pods to run on specific nodes based on labels.

### Steps to Use Node Selector

**1. Label the node:**
```bash
kubectl label nodes node01 size=large
kubectl label nodes node02 size=small
```

**2. Add nodeSelector to pod specification:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
  - name: nginx
    image: nginx
  nodeSelector:
    size: large
```

**Limitations:**
- Cannot use advanced expressions like "OR" or "NOT"
- Cannot specify "place on large OR medium nodes"
- For complex requirements, use Node Affinity

**View node labels:**
```bash
kubectl get nodes --show-labels
kubectl describe node node01
```

---

## 4. Node Affinity

Node Affinity provides advanced capabilities for constraining which nodes your pods can be scheduled on based on node labels.

### Types of Node Affinity

**1. `requiredDuringSchedulingIgnoredDuringExecution`**
- Hard requirement, pod won't be scheduled if rules aren't met
- Similar to nodeSelector but with more expressive syntax

**2. `preferredDuringSchedulingIgnoredDuringExecution`**
- Soft preference, scheduler tries to enforce but doesn't guarantee
- Pod will still be scheduled even if preference can't be met

**Future (Planned):**
- `requiredDuringSchedulingRequiredDuringExecution`: Will evict pods if node labels change

### Example YAML

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
  - name: nginx
    image: nginx
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: size
            operator: In
            values:
            - large
            - medium
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
```

### Operators Available
- `In`: Label value must be in the list
- `NotIn`: Label value must not be in the list
- `Exists`: Label key must exist (value doesn't matter)
- `DoesNotExist`: Label key must not exist
- `Gt`: Greater than (for numeric values)
- `Lt`: Less than (for numeric values)

### Key Differences: Node Selector vs Node Affinity

| Feature | Node Selector | Node Affinity |
|---------|---------------|---------------|
| Syntax | Simple | More expressive |
| Operators | Only equality | In, NotIn, Exists, etc. |
| Multiple values | No | Yes (OR logic) |
| Required vs Preferred | Only required | Both options |

---

## 5. Combining Node Affinity with Taints & Tolerations

Using both together provides fine-grained control over pod placement.

### Use Case
- **Taints & Tolerations**: Prevent unwanted pods from being placed on dedicated nodes
- **Node Affinity**: Ensure specific pods are placed on dedicated nodes

### Example Scenario
You have dedicated nodes for a specific application (blue app) and want to ensure:
1. Only blue pods can run on blue nodes (Taints & Tolerations)
2. Blue pods only run on blue nodes (Node Affinity)

**Step 1: Taint the nodes**
```bash
kubectl taint nodes node01 app=blue:NoSchedule
```

**Step 2: Create pod with both toleration and node affinity**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: blue-pod
spec:
  containers:
  - name: nginx
    image: nginx
  tolerations:
  - key: "app"
    operator: "Equal"
    value: "blue"
    effect: "NoSchedule"
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: app
            operator: In
            values:
            - blue
```

**Result:**
- Taint ensures other pods can't be scheduled on blue nodes
- Toleration allows blue pod to tolerate the taint
- Node Affinity ensures blue pod only goes to nodes labeled with app=blue

---

## 6. Resource Requirements

### 6a. Requests & Limits

Resource requests and limits define how much CPU and memory a container needs and can use.

**Requests:**
- Minimum resources guaranteed to a container
- Scheduler uses this to decide which node to place the pod on
- Container can use more than requested if available

**Limits:**
- Maximum resources a container can use
- If exceeded, container is throttled (CPU) or terminated (Memory)

**Example:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

**Resource Units:**
- **CPU**: 
  - 1 CPU = 1000m (millicores)
  - 0.1 = 100m
  - Minimum: 1m
- **Memory**:
  - 1 G (Gigabyte) = 1,000,000,000 bytes
  - 1 Gi (Gibibyte) = 1,073,741,824 bytes
  - Ki, Mi, Gi, Ti (binary) or K, M, G, T (decimal)

**Behavior:**
- **CPU throttling**: If container exceeds CPU limit, it's throttled
- **Memory termination**: If container exceeds memory limit, pod is terminated with OOMKilled (Out Of Memory)

**Default Behavior (if not specified):**
- No requests: Pod can be scheduled on any node
- No limits: Pod can consume all available resources on the node

---

### 6b. LimitRange

LimitRange sets default requests/limits and constraints for pods/containers in a namespace.

**Purpose:**
- Set default resource requests and limits
- Set minimum and maximum constraints
- Enforce resource ratios

**Example:**
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-memory-limitrange
  namespace: default
spec:
  limits:
  - default:           # Default limits
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:    # Default requests
      cpu: "250m"
      memory: "256Mi"
    max:               # Maximum allowed
      cpu: "1"
      memory: "1Gi"
    min:               # Minimum required
      cpu: "100m"
      memory: "128Mi"
    type: Container
  - max:               # For entire pod
      cpu: "2"
      memory: "2Gi"
    type: Pod
```

**Commands:**
```bash
# Create LimitRange
kubectl create -f limitrange.yaml

# View LimitRange
kubectl get limitrange
kubectl describe limitrange cpu-memory-limitrange
```

**Key Points:**
- Applied at namespace level
- Affects pods created after LimitRange is created
- Does not affect existing pods
- If pod doesn't specify resources, LimitRange defaults are applied

---

### 6c. ResourceQuota

ResourceQuota limits the total amount of resources that can be consumed in a namespace.

**Purpose:**
- Limit total compute resources (CPU, memory) in a namespace
- Limit total number of objects (pods, services, PVCs) in a namespace
- Enforce resource allocation across teams

**Example:**
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: dev
spec:
  hard:
    requests.cpu: "10"
    requests.memory: "20Gi"
    limits.cpu: "20"
    limits.memory: "40Gi"
    pods: "10"
    services: "5"
    persistentvolumeclaims: "5"
```

**Another Example (Object count limits):**
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: object-quota
  namespace: dev
spec:
  hard:
    configmaps: "10"
    secrets: "10"
    services.loadbalancers: "2"
```

**Commands:**
```bash
# Create ResourceQuota
kubectl create -f resourcequota.yaml -n dev

# View ResourceQuota
kubectl get resourcequota -n dev
kubectl describe resourcequota compute-quota -n dev
```

**Important Notes:**
- When ResourceQuota is enabled, pods MUST specify resource requests/limits
- Enforced at namespace level
- Total resource usage cannot exceed the quota
- If quota is reached, new pods won't be created until resources are freed

**Example Output:**
```bash
Name:            compute-quota
Namespace:       dev
Resource         Used   Hard
--------         ----   ----
limits.cpu       8      20
limits.memory    16Gi   40Gi
pods             7      10
requests.cpu     4      10
requests.memory  8Gi    20Gi
```

---

## Quick Command Reference

```bash
# Labels & Selectors
kubectl label nodes node01 key=value
kubectl get pods -l key=value
kubectl get pods --selector key=value

# Taints & Tolerations
kubectl taint nodes node01 key=value:NoSchedule
kubectl taint nodes node01 key=value:NoSchedule-  # Remove taint
kubectl describe node node01 | grep Taint

# View/Create Resources
kubectl get limitrange
kubectl get resourcequota
kubectl describe limitrange <name>
kubectl describe resourcequota <name>

# Check resource usage
kubectl top nodes
kubectl top pods

# Dry run (test without creating)
kubectl run nginx --image=nginx --dry-run=client -o yaml
```

---

## Exam Tips

1. **Practice imperative commands** for quick pod creation with labels
2. **Remember**: Taints on nodes, Tolerations on pods
3. **Node Affinity vs Node Selector**: Use Node Affinity for complex requirements
4. **Combining strategies**: Use both Taints/Tolerations AND Node Affinity for dedicated nodes
5. **LimitRange applies defaults**, ResourceQuota sets total limits
6. **Always specify resources** when ResourceQuota is in place
7. Use `kubectl explain` for field references during exam
8. Practice YAML syntax for all scheduling and resource configurations