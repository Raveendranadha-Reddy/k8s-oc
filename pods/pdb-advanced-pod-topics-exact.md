# Advanced Pod Topics – PodDisruptionBudget and Voluntary Disruptions

## 1. What Is a PodDisruptionBudget (PDB)?

A PodDisruptionBudget is a Kubernetes policy that limits how many Pods belonging to an application can be voluntarily disrupted at a given time.

It helps maintain application availability during operations such as:

- Worker node maintenance
- `kubectl drain`
- Cluster upgrades
- Planned infrastructure maintenance
- Other operations that use the Kubernetes Eviction API

Important: A PDB does not prevent every Pod failure. It primarily governs voluntary disruptions through the Eviction API. It does not guarantee protection against node crashes, hardware failures, or other involuntary disruptions.

## 2. What Are Voluntary Disruptions?

A disruption is an event that interrupts a Pod's availability.

Kubernetes distinguishes between voluntary and involuntary disruptions.

### Voluntary disruptions

These occur when an administrator or automation intentionally initiates an operation that may terminate or evict Pods.

Examples:

- Draining a node for maintenance
- Cluster upgrade workflows that evict Pods
- Deleting a workload through an operation that uses the Eviction API

### Involuntary disruptions

These occur without an intentional Pod eviction operation.

Examples:

- Worker node hardware failure
- Unexpected operating system crash
- Kernel panic
- Out-of-memory termination
- Node becoming unavailable

| Voluntary | Involuntary |
|---|---|
| Planned node drain | Node crash |
| Maintenance eviction | Hardware failure |
| Supported upgrade eviction | Unexpected power loss |
| Can be constrained by a PDB | Not prevented by a PDB |

A manual force deletion or an operation that bypasses the Eviction API may not respect a PDB.

## 3. How Does a PDB Work?

Suppose an application has five replicas and a PDB requires at least four Pods to remain available.

The cluster can normally allow one matching Pod to be voluntarily evicted at a time, assuming all five Pods are currently healthy and available.

```
Application replicas: 5

Before eviction:
[Pod 1] [Pod 2] [Pod 3] [Pod 4] [Pod 5]
 Available: 5

PDB:
Minimum available = 4

After one permitted eviction:
[Pod 1] [Pod 2] [Pod 3] [Pod 4] [Evicted]
 Available: 4
```

If another eviction would violate the budget, the Eviction API can reject or delay it until sufficient availability is restored.

Important: A PDB controls disruption admission; it does not create replacement Pods itself. A Deployment or StatefulSet is responsible for maintaining the desired workload.

## 4. PDB YAML – Using minAvailable

The `minAvailable` field defines the minimum number or percentage of matching Pods that should remain available during voluntary disruptions.

### Example

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
  namespace: default
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: web
```

This PDB requires at least two selected Pods to remain available during voluntary disruptions.

For example, if three matching Pods are available, one can normally be voluntarily disrupted while preserving the minimum of two.

### Apply and verify

```bash
kubectl apply -f web-pdb.yaml
kubectl get pdb
kubectl describe pdb web-pdb
```

Check the status fields:

- EXPECTED PODS: Number of Pods matched by the selector.
- CURRENT HEALTHY: Number of matching Pods considered healthy.
- DESIRED HEALTHY: Minimum number that should remain healthy.
- ALLOWED DISRUPTIONS: Number of additional voluntary disruptions currently permitted.

## 5. PDB Using maxUnavailable

Instead of specifying the minimum number that must remain available, you can specify the maximum number that may be unavailable due to voluntary disruptions.

### Example

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
spec:
  maxUnavailable: 1
  selector:
    matchLabels:
      app: web
```

This allows at most one selected Pod to be unavailable due to voluntary disruptions, subject to the current health and disruption budget.

### minAvailable vs maxUnavailable

| Field | Meaning |
|---|---|
| `minAvailable: 3` | At least three matching Pods should remain available |
| `maxUnavailable: 1` | At most one matching Pod may be unavailable due to voluntary disruptions |

Important: Specify either `minAvailable` or `maxUnavailable` in a PDB, not both.

Percentages are also supported. For example, `minAvailable: "75%"` specifies a minimum healthy fraction, subject to Kubernetes' integer rounding rules.

## 6. Complete Practical Lab – Deployment + PDB

### Step 1: Create a Deployment

Save as `web-deployment.yaml`.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
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
        - name: nginx
          image: nginx:stable
          ports:
            - containerPort: 80
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 2
            periodSeconds: 5
```

Apply:

```bash
kubectl apply -f web-deployment.yaml
kubectl rollout status deployment/web
kubectl get pods -l app=web
```

Wait until all three replicas are Ready.

### Step 2: Create the PDB

Save as `web-pdb.yaml`.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: web
```

Apply:

```bash
kubectl apply -f web-pdb.yaml
kubectl get pdb web-pdb
kubectl describe pdb web-pdb
```

With three healthy replicas and `minAvailable: 2`, the budget should normally permit one additional voluntary disruption.

### Step 3: Test with a Pod eviction

Identify a node running a web Pod:

```bash
kubectl get pods -l app=web -o wide
```

Drain a suitable worker node in a dedicated test cluster:

```bash
kubectl drain <worker-node-name> --ignore-daemonsets
```

The drain attempts to evict eligible Pods using the Eviction API. The PDB can limit how many selected Pods are evicted while availability recovers.

Lab caution: Do not drain a production node without an approved maintenance plan. `kubectl drain` can be blocked by PDBs, DaemonSets, local storage, or other constraints. A successful drain normally leaves the node unschedulable until you uncordon it.

### Step 4: Verify

```bash
kubectl get pdb web-pdb
kubectl get pods -l app=web
kubectl get nodes
```

If you drained a node and maintenance is complete:

```bash
kubectl uncordon <worker-node-name>
```

### Step 5: Cleanup

```bash
kubectl delete pdb web-pdb
kubectl delete deployment web
```

## 7. What Happens If All Pods Are Not Healthy?

Suppose a Deployment has three replicas, but only two are Ready.

The PDB requires:

```yaml
minAvailable: 2
```

Because only two matching Pods are currently healthy, the budget may allow zero additional voluntary disruptions.

If an administrator tries to drain a node hosting one of those healthy Pods, the eviction can be blocked until availability improves.

Production lesson: Fix unhealthy replicas before planned maintenance. A PDB can protect availability, but it cannot compensate for an application that is already unhealthy or lacks sufficient replicas.

## 8. PDB and Deployment Are Different

| Feature | Deployment | PodDisruptionBudget |
|---|---|---|
| Maintains replica count | Yes, through ReplicaSets | No |
| Recreates deleted Pods | Yes, through ReplicaSets | No |
| Performs rolling updates | Yes | No |
| Constrains supported voluntary evictions | No, not by itself | Yes |
| Defines availability requirements during disruption | Not its primary purpose | Yes |

A Deployment and PDB are commonly used together in production.

## 9. Common PDB Mistakes

### Mistake 1: Selecting the wrong Pods

If the PDB selector does not match the intended application labels, it does not protect the intended workload.

Check:

```bash
kubectl get pods --show-labels
kubectl describe pdb web-pdb
```

### Mistake 2: Setting minAvailable equal to replicas

For three replicas:

```yaml
minAvailable: 3
```

If all three are healthy, this normally allows no voluntary disruption. Node drains may remain blocked until the policy or workload state changes.

### Mistake 3: Using a single replica with minAvailable: 1

A single healthy Pod with a budget requiring one healthy Pod normally has zero disruption allowance. Planned node maintenance may be blocked until a suitable replacement or revised availability policy is available.

### Mistake 4: Assuming PDB prevents all failures

A PDB does not prevent involuntary disruptions such as a node crash. Maintain multiple replicas, suitable topology spread, readiness probes, and sufficient capacity.

### Mistake 5: Assuming PDB guarantees a successful upgrade

A PDB may constrain eviction-based upgrade workflows. If the budget cannot be satisfied, an upgrade or drain may stall until availability improves or the operational policy is changed.

## 10. Advanced Notes

### PDB and rolling updates

A Deployment uses its own rollout strategy, such as `maxUnavailable` and `maxSurge`, to control rolling updates.

A PDB is a separate mechanism for supported voluntary disruptions. Do not treat a PDB as a replacement for Deployment rollout settings.

### PDB and node failure

If a node crashes, its Pods may become unavailable regardless of the PDB. Kubernetes can then work to recreate managed Pods on eligible healthy nodes, subject to storage, scheduling, and controller behavior.

### PDB and multiple maintenance operations

The number of permitted disruptions depends on current Pod health, desired availability, and disruptions already in progress. Multiple maintenance operations may contend for the same budget.

### OpenShift

OpenShift supports Kubernetes PDB resources and the same fundamental disruption concepts.

Commands:

```bash
oc get pdb
oc describe pdb <pdb-name>
oc get pods -o wide
oc get events --sort-by=.metadata.creationTimestamp
```

## 11. Interview Questions

**Q1. What is a PodDisruptionBudget?**
A PDB defines the minimum availability or maximum unavailability allowed for selected Pods during supported voluntary disruptions.

**Q2. What are voluntary disruptions?**
Planned actions such as node draining and certain upgrade operations that use the Eviction API.

**Q3. What are involuntary disruptions?**
Unexpected events such as node crashes, hardware failures, or certain resource-pressure failures.

**Q4. Does PDB restart failed Pods?**
No. Workload controllers such as Deployments or StatefulSets manage replacement Pods.

**Q5. What is the difference between minAvailable and maxUnavailable?**
`minAvailable` sets the minimum number or percentage of healthy Pods to preserve. `maxUnavailable` sets the maximum number or percentage that may be unavailable due to voluntary disruptions.

**Q6. Why can kubectl drain get stuck?**
A PDB may prevent further evictions without violating the configured availability budget. Other causes include local storage, DaemonSets, or workload constraints.

**Q7. Can a PDB prevent a node crash?**
No. A PDB does not prevent involuntary failures.

**Q8. What happens when all replicas are unhealthy?**
The budget may allow no voluntary disruption. The workload's health and availability should be restored before proceeding with maintenance where possible.

**Q9. Does PDB replace Deployment maxUnavailable?**
No. Deployment rollout strategy and PDB govern different mechanisms.

**Q10. How do you troubleshoot a blocked eviction?**
Check `kubectl describe pdb`, matching Pod labels, healthy replica counts, pending disruptions, and the Events from the drain operation.

## 12. Quick Revision

| Term | Meaning |
|---|---|
| PDB | Defines availability constraints for supported voluntary disruptions |
| Voluntary disruption | Planned eviction or maintenance operation |
| Involuntary disruption | Unexpected failure |
| minAvailable | Minimum healthy matching Pods |
| maxUnavailable | Maximum unavailable matching Pods |
| `kubectl drain` | Evicts eligible Pods for node maintenance |
| `kubectl uncordon` | Makes the node schedulable again |
| Deployment | Manages replicas and rollout strategy |

Final takeaway: For production high availability, use a suitable replica count, readiness probes, topology-aware placement, and a PDB. Before draining a node, verify that enough healthy replicas and sufficient cluster capacity exist to complete maintenance safely.
