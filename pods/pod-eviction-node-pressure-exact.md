# Kubernetes Pod Eviction and Node-Pressure Eviction – Complete Notes

## 1. What Is Pod Eviction?

Pod eviction means terminating a Pod through an eviction mechanism, usually to free resources or perform a planned operation.

Kubernetes lo two important eviction scenarios unnayi:

1. **Voluntary eviction:** Administrator or automation intentionally evicts a Pod, for example during node maintenance.
2. **Node-pressure eviction:** Kubelet evicts Pods because node resources are under pressure.

### Common examples

- `kubectl drain` evicts eligible Pods during node maintenance.
- Kubelet evicts Pods when memory pressure becomes critical.
- Kubelet evicts Pods when disk space or inode availability becomes critically low.
- A node can experience PID pressure when too many processes are running.

Important: Node-pressure eviction is initiated by the kubelet, not by the PodDisruptionBudget mechanism.

## 2. Voluntary Eviction vs Node-Pressure Eviction

| Feature | Voluntary eviction | Node-pressure eviction |
|---|---|---|
| Initiated by | Administrator or automation | Kubelet |
| Typical reason | Maintenance or planned disruption | Resource shortage |
| Example | `kubectl drain` | Memory or disk pressure |
| Uses Eviction API | Typically yes | Kubelet performs local eviction |
| PDB behavior | Can constrain supported eviction operations | Does not prevent node-pressure eviction |
| Main objective | Planned workload disruption | Protect node stability |

A PDB can limit supported voluntary disruptions, but it cannot prevent the kubelet from evicting Pods because of node resource pressure.

## 3. What Is Node Pressure?

Node pressure occurs when a worker node approaches a critical resource threshold.

Kubelet monitors resource signals and can evict Pods when configured eviction thresholds are crossed.

### Main node-pressure conditions

| Condition | Meaning |
|---|---|
| MemoryPressure | Available memory is critically low |
| DiskPressure | Available disk space or inodes are critically low |
| PIDPressure | Available process IDs are critically low |

These conditions can affect scheduling and workload availability.

### Check node conditions

```bash
kubectl get nodes
kubectl describe node <node-name>
kubectl get events -A --sort-by=.metadata.creationTimestamp
```

Look for conditions such as:

```
MemoryPressure
DiskPressure
PIDPressure
```

A node may report `Ready=True` while also experiencing pressure. Inspect all node conditions rather than relying on the Ready status alone.

## 4. Memory Pressure Eviction

Memory pressure occurs when the node has insufficient available memory.

### Common causes

- Applications consume too much memory.
- Containers have no realistic memory limits.
- Memory leaks.
- Too many workloads scheduled on one node.
- System daemons consume excessive memory.
- Burstable or BestEffort workloads consume memory beyond what the node can sustain.

### Example scenario

A worker node has 8 GiB of memory. Several workloads consume most of the available memory, and the kubelet reaches its memory eviction threshold.

The kubelet may evict selected Pods to protect the node.

### Troubleshooting commands

```bash
kubectl top nodes
kubectl top pods -A
kubectl describe node <node-name>
kubectl get pods -A -o wide
```

`kubectl top` requires a functioning metrics pipeline and reports usage, not every aspect of node memory availability.

### Fixes

- Identify high-memory workloads.
- Set realistic memory requests and limits.
- Investigate memory leaks and JVM heap configuration.
- Reduce excessive workload density.
- Add worker-node capacity when required.
- Review kubelet eviction settings where authorized.

Do not simply increase every Pod's memory limit. That may increase contention rather than solve the underlying capacity problem.

## 5. Disk Pressure Eviction

Disk pressure occurs when available storage space or filesystem inodes fall below applicable thresholds.

### Common causes

- Large container logs.
- Container image layers consuming disk.
- Excessive writable-layer usage.
- Disk-backed `emptyDir` volumes.
- Temporary files.
- Container runtime storage growth.
- Filesystem inode exhaustion.

### Troubleshooting

```bash
kubectl describe node <node-name>
kubectl get events -A --sort-by=.metadata.creationTimestamp
kubectl describe pod <pod-name>
```

If authorized to access the worker node, inspect its filesystem:

```bash
df -h
df -i
du -xhd1 /var/lib/kubelet 2>/dev/null
du -xhd1 /var/log 2>/dev/null
```

Actual runtime and log paths vary by distribution and configuration. Use read-only inspection first, and do not delete runtime-managed files manually.

### Fixes

- Rotate and retain logs appropriately.
- Remove unused images through supported runtime or platform procedures.
- Investigate large temporary files.
- Review ephemeral-storage requests and limits.
- Expand disk capacity where necessary.
- Restore filesystem and inode headroom.

Important: Disk-pressure eviction is not limited to Pods that exceed an explicit ephemeral-storage limit. Node-level filesystem pressure can also trigger eviction.

## 6. PID Pressure Eviction

PID pressure occurs when the node has too few available process IDs.

### Common causes

- Applications create too many processes.
- Fork bombs or runaway subprocess creation.
- Misconfigured worker pools.
- Containers fail to clean up child processes.
- Excessive workloads on the node.

### Troubleshooting

```bash
kubectl describe node <node-name>
kubectl get events -A --sort-by=.metadata.creationTimestamp
```

With authorized node access:

```bash
ps -e --no-headers | wc -l
cat /proc/sys/kernel/pid_max
```

The host process count and PID limits help diagnose the problem, but kubelet's actual eviction decisions depend on its configured signals and thresholds.

### Fixes

- Investigate processes and thread creation.
- Correct runaway workloads.
- Apply appropriate process limits where supported.
- Reduce workload density or add capacity.

## 7. How Kubelet Selects Pods for Node-Pressure Eviction

Kubelet does not select Pods based only on their names or replica counts.

For memory-pressure eviction, the important considerations include:

1. Whether the Pod's resource usage exceeds its requests.
2. Pod priority.
3. How much the Pod's usage exceeds its requests.

Higher-priority workloads are generally better protected than lower-priority workloads when other relevant factors are comparable.

For disk-pressure eviction, kubelet also considers ephemeral-storage usage relative to requests, priority, and the specific resource signal involved.

Important: Pod QoS class can be useful for understanding eviction risk, but QoS class alone is not a complete description of kubelet eviction ordering.

## 8. Pod QoS Classes and Eviction Risk

Kubernetes assigns Pods a Quality of Service (QoS) class based on container resource requests and limits.

### BestEffort

A Pod is BestEffort when none of its containers has CPU or memory requests or limits configured.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: besteffort-demo
spec:
  containers:
    - name: app
      image: nginx:stable
```

BestEffort Pods have no declared CPU or memory requests or limits and are generally more vulnerable during node-pressure eviction.

### Burstable

A Pod is Burstable when it does not qualify as Guaranteed and at least one container has a CPU or memory request or limit.

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
```

Burstable Pods may use resources beyond their requests, subject to limits and node capacity.

### Guaranteed

A Pod qualifies as Guaranteed when every container has CPU and memory requests and limits specified, with each container's request equal to its corresponding limit.

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

Guaranteed Pods generally receive stronger protection than BestEffort Pods during node-pressure eviction when other relevant factors are comparable.

Important: Guaranteed QoS does not mean a Pod can never be evicted or OOM-killed.

Check a Pod's QoS class:

```bash
kubectl get pod <pod-name> \
  -o jsonpath='{.status.qosClass}{"\n"}'
```

## 9. Pod Priority and Preemption

Pod Priority is another factor in scheduling and eviction behavior.

A higher-priority Pod can be scheduled ahead of lower-priority workloads, and the scheduler may preempt lower-priority Pods to make room when necessary.

### Important distinction

- **Scheduler preemption:** The scheduler may terminate lower-priority Pods to make room for a higher-priority Pod.
- **Node-pressure eviction:** Kubelet evicts Pods to protect the node from resource exhaustion.
- **PDB:** Can constrain supported voluntary evictions, but does not prevent node-pressure eviction.

These are different mechanisms with different purposes.

## 10. What Happens After Eviction?

When a Pod is evicted, its lifecycle ends and its status may show `Failed` with a reason such as `Evicted`.

Whether another Pod is created depends on its controller.

| Workload | Typical recovery |
|---|---|
| Standalone Pod | No automatic replacement |
| Deployment | ReplicaSet attempts to maintain desired replicas |
| StatefulSet | Controller attempts to restore the desired Pod identity |
| Job | Job controller may retry according to its policy |
| DaemonSet | Controller attempts to restore a Pod on eligible nodes |

A replacement Pod may be scheduled onto another suitable node, but recovery can be delayed by resource shortages, affinity constraints, taints, or storage attachment restrictions.

For stateful applications, verify volume behavior and data persistence before assuming recovery is complete.

## 11. What Is kubectl drain?

`kubectl drain` is commonly used to prepare a node for maintenance.

It marks the node unschedulable and attempts to evict eligible Pods.

### Commands

```bash
kubectl get nodes
kubectl drain <node-name> --ignore-daemonsets
kubectl get pods -o wide
kubectl uncordon <node-name>
```

### Important points

- `drain` is an administrative maintenance operation, not node-pressure eviction.
- Supported Pod evictions can be blocked by a PDB.
- DaemonSet Pods are not removed by the usual drain behavior.
- Local storage and other constraints may require additional handling.
- `uncordon` allows normal scheduling again after maintenance.

Do not use `--force` or `--disable-eviction` casually. The latter bypasses the Eviction API and can bypass PDB protections.

## 12. Kubelet Eviction Thresholds

Kubelet uses eviction signals and thresholds to decide when node resources are under pressure.

Signals can include:

- Available memory
- Available node filesystem space
- Available image filesystem space, where applicable
- Available filesystem inodes
- Available process IDs

Thresholds can be configured as `evictionHard` and `evictionSoft`.

### Hard eviction threshold

A hard threshold triggers eviction without the soft-threshold grace period once the applicable condition is reached.

### Soft eviction threshold

A soft threshold can trigger eviction after the configured condition persists for its configured grace period.

Exact behavior depends on kubelet configuration, resource signals, and the Kubernetes distribution.

Inspecting or changing kubelet settings usually requires node-level administrative access and should follow the cluster platform's supported procedures.

## 13. Practical Troubleshooting Scenario

### Scenario

A worker node reports `MemoryPressure=True`. Several Pods are evicted, and the application has fewer Ready replicas than expected.

### Step 1: Inspect node conditions

```bash
kubectl describe node <node-name>
```

### Step 2: Inspect evicted Pods and events

```bash
kubectl get pods -A
kubectl get events -A --sort-by=.metadata.creationTimestamp
```

Look for eviction messages and identify the resource signal.

### Step 3: Check resource usage

```bash
kubectl top nodes
kubectl top pods -A
```

### Step 4: Inspect workload resource settings

```bash
kubectl describe pod <pod-name>
kubectl get deployment <deployment-name> -o yaml
```

### Step 5: Correct the root cause

- Fix memory leaks or excessive memory usage.
- Set realistic requests and limits.
- Add node capacity if necessary.
- Review workload distribution and priority.
- Verify that controller-managed replicas return to Ready.

### Step 6: Verify recovery

```bash
kubectl get nodes
kubectl get pods -A -o wide
kubectl get deployment -A
```

Confirm that node pressure has cleared, expected replicas are available, and application requests succeed.

## 14. Kubernetes and OpenShift Commands

The same core commands apply to both platforms.

| Task | Kubernetes | OpenShift |
|---|---|---|
| Inspect nodes | `kubectl get nodes` | `oc get nodes` |
| Inspect node pressure | `kubectl describe node <node>` | `oc describe node <node>` |
| Check Pods | `kubectl get pods -A` | `oc get pods -A` |
| Inspect events | `kubectl get events -A` | `oc get events -A` |
| Check resource usage | `kubectl top nodes` | `oc adm top nodes` |
| Check Pod usage | `kubectl top pods -A` | `oc adm top pods -A` |
| Drain node | `kubectl drain <node> --ignore-daemonsets` | `oc adm drain <node> --ignore-daemonsets` |
| Resume scheduling | `kubectl uncordon <node>` | `oc adm uncordon <node>` |

OpenShift node access and kubelet configuration should follow the platform's supported administrative workflows.

## 15. Common Interview Questions

**Q1. What is Pod eviction?**
Pod eviction is the termination of a Pod through an eviction mechanism, commonly for planned maintenance or to protect a node from resource exhaustion.

**Q2. What is node-pressure eviction?**
It is kubelet-initiated eviction when node resources such as memory, disk space, inodes, or PIDs reach critical thresholds.

**Q3. Can a PDB prevent node-pressure eviction?**
No. PDBs govern supported voluntary disruptions and do not prevent kubelet node-pressure eviction.

**Q4. What is the difference between drain and node-pressure eviction?**
Drain is an administrative maintenance operation that uses the Eviction API for eligible Pods. Node-pressure eviction is initiated by kubelet to protect the node.

**Q5. Which QoS class is generally most vulnerable to node-pressure eviction?**
BestEffort Pods are generally more vulnerable than Burstable and Guaranteed Pods, but actual eviction decisions depend on resource usage, requests, priority, and the resource signal involved.

**Q6. Does Guaranteed QoS prevent eviction?**
No. Guaranteed QoS provides no absolute guarantee against eviction or OOM termination.

**Q7. What happens when a Deployment Pod is evicted?**
The Deployment's ReplicaSet attempts to maintain the desired replica count by creating a replacement Pod.

**Q8. What is MemoryPressure?**
A node condition indicating critically low available memory according to the node's pressure detection logic.

**Q9. What is DiskPressure?**
A node condition indicating low available storage space or inodes according to configured signals and thresholds.

**Q10. What are hard and soft eviction thresholds?**
Hard thresholds trigger eviction without the soft grace period. Soft thresholds allow a configured condition to persist for a specified grace period before eviction.

**Q11. Does kubectl drain respect PDBs?**
Normal drain operations use the Eviction API for eligible Pods and can be blocked by PDBs.

**Q12. How do you troubleshoot an Evicted Pod?**
Inspect the Pod description, Events, node conditions, resource usage, and workload requests and limits. Identify the specific pressure signal before making changes.

## 16. Quick Revision

| Topic | Key point |
|---|---|
| Voluntary eviction | Planned disruption |
| Node-pressure eviction | Kubelet protects the node |
| MemoryPressure | Memory availability is critically low |
| DiskPressure | Disk space or inode availability is critically low |
| PIDPressure | Process ID availability is critically low |
| BestEffort | No CPU or memory requests/limits |
| Burstable | Some CPU or memory requests/limits configured |
| Guaranteed | Every container has matching CPU and memory requests and limits |
| PDB | Constrains supported voluntary evictions |
| `kubectl drain` | Prepares a node for maintenance |
| `kubectl uncordon` | Restores normal scheduling |
| evictionHard | Hard eviction thresholds |
| evictionSoft | Thresholds with a grace period |

Final takeaway: When Pods are evicted, identify whether the cause is planned maintenance or node resource pressure. Check node conditions, Events, resource usage, QoS, priority, and workload controllers. Fix the underlying capacity or application issue rather than repeatedly deleting Pods.
