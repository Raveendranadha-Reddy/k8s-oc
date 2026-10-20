# Kubernetes Pod Scheduling – Complete Notes

## 1. What Is Pod Scheduling?

Pod Scheduling is the process of selecting a suitable worker node for a Pod.

The Kubernetes kube-scheduler monitors Pods that do not have an assigned node, evaluates eligible nodes, and selects a suitable node for each Pod.

After scheduling, the kubelet on the selected node works with the container runtime to start the Pod's containers.

### Scheduling workflow

1. A user creates a Pod or a controller creates one.
2. The API server stores the Pod object.
3. The scheduler identifies the unscheduled Pod.
4. The scheduler filters nodes that cannot run the Pod.
5. It scores the remaining feasible nodes.
6. The scheduler selects a node and records the binding.
7. The kubelet on that node starts the Pod.

```
User / Deployment
        |
        v
   API Server
        |
        v
  Pending Pod
        |
        v
 kube-scheduler
        |
        v
Filter eligible nodes
        |
        v
Score eligible nodes
        |
        v
 Select a node
        |
        v
  API Server records
  node assignment
        |
        v
 Node kubelet
        |
        v
 Container Runtime
        |
        v
   Running Pod
```

Important: The scheduler selects a node; it does not directly start the containers.

## 2. What Does the Scheduler Consider?

The scheduler considers multiple constraints, including:

- CPU and memory requests
- Node readiness and schedulability
- Node selectors and node affinity
- Pod affinity and anti-affinity
- Taints and tolerations
- Topology spread constraints
- Volume topology and attachment constraints
- Scheduling policies and resource availability

A node may have free CPU or memory and still be unsuitable because another scheduling constraint is not satisfied.

### Requests vs actual usage

Scheduling primarily uses declared resource requests when checking whether a node has sufficient allocatable resources.

For example:

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
```

The scheduler considers the requested CPU and memory, not just the application's current usage.

## 3. Pod Scheduling and Pod Phases

A Pod can remain in the Pending phase while waiting for scheduling or other setup tasks.

Useful commands:

```bash
kubectl get pods -o wide
kubectl describe pod <pod-name>
kubectl get events --sort-by=.metadata.creationTimestamp
```

In `kubectl describe pod`, inspect:

- Node
- Conditions
- Events
- Scheduling failure messages such as `FailedScheduling`

Important: Pending is a Pod phase. FailedScheduling is an event reason, not a Pod phase.

## 4. Node Selector

A `nodeSelector` is the simplest way to constrain a Pod to nodes with specific labels.

### Step 1: Check node labels

```bash
kubectl get nodes --show-labels
```

### Step 2: Add a label

```bash
kubectl label node worker-1 disktype=ssd
```

Replace `worker-1` with an actual node name.

### Step 3: Create a Pod

Save as `node-selector.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ssd-pod
spec:
  nodeSelector:
    disktype: ssd
  containers:
    - name: app
      image: nginx:stable
      resources:
        requests:
          cpu: "100m"
          memory: "128Mi"
```

Deploy and verify:

```bash
kubectl apply -f node-selector.yaml
kubectl get pod ssd-pod -o wide
kubectl describe pod ssd-pod
```

The Pod can be scheduled only onto a node with the matching label, assuming all other scheduling requirements are satisfied.

### Use cases

- Schedule workloads on GPU nodes.
- Select nodes with specific hardware.
- Separate production and development workloads.

## 5. Node Affinity

Node affinity is a more expressive way to select nodes based on labels.

Two main types are available.

**A. requiredDuringSchedulingIgnoredDuringExecution**
The node must satisfy the rule when scheduling occurs. If no suitable node exists, the Pod cannot be scheduled.

**B. preferredDuringSchedulingIgnoredDuringExecution**
The scheduler prefers matching nodes but may choose another feasible node.

### Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: affinity-pod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: disktype
                operator: In
                values:
                  - ssd
  containers:
    - name: app
      image: nginx:stable
```

### Common operators

| Operator | Meaning |
|---|---|
| In | Value must be in the specified list |
| NotIn | Value must not be in the list |
| Exists | Label key must exist |
| DoesNotExist | Label key must not exist |
| Gt | Label value must be greater than the specified value |
| Lt | Label value must be less than the specified value |

Gt and Lt require numeric label values.

### Important distinction

`IgnoredDuringExecution` means that if a node's labels change after the Pod is scheduled, Kubernetes does not automatically evict the Pod solely because the affinity rule is no longer satisfied.

## 6. Taints and Tolerations

Taints are applied to nodes to repel Pods that do not tolerate them.

Tolerations allow Pods to be considered for scheduling onto nodes with matching taints. A toleration does not guarantee that the Pod will be scheduled there.

### Common taint effects

| Effect | Meaning |
|---|---|
| NoSchedule | New Pods without a matching toleration are not scheduled onto the node |
| PreferNoSchedule | Scheduler tries to avoid placing non-tolerating Pods there |
| NoExecute | Non-tolerating Pods may be evicted, and new ones are not scheduled there |

### Step 1: Add a taint

```bash
kubectl taint nodes worker-1 dedicated=app:NoSchedule
```

### Step 2: Create a tolerating Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: tolerated-pod
spec:
  tolerations:
    - key: dedicated
      operator: Equal
      value: app
      effect: NoSchedule
  containers:
    - name: app
      image: nginx:stable
```

Apply:

```bash
kubectl apply -f tolerated-pod.yaml
```

This Pod is allowed to be considered for the tainted node, but other scheduling constraints still apply.

### Remove the taint

```bash
kubectl taint nodes worker-1 dedicated=app:NoSchedule-
```

### Common use cases

- Dedicated database nodes
- GPU worker nodes
- Special-purpose infrastructure
- Nodes reserved for specific workloads

## 7. Node Selector vs Node Affinity vs Taints

| Feature | Purpose |
|---|---|
| nodeSelector | Simple mandatory node-label matching |
| Node affinity | Mandatory or preferred node-label matching |
| Taint | Repels Pods from a node |
| Toleration | Allows a Pod to tolerate a matching taint |

Interview point: Node affinity attracts or selects Pods toward suitable nodes. Taints repel Pods unless they tolerate the taint.

## 8. Pod Affinity and Pod Anti-Affinity

These rules consider labels on other Pods rather than only node labels.

**Pod affinity**
Places a Pod near other matching Pods within a specified topology domain.

Example: Schedule an application component close to another component.

**Pod anti-affinity**
Avoids placing a Pod in a topology domain that already contains matching Pods.

Example: Spread replicas across different nodes to improve availability.

### Example: Anti-affinity across nodes

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: web-app
                topologyKey: kubernetes.io/hostname
      containers:
        - name: nginx
          image: nginx:stable
```

This is a preference to spread replicas across different nodes. Because it uses `preferred`, the scheduler may still place replicas together when necessary.

To enforce separation, use `requiredDuringSchedulingIgnoredDuringExecution`, but ensure there are enough eligible nodes or some replicas may remain Pending.

### topologyKey

Defines the topology domain used for the rule.

Examples:

- `kubernetes.io/hostname`: individual node
- `topology.kubernetes.io/zone`: availability zone

## 9. Topology Spread Constraints

Topology spread constraints distribute Pods across topology domains, such as nodes or availability zones.

They are useful for highly available applications.

### Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spread-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: spread-app
  template:
    metadata:
      labels:
        app: spread-app
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: spread-app
      containers:
        - name: app
          image: nginx:stable
```

### Field meanings

- `maxSkew`: Maximum permitted difference in matching Pod counts across eligible topology domains.
- `topologyKey`: Label identifying the topology domain.
- `ScheduleAnyway`: Prefer a placement that improves distribution but do not fail scheduling solely because of this constraint.
- `DoNotSchedule`: Do not schedule the Pod if doing so would violate the constraint.

Topology spread constraints and anti-affinity solve related but different problems. Spread constraints control distribution more directly; anti-affinity defines placement rules relative to matching Pods.

## 10. Resource-Based Scheduling

The scheduler evaluates resource requests against node allocatable resources and existing Pod requests.

### Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-pod
spec:
  containers:
    - name: app
      image: nginx:stable
      resources:
        requests:
          cpu: "500m"
          memory: "512Mi"
        limits:
          cpu: "1"
          memory: "1Gi"
```

Meaning:

- CPU request: 0.5 CPU core.
- Memory request: 512 MiB.
- CPU limit: 1 CPU core.
- Memory limit: 1 GiB.

If no eligible node has enough allocatable CPU or memory for the Pod's requests, the Pod may remain Pending.

Inspect nodes and Pods:

```bash
kubectl describe node <node-name>
kubectl top nodes
kubectl top pods
kubectl describe pod resource-pod
```

`kubectl top` requires working metrics support. Actual usage reported by `kubectl top` is not the same as the scheduler's resource-request accounting.

## 11. Manual Scheduling with nodeName

The `nodeName` field directly specifies the node for a Pod.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: manual-pod
spec:
  nodeName: worker-1
  containers:
    - name: app
      image: nginx:stable
```

This bypasses normal scheduler selection for that Pod.

Caution: Kubernetes does not use the scheduler to choose a different node if the specified node is unavailable or unsuitable. Avoid using `nodeName` for ordinary production scheduling unless there is a specific operational reason.

## 12. cordon and drain

These commands control whether a node accepts new scheduled workloads and help prepare nodes for maintenance.

### Cordon

```bash
kubectl cordon worker-1
```

Marks the node unschedulable for ordinary new scheduling.

Existing Pods continue running unless something else affects them.

### Uncordon

```bash
kubectl uncordon worker-1
```

Allows normal scheduling on the node again.

### Drain

```bash
kubectl drain worker-1 --ignore-daemonsets
```

Evicts eligible Pods so the node can be maintained.

Drain behavior depends on PodDisruptionBudgets, local storage, DaemonSets, and other workload constraints. It may require additional flags or intervention.

Important: `drain` is not the same as `cordon`. Cordon prevents normal new placements; drain attempts to evict eligible existing Pods.

## 13. Scheduling in OpenShift

OpenShift uses Kubernetes scheduling concepts and adds platform-specific integrations.

Useful commands:

```bash
oc get nodes
oc get pods -o wide
oc describe pod <pod-name>
oc get events --sort-by=.metadata.creationTimestamp
oc get machineconfigpools
```

Depending on the environment, OpenShift administrators may use:

- Node labels and taints
- Pod affinity and anti-affinity
- Topology spread constraints
- Pod priority and preemption
- Scheduler profiles or custom schedulers, where configured
- MachineSets and node pools to provision suitable worker capacity

Important: MachineSets provision and manage machines in supported OpenShift platforms; they do not replace the kube-scheduler.

## 14. Pod Scheduling Troubleshooting

### Scenario 1: Pod is Pending

```bash
kubectl describe pod <pod-name>
kubectl get events --sort-by=.metadata.creationTimestamp
```

Look for `FailedScheduling` and messages indicating insufficient CPU or memory, unmatched node selectors, untolerated taints, affinity conflicts, or volume constraints.

### Scenario 2: Insufficient CPU

Typical event:

```
0/3 nodes are available: 3 Insufficient cpu
```

Possible solutions:

- Add worker capacity.
- Reduce unrealistic CPU requests.
- Check existing workload requests.
- Review node allocatable resources.

Do not reduce requests below realistic application needs simply to force scheduling.

### Scenario 3: Node selector mismatch

Typical event:

```
node(s) didn't match Pod's node affinity/selector
```

Check labels:

```bash
kubectl get nodes --show-labels
kubectl get pod <pod-name> -o yaml
```

Correct the selector or apply the intended label to an appropriate node.

### Scenario 4: Untolerated taint

Typical event:

```
node(s) had untolerated taint
```

Check taints:

```bash
kubectl describe node <node-name>
```

Add a toleration only if the workload is supposed to run on that node. Do not remove system taints without understanding their purpose.

### Scenario 5: Pod cannot spread across nodes

Possible causes:

- Too few eligible nodes
- Required anti-affinity
- Topology spread constraints
- Node labels missing for the selected topology key
- Resource shortages

Check:

```bash
kubectl get nodes --show-labels
kubectl get pods -o wide
kubectl describe pod <pod-name>
```

### Scenario 6: Node is NotReady

```bash
kubectl get nodes
kubectl describe node <node-name>
```

Investigate kubelet health, node conditions, resource pressure, and network connectivity.

## 15. Practical Lab

### Objective

Schedule a Pod only on a node labeled `workload=testing`.

### Step 1: Identify a suitable worker node

```bash
kubectl get nodes
```

### Step 2: Label the node

Replace `worker-1` with the actual node name:

```bash
kubectl label node worker-1 workload=testing
```

### Step 3: Create the manifest

Save as `scheduled-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: scheduled-pod
spec:
  nodeSelector:
    workload: testing
  containers:
    - name: nginx
      image: nginx:stable
      resources:
        requests:
          cpu: "100m"
          memory: "128Mi"
```

### Step 4: Deploy and verify

```bash
kubectl apply -f scheduled-pod.yaml
kubectl get pod scheduled-pod -o wide
kubectl describe pod scheduled-pod
```

Verify that the NODE column shows the selected node.

### Step 5: Clean up

```bash
kubectl delete pod scheduled-pod
kubectl label node worker-1 workload-
```

Only remove the label if you added it for this lab and no other workload depends on it.

## 16. Real-Time Interview Questions

**Q1. What is Pod Scheduling?**
The process of selecting a suitable worker node for a Pod based on resources, policies, and placement constraints.

**Q2. Which component schedules Pods?**
The kube-scheduler.

**Q3. What is the difference between filtering and scoring?**
Filtering removes nodes that cannot run the Pod. Scoring ranks the remaining feasible nodes.

**Q4. Why does a Pod remain Pending?**
Possible reasons include insufficient resources, unmatched selectors, affinity constraints, untolerated taints, or storage scheduling constraints.

**Q5. What is the difference between nodeSelector and node affinity?**
nodeSelector provides simple required label matching. Node affinity supports richer expressions and required or preferred rules.

**Q6. What is the difference between taints and tolerations?**
Taints repel non-tolerating Pods from nodes. Tolerations permit matching Pods to be considered for placement on those nodes.

**Q7. Does a toleration guarantee scheduling on a node?**
No. Resource availability and all other scheduling requirements still apply.

**Q8. What is pod anti-affinity?**
A placement rule that avoids placing a Pod in a topology domain containing matching Pods.

**Q9. What is topology spread?**
A mechanism for distributing matching Pods across topology domains, such as nodes or zones.

**Q10. What is the difference between cordon and drain?**
Cordon marks a node unschedulable for ordinary new placements. Drain attempts to evict eligible existing Pods for maintenance.

**Q11. What does nodeName do?**
It specifies the node directly and bypasses normal scheduler node selection for that Pod.

**Q12. What happens if no node satisfies required affinity?**
The Pod remains unscheduled until the constraint or available cluster conditions change.

**Q13. Does the scheduler use current CPU usage alone?**
No. Scheduling decisions primarily consider declared resource requests against allocatable capacity, together with other scheduling constraints.

**Q14. How do you troubleshoot FailedScheduling?**
Use `kubectl describe pod`, inspect Events, check node labels and taints, verify resource requests, and review affinity and storage constraints.

**Q15. Is OpenShift scheduling different from Kubernetes?**
The fundamental scheduler behavior is Kubernetes-based. OpenShift adds platform-specific integrations, security controls, and administrative workflows.

## 17. Quick Revision

| Topic | Key Point |
|---|---|
| kube-scheduler | Selects a node for an unscheduled Pod |
| Filtering | Identifies feasible nodes |
| Scoring | Ranks feasible nodes |
| nodeSelector | Required label matching |
| Node affinity | Required or preferred node placement |
| Taints | Repel Pods |
| Tolerations | Permit matching tainted-node placement |
| Pod affinity | Prefer or require placement near matching Pods |
| Pod anti-affinity | Avoid matching Pods in a topology domain |
| Topology spread | Distribute Pods across topology domains |
| Resource requests | Used for capacity-aware scheduling |
| nodeName | Direct node assignment |
| Cordon | Prevents normal new scheduling on a node |
| Drain | Evicts eligible Pods for maintenance |
| FailedScheduling | Event reason explaining why placement failed |

Key takeaway: When a Pod is Pending, start with `kubectl describe pod` and Events. Identify the exact scheduling constraint before changing resources, labels, taints, affinity, or storage settings.
