# Kubernetes Pod Preemption – Complete Notes

## 1. What Is Preemption?

Pod preemption is a Kubernetes scheduling mechanism that allows a higher-priority Pod to take the place of lower-priority Pods when cluster resources are insufficient.

When a high-priority Pod cannot be scheduled because there are not enough resources on any suitable node, the Kubernetes scheduler may select a node and evict lower-priority Pods to make room.

### Example

- Node capacity: 4 CPU cores.
- Existing low-priority Pods consume all 4 CPU cores.
- A new high-priority Pod requests 2 CPU cores.
- The scheduler may preempt lower-priority Pods to free enough CPU resources.
- The high-priority Pod can then be scheduled if all other scheduling requirements are satisfied.

Important: Preemption does not increase node capacity. It makes existing resources available by removing selected Pods.

## 2. Why Is Preemption Required?

Preemption helps Kubernetes schedule critical workloads when cluster resources are limited.

Common use cases:

- Critical production applications.
- Monitoring and infrastructure workloads.
- High-priority batch processing.
- Time-sensitive workloads.
- Applications that must be scheduled ahead of less important workloads.

Preemption is not guaranteed. The scheduler must find a suitable node, and the required resources and scheduling constraints must allow the higher-priority Pod to run.

## 3. Kubernetes Priority and PriorityClass

Kubernetes uses a PriorityClass to assign priority values to Pods.

A higher numerical priority value means a higher scheduling priority.

| Priority value | Meaning |
|---|---|
| 1000000 | Higher priority |
| 1000 | Medium priority |
| 100 | Lower priority |
| 0 | Default priority when no other priority is assigned |

These are example values, not fixed Kubernetes priority levels.

A PriorityClass is a cluster-scoped resource. You create it once and reference it in Pod specifications.

### Create a PriorityClass

File: `priorityclass.yaml`

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "Priority class for critical applications"
```

Apply it:

```bash
kubectl apply -f priorityclass.yaml
```

Verify:

```bash
kubectl get priorityclass
kubectl describe priorityclass high-priority
```

### Important fields

- `apiVersion`: API version for PriorityClass.
- `kind`: Resource type.
- `metadata.name`: Name referenced by Pods.
- `value`: Numerical scheduling priority.
- `globalDefault`: If true, this class becomes the default priority class for Pods that do not specify a class, provided no other global default is configured.
- `description`: Explains the purpose of the class.

Only one PriorityClass should be configured as the global default. If none is configured, Pods without a priority class normally receive priority 0.

## 4. Create Low-Priority and High-Priority Pods

### Step 1: Create a low-priority class

File: `low-priority.yaml`

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority
value: 100
globalDefault: false
description: "Priority class for non-critical workloads"
```

Apply:

```bash
kubectl apply -f low-priority.yaml
```

### Step 2: Create a low-priority Pod

File: `low-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: low-pod
spec:
  priorityClassName: low-priority
  containers:
    - name: app
      image: nginx:1.28
      resources:
        requests:
          cpu: "500m"
          memory: "128Mi"
```

Apply:

```bash
kubectl apply -f low-pod.yaml
```

### Step 3: Create a high-priority Pod

File: `high-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: high-pod
spec:
  priorityClassName: high-priority
  containers:
    - name: app
      image: nginx:1.28
      resources:
        requests:
          cpu: "2"
          memory: "512Mi"
```

Apply:

```bash
kubectl apply -f high-pod.yaml
```

Check scheduling:

```bash
kubectl get pods -o wide
kubectl describe pod high-pod
kubectl get events --sort-by=.metadata.creationTimestamp
```

Lab note: These Pods demonstrate priority configuration, but they will not necessarily trigger preemption on a normal cluster. To test actual preemption, create controlled resource contention on a suitable test cluster. Do not deliberately disrupt production workloads.

## 5. How Preemption Works

The scheduler generally follows this process:

1. A Pod enters the scheduling queue.
2. The scheduler attempts to find a suitable node.
3. If no node can accommodate the Pod, the scheduler may consider preemption.
4. It evaluates nodes and identifies lower-priority Pods that could be removed.
5. It chooses a suitable candidate node and victim Pods.
6. Victim Pods are terminated through the relevant deletion process.
7. Once sufficient resources become available, the higher-priority Pod may be scheduled.

The exact outcome depends on resource requests, node capacity, affinity rules, taints and tolerations, PodDisruptionBudgets, priorities, and other scheduling constraints.

Important: A high-priority Pod does not automatically preempt every lower-priority Pod. The scheduler must determine that preemption can help the Pod fit on a node.

## 6. What Is a Preemption Victim?

A preemption victim is a lower-priority Pod selected for removal to make resources available for a higher-priority Pod.

For example:

| Pod | Priority | CPU request |
|---|---|---|
| app-a | 100 | 1 CPU |
| app-b | 100 | 1 CPU |
| app-c | 100 | 1 CPU |
| critical-app | 1000000 | 2 CPU |

If the node has no available CPU capacity, the scheduler may select one or more lower-priority Pods as victims.

The scheduler considers which victims can free sufficient resources while satisfying the scheduling requirements. It does not simply choose Pods by name or always remove the single lowest-priority Pod.

## 7. Preemption vs Pod Eviction

These terms are related but have different meanings.

| Feature | Preemption | Eviction |
|---|---|---|
| Purpose | Make room for a higher-priority Pod | Remove a Pod from a node or cluster |
| Typical trigger | Scheduling resource shortage | Maintenance, resource pressure, or another eviction request |
| Initiator | Scheduler selects lower-priority victims | Eviction API, kubelet, or another controller/process |
| Priority-based | Yes, in scheduler preemption | Not necessarily |
| PDB behavior | Scheduler preemption has special PDB-related handling | Eviction API normally checks PDBs; node-pressure eviction does not respect them |
| Typical example | Critical Pod needs CPU currently occupied by lower-priority Pods | `kubectl drain` or node memory pressure |

Remember: Preemption is a scheduling decision; eviction is the process or mechanism by which a Pod is removed.

## 8. Preemption vs Node-Pressure Eviction

| Feature | Preemption | Node-pressure eviction |
|---|---|---|
| Main reason | Higher-priority Pod cannot be scheduled | Node is under resource pressure |
| Main component | Scheduler | Kubelet |
| Priority | Central to victim selection | One factor in eviction ranking |
| PDB protection | Do not assume PDB guarantees protection from scheduler preemption | PDB does not protect against kubelet node-pressure eviction |
| Typical signals | Failed scheduling and preemption events | MemoryPressure, DiskPressure, PIDPressure |

A node can be Ready and still experience memory or disk pressure.

## 9. What Is preemptionPolicy?

A Pod can control whether it is allowed to preempt lower-priority Pods by setting `preemptionPolicy`.

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: critical-app
spec:
  priorityClassName: high-priority
  preemptionPolicy: Never
  containers:
    - name: app
      image: nginx:1.28
      resources:
        requests:
          cpu: "1"
          memory: "256Mi"
```

Possible values:

- `PreemptLowerPriority` — the default behavior. The Pod can trigger preemption of lower-priority Pods if needed.
- `Never` — the Pod cannot preempt other Pods, even if it has a high priority.

A high-priority Pod with `preemptionPolicy: Never` can still be scheduled ahead of lower-priority Pods waiting in the scheduling queue. However, it must wait for resources to become available naturally if no suitable node has enough capacity.

Important: `preemptionPolicy: Never` prevents the Pod from initiating preemption. It does not make the Pod immune to being preempted by another higher-priority Pod.

## 10. Does Preemption Respect PodDisruptionBudget?

A PodDisruptionBudget (PDB) describes the number of voluntary disruptions allowed for a group of Pods.

During scheduler preemption, Kubernetes considers PDBs when selecting victims. It attempts to avoid violating a PDB when possible. However, a PDB is not an absolute guarantee against scheduler preemption; if no suitable alternative exists, preemption may still proceed in a way that violates a PDB.

### Check PDBs

```bash
kubectl get pdb -A
kubectl describe pdb -n default
```

### Example PDB

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: app-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: my-app
```

This asks Kubernetes to keep at least two matching Pods available during supported voluntary disruptions. It does not guarantee that the application will always have two healthy replicas during every failure or preemption scenario.

## 11. Troubleshooting Preemption

### Check Pod status

```bash
kubectl get pods -A -o wide
```

### Inspect the unscheduled Pod

```bash
kubectl describe pod critical-app
```

Look for events such as:

- FailedScheduling
- Preemption
- Insufficient cpu
- Insufficient memory
- Affinity or taint-related scheduling failures

The exact event wording varies by Kubernetes version and situation.

### Check Pod priority

```bash
kubectl get pod critical-app \
  -o jsonpath='{.spec.priorityClassName}{"\n"}{.spec.priority}{"\n"}'
```

### Check PriorityClasses

```bash
kubectl get priorityclass
kubectl describe priorityclass high-priority
```

### Check node capacity and allocatable resources

```bash
kubectl describe nodes
kubectl top nodes
kubectl top pods -A
```

`kubectl top` requires a working Metrics Server or compatible metrics pipeline. Scheduling decisions are primarily based on resource requests and allocatable capacity, not simply the current CPU percentage shown by `top`.

### Check recent events

```bash
kubectl get events -A \
  --sort-by=.metadata.creationTimestamp
```

### Common causes when preemption does not help

1. No lower-priority Pods can be removed to free enough resources.
2. The Pod's CPU or memory request is larger than any suitable node can provide.
3. Node affinity, Pod affinity, topology spread, taints, or other constraints prevent placement.
4. Removing victims would not resolve the scheduling constraint.
5. The Pod has `preemptionPolicy: Never`.
6. Resources are temporarily unavailable while victim Pods terminate.
7. The cluster has insufficient total capacity.

The correct response is to inspect the scheduler events and the Pod's scheduling requirements rather than assuming that priority alone guarantees scheduling.

## 12. OpenShift-Specific Notes

OpenShift uses Kubernetes scheduling concepts, including PriorityClasses and Pod preemption.

Useful commands:

```bash
oc get priorityclass
oc describe priorityclass high-priority
oc get pods -A -o wide
oc describe pod critical-app -n my-project
oc get events -n my-project \
  --sort-by=.metadata.creationTimestamp
oc adm top nodes
oc adm top pods -n my-project
```

Replace `my-project` and `critical-app` with your actual project and Pod names.

In OpenShift, also inspect project quotas, LimitRanges, node selectors, tolerations, and security or policy constraints where relevant. Priority does not override those requirements.

## 13. Interview Questions and Answers

**Q1. What is Pod preemption?**
Pod preemption is a scheduler mechanism that allows a higher-priority Pod to displace lower-priority Pods when resources are insufficient and removing those Pods can make scheduling possible.

**Q2. Which Kubernetes component performs preemption?**
The Kubernetes scheduler.

**Q3. How do you configure Pod priority?**
Create a PriorityClass and specify its name in the Pod's `priorityClassName` field.

**Q4. Does a high-priority Pod always get scheduled immediately?**
No. Priority affects scheduling order and can allow preemption, but resource capacity and other scheduling constraints must still be satisfied.

**Q5. What is a preemption victim?**
A lower-priority Pod selected for removal to free resources for a higher-priority Pod.

**Q6. What is the difference between preemption and node-pressure eviction?**
Preemption is initiated by the scheduler to help schedule a higher-priority Pod. Node-pressure eviction is initiated by the kubelet to respond to node resource pressure.

**Q7. What does `preemptionPolicy: Never` do?**
It prevents a Pod from preempting other Pods. The Pod can still have high scheduling priority, but may have to wait for resources to become available.

**Q8. Does a PDB completely prevent preemption?**
No. The scheduler considers PDBs when selecting victims, but a PDB is not an absolute guarantee against scheduler preemption.

**Q9. Can a Pod with high priority run if its resource request exceeds the capacity of every node?**
No. Preemption cannot create capacity beyond the physical or allocatable capacity of a suitable node.

**Q10. How do you troubleshoot a Pod that remains Pending despite having high priority?**
Check `kubectl describe pod`, scheduling events, PriorityClass, resource requests, node allocatable resources, affinity, taints, tolerations, and `preemptionPolicy`.

## 14. Quick Revision

- Preemption: Removes lower-priority Pods to make room for a higher-priority Pod.
- PriorityClass: Defines a numerical scheduling priority.
- `priorityClassName`: Assigns a PriorityClass to a Pod.
- `preemptionPolicy: PreemptLowerPriority`: Allows preemption; this is the default.
- `preemptionPolicy: Never`: Prevents the Pod from initiating preemption.
- Preemption victim: Pod selected for removal.
- Scheduler: Handles preemption decisions.
- Kubelet: Handles node-pressure eviction.
- PDB: Influences supported voluntary disruptions and is considered during scheduler preemption, but is not an absolute guarantee.
- Key diagnostic command: `kubectl describe pod <pod-name>`.

Final takeaway: Pod priority determines scheduling importance, but preemption only helps when removing lower-priority Pods can actually make the higher-priority Pod schedulable.
