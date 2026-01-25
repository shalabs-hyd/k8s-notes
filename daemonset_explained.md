
# DaemonSet Scheduling – Visual Explanation

## What is a DaemonSet?
A **DaemonSet** ensures that **exactly one Pod runs on every eligible node** in a Kubernetes cluster.

It is commonly used for **system-level components** such as:
- kube-proxy
- CNI plugins
- log collectors
- monitoring agents

---

## High-Level Diagram

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

---

## What Happens When a New Node Joins

```
New Node Added
      |
      v
DaemonSet Controller
- Detects new node
- Checks nodeSelector & tolerations
- Creates Pod with spec.nodeName
      |
      v
Pod runs on the new node
```

Key Points:
- Scheduler is **not involved**
- Pod is **directly bound** to the node
- Exactly **one Pod per node**

---

## What Happens When a Node Is Removed

```
Node Deleted
      |
      v
DaemonSet Controller
- Detects node removal
- Deletes corresponding Pod
```

Key Points:
- No orphan Pods
- Desired state is maintained automatically

---

## Why Scheduler Is Bypassed

DaemonSet Pods are created with:
```
spec:
  nodeName: <node-name>
```

This means:
- Pods never stay in Pending
- Placement is deterministic
- Controller-driven scheduling

---

## Mapping to kube-proxy DaemonSet YAML

| YAML Field | Purpose |
|-----------|--------|
| kind: DaemonSet | One Pod per node |
| nodeSelector | Restrict eligible nodes |
| tolerations: Exists | Run on tainted/system nodes |
| hostNetwork: true | Access node networking |
| priorityClassName | Prevent eviction |
| updateStrategy | Rolling updates |

---

## Lifecycle Summary

```
Node joins  → Pod created
Node leaves → Pod deleted
Scheduler  → Not used
```

---

## Interview One-Liner

"A DaemonSet controller watches nodes and ensures exactly one Pod runs on each eligible node. Pods are automatically created when nodes join and deleted when nodes leave."

---

## Quick Recall

- DaemonSet = one Pod per node
- Controller-driven
- Ideal for system workloads
