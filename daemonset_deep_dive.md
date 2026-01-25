# DaemonSet Scheduling — Complete & Updated Reference

This document combines the original DaemonSet explanation with the post—Kubernetes v1.12 scheduling update. It is designed for solid understanding, quick revision, and interviews.

---

## 1. What is a DaemonSet?

A DaemonSet ensures that exactly one Pod runs on every eligible node in a Kubernetes cluster.

**Typical use cases:**
- **kube-proxy** — Network proxy on every node
- **CNI plugins** — Network fabric (Calico, Flannel, Weave)
- **Log collectors** — Fluentd, Filebeat
- **Monitoring agents** — Prometheus Node Exporter, Datadog agent
- **Security agents** — Falco, Twistlock
- **Storage daemons** — Ceph, GlusterFS

**Mental model:** One Pod per Node (like a system daemon)

---

## 2. High-Level Cluster View

```
                    Kubernetes Cluster
                 DaemonSet: kube-proxy
            (Desired State: 1 Pod per Node)

        +-----------+   +-----------+   +-----------+
        |  Node-1   |   |  Node-2   |   |  Node-3   |
        |  linux    |   |  linux    |   |  linux    |
        | kube-proxy|   | kube-proxy|   | kube-proxy|
        +-----------+   +-----------+   +-----------+
```

**Key behavior:**
- Automatic Pod creation on each node
- No replicas count — tied to node count
- Self-healing if Pod crashes

---

## 3. What Happens When a New Node Joins

```
New Node joins cluster
        ↓
DaemonSet Controller (in kube-controller-manager)
- Detects new node via API server watch
- Checks nodeSelector & tolerations
- Creates Pod object with NodeAffinity
        ↓
Default Scheduler
- Evaluates Pod
- Binds Pod to the target node
        ↓
Kubelet on that node
- Pulls image
- Starts container
        ↓
Pod runs on that node
```

**Key points:**
- **Fully automated** — no manual action required
- **Exactly one Pod per node** guarantee
- **Works with node taints** via tolerations
- **Respects node selectors** for targeting specific nodes

---

## 4. What Happens When a Node Is Removed

```
Node removed/cordoned from cluster
        ↓
DaemonSet Controller
- Detects node removal via API server
- Deletes corresponding Pod object
        ↓
Pod is gracefully terminated
```

**Key points:**
- No orphan Pods left behind
- Desired state automatically maintained
- If node is drained, DaemonSet Pods are deleted (respecting PodDisruptionBudget if configured)

---

## 5. Scheduling Behavior — Evolution & Technical Detail

### **Before Kubernetes v1.12** (Legacy)
- Pods were **bound directly** using `spec.nodeName`
- **Scheduler was bypassed** completely
- DaemonSet controller wrote nodeName directly into Pod spec

**Issues with old approach:**
- Ignored scheduler features (affinity, taints, resource limits)
- No integration with scheduler plugins
- Hard to extend or customize

---

### **From Kubernetes v1.12 onwards** (Current)
- DaemonSet Pods **use NodeAffinity**
- **Default scheduler is used** for binding
- Controller creates Pod with node-specific affinity

**Internal representation:**
```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchFields:
        - key: metadata.name
          operator: In
          values:
          - <target-node-name>
```

**Why this change matters:**
- ✅ Works with scheduler extenders and plugins
- ✅ Honors resource requests/limits
- ✅ Integrates with taints/tolerations properly
- ✅ Supports priority and preemption
- ✅ Better observability (scheduler events)

---

## 6. Node Selection & Filtering

DaemonSet Pods can be restricted to specific nodes using:

### **a) nodeSelector**
```yaml
spec:
  template:
    spec:
      nodeSelector:
        disktype: ssd
        region: us-west
```

### **b) Node Affinity** (more flexible)
```yaml
spec:
  template:
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: node-role.kubernetes.io/worker
                operator: Exists
```

### **c) Tolerations** (for tainted nodes)
```yaml
spec:
  template:
    spec:
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        effect: NoSchedule
      - key: node.kubernetes.io/not-ready
        effect: NoExecute
```

**Common pattern:** System DaemonSets tolerate all taints to run everywhere
```yaml
tolerations:
- operator: Exists  # Tolerates ALL taints
```

---

## 7. What Did Not Change (User Perspective)

- ✅ One Pod per eligible node
- ✅ Pods automatically created on node join
- ✅ Pods automatically deleted on node removal
- ✅ Controlled by DaemonSet controller in kube-controller-manager
- ✅ Rolling updates supported
- ✅ Rollback capability

**The abstraction remained stable — only internal implementation changed.**

---

## 8. Mapping to kube-proxy DaemonSet

| Field | Purpose | Example |
|-------|---------|---------|
| `kind: DaemonSet` | One Pod per node | - |
| `nodeSelector` | Target specific nodes | `beta.kubernetes.io/os: linux` |
| `tolerations` | Run on tainted nodes | Tolerate control-plane taint |
| `hostNetwork: true` | Use node's network namespace | Access node IP directly |
| `priorityClassName` | Prevent eviction | `system-node-critical` |
| `updateStrategy` | Control rolling updates | `RollingUpdate` or `OnDelete` |

---

## 9. Update Strategies

### **RollingUpdate (default)**
```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1  # Only 1 node without DaemonSet Pod at a time
```
- Updates Pods node by node
- Controlled rollout
- Safer for critical system components

### **OnDelete**
```yaml
spec:
  updateStrategy:
    type: OnDelete
```
- Pods updated only when manually deleted
- Maximum control
- Useful for testing updates selectively

---

## 10. Lifecycle Summary

```
Node joins  → Controller creates Pod → Scheduler binds → Kubelet runs → Pod running
Node leaves → Controller deletes Pod → Pod terminated
Pod crashes → Kubelet restarts → Self-healing
DaemonSet updated → Rolling update starts → Pods recreated node by node
```

---

## 11. Common Pitfalls & Troubleshooting

### **Problem: DaemonSet Pod not scheduled on a node**
**Check:**
1. Node labels match `nodeSelector`
2. Node taints vs Pod tolerations
3. Node has sufficient resources (CPU/memory)
4. Check `kubectl describe pod <pod-name>` for events
5. Verify DaemonSet controller is running

### **Problem: Multiple Pods on one node**
**Cause:** Usually misconfigured anti-affinity or manual Pod creation
**Fix:** Delete extra Pods; controller will reconcile

### **Problem: DaemonSet not updating**
**Check:**
- Update strategy is `RollingUpdate`
- `maxUnavailable` setting
- PodDisruptionBudget might be blocking

---

## 12. Real-World Example: CNI DaemonSet

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: calico-node
  namespace: kube-system
spec:
  selector:
    matchLabels:
      k8s-app: calico-node
  template:
    metadata:
      labels:
        k8s-app: calico-node
    spec:
      hostNetwork: true  # Access node network
      tolerations:
      - operator: Exists  # Run on all nodes including control-plane
      priorityClassName: system-node-critical
      containers:
      - name: calico-node
        image: calico/node:v3.25.0
        env:
        - name: DATASTORE_TYPE
          value: "kubernetes"
        securityContext:
          privileged: true  # Needed for network manipulation
        volumeMounts:
        - name: lib-modules
          mountPath: /lib/modules
          readOnly: true
      volumes:
      - name: lib-modules
        hostPath:
          path: /lib/modules
```

---

## 13. Interview Questions & Answers

**Q: How does DaemonSet differ from Deployment?**
**A:** DaemonSet ensures one Pod per node (node-scoped), while Deployment ensures N replicas cluster-wide (cluster-scoped). DaemonSet doesn't use replica count; it's tied to node count.

**Q: How did DaemonSet scheduling change in v1.12?**
**A:** Before v1.12, Pods used `spec.nodeName` and bypassed the scheduler. From v1.12+, they use NodeAffinity and go through the default scheduler, enabling better integration with scheduler features.

**Q: Can DaemonSet Pods be evicted?**
**A:** Yes, unless they have `priorityClassName: system-node-critical` or `system-cluster-critical`, which prevents eviction during resource pressure.

**Q: What happens if I delete a DaemonSet?**
**A:** All Pods managed by it are deleted. Use `--cascade=orphan` to keep Pods running.

**Q: Can I run DaemonSet only on specific nodes?**
**A:** Yes, using `nodeSelector`, node affinity, or by ensuring only specific nodes lack certain taints.

---

## 14. Key kubectl Commands

```bash
# List DaemonSets
kubectl get daemonsets -A

# Describe DaemonSet
kubectl describe daemonset <name> -n <namespace>

# Check DaemonSet Pod distribution
kubectl get pods -o wide -l <daemonset-label>

# Update DaemonSet image
kubectl set image daemonset/<name> <container>=<new-image> -n <namespace>

# Check rollout status
kubectl rollout status daemonset/<name> -n <namespace>

# Rollback DaemonSet
kubectl rollout undo daemonset/<name> -n <namespace>

# Delete DaemonSet (keeps Pods)
kubectl delete daemonset <name> --cascade=orphan
```

---

## 15. Interview One-Liner

**"A DaemonSet ensures exactly one Pod runs on every eligible node in the cluster. Since Kubernetes v1.12, DaemonSet Pods are scheduled using NodeAffinity and the default scheduler, while the DaemonSet controller manages their creation, deletion, and lifecycle. This enables system-level services like networking, logging, and monitoring to run uniformly across all nodes."**

---

## Final Takeaway

**The abstraction never changed** — DaemonSets always guaranteed one Pod per node. Only the **internal scheduling mechanism evolved** to leverage the scheduler properly. This behavior is:
- ✅ Stable and production-proven
- ✅ Predictable and deterministic  
- ✅ Interview-safe and exam-ready

**Master this, and you've mastered DaemonSets.**