# Kubernetes Pod Scheduling Gates – Complete Notes

## 1. What Are Pod Scheduling Gates?

Scheduling gates allow you to prevent a newly created Pod from being scheduled until specified conditions or prerequisites have been handled by an external process.

A Pod with scheduling gates remains unscheduled while its `spec.schedulingGates` list contains one or more gates. Kubernetes does not schedule the Pod until those gates are removed.

Scheduling gates are useful when a workload must wait for an external controller or orchestration process before entering the normal scheduling process.

Examples:

- A batch workload must wait for resource allocation approval.
- A custom controller must complete workload preparation.
- A multi-step deployment workflow must release Pods for scheduling in a controlled order.
- An external queue or admission workflow must authorize a workload before it consumes node resources.

Important: Kubernetes does not automatically evaluate whether an arbitrary business condition has been met. An authorized controller or operator must remove the scheduling gate when the Pod is allowed to proceed.

## 2. Why Are Scheduling Gates Required?

Normally, once a Pod is accepted by the API server, it can enter the scheduler's queue.

Scheduling gates let an administrator or controller hold a Pod back before the scheduler attempts to place it on a node.

This can help prevent workloads from consuming scheduling capacity before they are ready to proceed.

For example, a controller could hold a group of Pods until a deployment approval is complete, then remove their gates so the scheduler can process them.

## 3. How Scheduling Gates Work

1. **Pod created.**
2. **Scheduling gate present:** The Pod is held back from scheduling.
3. **External controller removes the gate:** The Pod becomes eligible for normal scheduling.
4. **Scheduler evaluates the Pod** and selects a node if possible.

Removing a scheduling gate does not guarantee immediate scheduling. The scheduler must still find a suitable node based on resource requests, affinity rules, taints, tolerations, and other scheduling constraints.

## 4. Scheduling Gates YAML Example

File: `scheduling-gate.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: scheduling-gate-demo
spec:
  schedulingGates:
    - name: example.com/approval-required
  containers:
    - name: nginx
      image: nginx:1.28
      resources:
        requests:
          cpu: "100m"
          memory: "64Mi"
```

Apply the manifest:

```bash
kubectl apply -f scheduling-gate.yaml
```

Check the Pod:

```bash
kubectl get pod scheduling-gate-demo -o wide
kubectl describe pod scheduling-gate-demo
```

The Pod will remain unscheduled while the gate is present. Its node assignment is normally empty.

### Important fields

| Field | Purpose |
|---|---|
| `spec.schedulingGates` | List of gates that block scheduling |
| `name` | Name identifying a scheduling gate |
| `containers` | Application containers |
| `resources.requests` | Resource requirements evaluated after the Pod becomes eligible for scheduling |

The gate name is an identifier. Kubernetes does not interpret `example.com/approval-required` as an approval mechanism by itself.

## 5. How to Check Scheduling Gates

### Check the Pod status

```bash
kubectl get pod scheduling-gate-demo -o wide
```

A Pod blocked by scheduling gates commonly shows `SchedulingGated` as its status reason in the `kubectl get` output, although presentation can vary by client version.

### Inspect the complete Pod specification

```bash
kubectl get pod scheduling-gate-demo -o yaml
```

Look for:

```yaml
spec:
  schedulingGates:
    - name: example.com/approval-required
```

### Inspect Pod conditions and events

```bash
kubectl describe pod scheduling-gate-demo
kubectl get events \
  --sort-by=.metadata.creationTimestamp
```

A scheduling-gated Pod is different from a Pod that has already been considered by the scheduler but cannot fit on any node.

## 6. Removing a Scheduling Gate

A scheduling gate must be removed from the Pod specification before the scheduler can schedule the Pod.

In normal production workflows, an authorized controller removes the gate after checking the required prerequisites.

For a simple lab, you can use a controller or a controlled API update to remove the entry from `spec.schedulingGates`.

Important: Scheduling gates are designed to be removed, not added, after Pod creation. Kubernetes restricts updates to this field: existing gates may be removed, but new gates cannot generally be added to a Pod after it has been created.

### Conceptual example

Before:

```yaml
spec:
  schedulingGates:
    - name: example.com/approval-required
```

After the gate is removed:

```yaml
spec:
  containers:
    - name: nginx
      image: nginx:1.28
```

Once the last gate is removed, the Pod becomes eligible for normal scheduling.

### Verify the result

```bash
kubectl get pod scheduling-gate-demo -o wide
kubectl get pod scheduling-gate-demo -o yaml
kubectl describe pod scheduling-gate-demo
```

If the gate is removed but the Pod remains Pending, check the scheduler events and resource requirements. The Pod may now be eligible but unable to fit on a node.

## 7. Scheduling Gates vs Readiness Gates

These are different features that operate at different stages of a Pod's lifecycle.

| Feature | Scheduling gates | Readiness gates |
|---|---|---|
| YAML field | `spec.schedulingGates` | `spec.readinessGates` |
| Main purpose | Delay scheduling | Delay Pod readiness |
| Scheduler assigns a node? | Not while gates remain | Scheduling can proceed normally |
| Container can start? | Not before normal scheduling and startup | Yes, if scheduled and otherwise able to start |
| External controller action | Removes scheduling gates | Sets the custom Pod condition to `True` |
| Typical use case | Approval before placement | External readiness before traffic |

Interview shortcut:

- Scheduling gate = Do not schedule this Pod yet.
- Readiness gate = Do not consider this Pod Ready yet.

## 8. Scheduling Gates vs Pod Priority and Preemption

| Feature | Scheduling gates | Priority and preemption |
|---|---|---|
| Main purpose | Hold back a Pod before scheduling | Influence scheduling order and potentially free resources |
| Scheduler evaluates placement? | Not while a gate remains | Yes, when the Pod is eligible |
| Frees node resources? | No | Preemption may remove lower-priority Pods |
| Typical configuration | `spec.schedulingGates` | PriorityClass and `priorityClassName` |

A high-priority Pod with a scheduling gate is still held back. High priority does not bypass the gate.

## 9. Practical Lab

This lab demonstrates the behavior of a gated Pod.

### Step 1: Create the Pod

Save the YAML as `scheduling-gate.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gate-lab
spec:
  schedulingGates:
    - name: lab.example.com/wait
  containers:
    - name: app
      image: nginx:1.28
```

Apply it:

```bash
kubectl apply -f scheduling-gate.yaml
```

### Step 2: Verify that it is gated

```bash
kubectl get pod gate-lab -o wide
kubectl get pod gate-lab -o yaml
kubectl describe pod gate-lab
```

Confirm that `spec.schedulingGates` contains `lab.example.com/wait` and that the Pod has not been assigned a node.

### Step 3: Remove the gate

For a controlled lab, use an authorized API update that removes the gate entry. In production, let the responsible controller perform this operation.

After removal, verify:

```bash
kubectl get pod gate-lab -o wide
kubectl describe pod gate-lab
```

The scheduler should now be able to evaluate the Pod. If the cluster has sufficient resources and all scheduling constraints are satisfied, the Pod can be assigned to a node and start.

### Lab cleanup

```bash
kubectl delete pod gate-lab
```

## 10. Common Troubleshooting

### Problem 1: Pod remains unscheduled

Check:

```bash
kubectl get pod <pod-name> -o yaml
```

If `spec.schedulingGates` is not empty, identify the controller or workflow responsible for removing the gates.

### Problem 2: Gate was removed, but the Pod is still Pending

Check:

```bash
kubectl describe pod <pod-name>
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl describe nodes
```

Possible causes:

- Insufficient CPU or memory.
- Node affinity or selector mismatch.
- Untolerated taints.
- Storage provisioning or attachment issues.
- Other scheduling constraints.

### Problem 3: Controller does not remove the gate

Check:

- Controller logs and status.
- Whether the prerequisite or approval has completed.
- Whether the controller watches the correct namespace and Pods.
- Whether its ServiceAccount has permission to update Pod specifications.
- Whether the controller handles errors when updating the Pod.

### Problem 4: Trying to add a gate to an existing Pod

Kubernetes restricts scheduling gate updates. Design the workflow so the required gates are present when the Pod is created. Do not rely on adding a new scheduling gate to an already-created Pod.

## 11. OpenShift Notes

OpenShift supports Kubernetes scheduling gates in versions that include the relevant Kubernetes API functionality.

Useful commands:

```bash
oc get pods -n <project-name> -o wide
oc describe pod <pod-name> -n <project-name>
oc get pod <pod-name> -n <project-name> -o yaml
oc get events -n <project-name> \
  --sort-by=.metadata.creationTimestamp
```

When a Pod is not scheduled, inspect `spec.schedulingGates` before troubleshooting node capacity. If no scheduling gates remain, investigate the normal scheduler constraints and events.

## 12. Interview Questions and Answers

**Q1. What are Pod scheduling gates?**
They are named conditions in `spec.schedulingGates` that prevent a Pod from being scheduled until the gates are removed.

**Q2. Which Kubernetes field configures scheduling gates?**
`spec.schedulingGates`.

**Q3. Does Kubernetes automatically remove a custom scheduling gate?**
No. An authorized controller or process must remove it.

**Q4. What happens after all scheduling gates are removed?**
The Pod becomes eligible for normal scheduling, subject to resource availability and other scheduling constraints.

**Q5. What is the difference between scheduling gates and readiness gates?**
Scheduling gates delay node assignment. Readiness gates delay the Pod's Ready status after scheduling and startup.

**Q6. Can a high-priority Pod bypass a scheduling gate?**
No. Priority and preemption do not bypass an existing scheduling gate.

**Q7. Can scheduling gates be added after Pod creation?**
Kubernetes restricts updates to this field. Gates are intended to be specified at creation and subsequently removed, not added later.

**Q8. Do scheduling gates reserve CPU and memory on a node?**
No. The Pod is held back from scheduling and does not reserve node resources through a normal scheduler assignment.

**Q9. What should you check if a Pod remains Pending after its gate is removed?**
Check Pod events, CPU and memory requests, node selectors, affinity, taints, tolerations, and storage requirements.

**Q10. Give a real-world use case for scheduling gates.**
A custom workload controller can hold Pods until an external approval or preparation workflow completes, then remove the gates to allow scheduling.

## 13. Quick Revision

- Scheduling gates: Delay scheduling of a Pod.
- YAML field: `spec.schedulingGates`.
- Gate identifier: `name`.
- Who removes gates? An authorized controller or process.
- After removal: The Pod becomes eligible for normal scheduling.
- Does it reserve node resources? No.
- Can priority bypass it? No.
- Readiness gate difference: Scheduling gates affect placement; readiness gates affect Ready status.
- Troubleshooting: Inspect the Pod specification, gate-removal controller, and scheduler events.

Final takeaway: Scheduling gates give workload controllers control over when a Pod enters normal scheduling. Removing a gate makes the Pod eligible for scheduling, but does not guarantee that a suitable node is available.
