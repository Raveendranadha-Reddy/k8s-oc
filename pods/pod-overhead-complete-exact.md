# Kubernetes Pod Overhead – Complete Notes

## 1. What Is Pod Overhead?

Pod overhead is the additional amount of CPU and memory reserved for the runtime overhead of running a Pod, beyond the resources requested by its containers.

Kubernetes supports Pod overhead for workloads that use a RuntimeClass to configure a container runtime with additional per-Pod resource costs.

For example, a sandboxed container runtime may consume extra memory to create and manage a Pod sandbox. Kubernetes can account for this additional resource consumption through Pod overhead.

Important: Pod overhead is not the same as application container resources. It represents extra resources associated with running the Pod itself.

## 2. Why Is Pod Overhead Required?

Pod overhead helps Kubernetes account for runtime costs that are not included in individual container resource requests.

Common reasons include:

- Sandboxed container runtimes.
- Virtualization-based container isolation.
- Additional resources required to create a Pod sandbox.
- More accurate scheduling and resource accounting.

For example, a runtime may create an additional lightweight virtual machine or sandbox environment for each Pod. That environment consumes resources even if the application containers have small resource requests.

## 3. How Pod Overhead Works

The basic flow is:

1. An administrator configures a RuntimeClass.
2. The RuntimeClass specifies a Pod overhead value.
3. A Pod references that RuntimeClass using `runtimeClassName`.
4. Kubernetes associates the configured overhead with the Pod.
5. The scheduler accounts for container resource requests plus Pod overhead when evaluating resource requirements.

Example:

- Application container CPU request: 500m
- Pod overhead CPU: 100m
- Total CPU request used for scheduling: 600m

Memory example:

- Application container memory request: 128Mi
- Pod overhead memory: 64Mi
- Total memory request used for scheduling: 192Mi

These are illustrative values; the actual overhead depends on the runtime configuration.

## 4. RuntimeClass Configuration

A RuntimeClass defines the container runtime configuration used by Pods. Its handler must be supported and configured on the relevant nodes.

Example: `runtimeclass.yaml`

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: sandboxed-runtime
handler: sandboxed-handler
overhead:
  podFixed:
    cpu: "100m"
    memory: "64Mi"
```

Apply the configuration:

```bash
kubectl apply -f runtimeclass.yaml
```

Check the RuntimeClass:

```bash
kubectl get runtimeclass
kubectl describe runtimeclass sandboxed-runtime
kubectl get runtimeclass sandboxed-runtime -o yaml
```

### Important fields

| Field | Purpose |
|---|---|
| apiVersion | API group and version for RuntimeClass |
| kind | Kubernetes resource type |
| metadata.name | RuntimeClass name |
| handler | Runtime handler configured on nodes |
| overhead.podFixed | Additional resource overhead associated with each Pod |

Important: `sandboxed-handler` is an example, not a built-in runtime handler guaranteed to exist on your cluster. Do not use this RuntimeClass for a real workload unless the runtime handler is installed and configured on the target nodes.

## 5. Create a Pod That Uses Pod Overhead

Example: `overhead-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: overhead-demo
spec:
  runtimeClassName: sandboxed-runtime
  containers:
    - name: app
      image: nginx:1.28
      resources:
        requests:
          cpu: "500m"
          memory: "128Mi"
        limits:
          cpu: "1"
          memory: "256Mi"
```

Apply:

```bash
kubectl apply -f overhead-pod.yaml
```

Inspect the Pod:

```bash
kubectl get pod overhead-demo -o yaml
kubectl describe pod overhead-demo
kubectl get pod overhead-demo -o jsonpath='{.spec.overhead}'
```

If the RuntimeClass is valid and overhead admission succeeds, Kubernetes can populate the Pod's `spec.overhead` with the configured values.

If the runtime handler is unavailable, the Pod may fail to start or remain unscheduled, depending on the configuration and failure.

## 6. How Does Pod Overhead Affect Scheduling?

The scheduler considers the resource requests of the Pod's containers and the Pod overhead.

Example:

| Resource | CPU | Memory |
|---|---|---|
| Container request | 500m | 128Mi |
| Pod overhead | 100m | 64Mi |
| Total scheduling request | 600m | 192Mi |

If a node has only 550m CPU available for scheduling, this Pod cannot fit there because its combined request is 600m.

The scheduler may consider another suitable node. If none has enough resources, the Pod can remain Pending.

Key point: Scheduling is based primarily on resource requests and node allocatable capacity, not simply on the current CPU or memory usage reported by monitoring tools.

## 7. Pod Overhead vs Container Resources

| Feature | Container resources | Pod overhead |
|---|---|---|
| Applies to | Individual containers | Pod-level runtime overhead |
| Configured through | `resources.requests` and `resources.limits` | `RuntimeClass.overhead.podFixed` |
| Main purpose | Declare application resource requirements | Account for runtime resources beyond containers |
| Used in scheduling | Yes | Yes |
| Example | Application requests 500m CPU | Runtime overhead adds 100m CPU |

Container requests and limits should be configured based on the application's requirements. Pod overhead should represent the additional runtime cost, not an arbitrary buffer for the application.

## 8. Pod Overhead vs Kubernetes ResourceQuota

These are related but different concepts.

- Pod overhead accounts for extra resources associated with a Pod's runtime.
- ResourceQuota limits aggregate resource consumption or resource requests within a namespace.

When a namespace uses ResourceQuota, Kubernetes resource accounting can include Pod overhead in relevant Pod resource calculations.

Check quotas:

```bash
kubectl get resourcequota -A
kubectl describe resourcequota -n default
```

If a Pod is rejected because of quota, inspect the admission error and quota details rather than assuming the node lacks capacity.

## 9. Pod Overhead vs QoS Classes

Pod Quality of Service (QoS) classes are Guaranteed, Burstable, and BestEffort.

Pod overhead is not a separate QoS class.

| Concept | Purpose |
|---|---|
| Pod overhead | Accounts for runtime resource requirements |
| QoS class | Classifies Pods according to container resource requests and limits |
| PriorityClass | Influences scheduling order and preemption |
| ResourceQuota | Enforces namespace-level resource limits |

Do not assume that adding Pod overhead automatically makes a Pod Guaranteed or changes its QoS class in a particular way. Determine QoS from the Pod's resource configuration and the applicable Kubernetes rules.

## 10. Troubleshooting Pod Overhead

### Step 1: Check Pod status

```bash
kubectl get pods -A -o wide
```

### Step 2: Inspect Pod events

```bash
kubectl describe pod overhead-demo
```

Look for:

- FailedScheduling
- Insufficient CPU or memory
- RuntimeClass-related errors
- Admission or quota errors

### Step 3: Check RuntimeClass

```bash
kubectl get runtimeclass
kubectl describe runtimeclass sandboxed-runtime
```

Verify that:

- The RuntimeClass exists.
- The handler is configured on the relevant nodes.
- The overhead values are correct.

### Step 4: Inspect Pod overhead

```bash
kubectl get pod overhead-demo -o jsonpath='{.spec.overhead}'
echo
```

### Step 5: Check node resources

```bash
kubectl describe nodes
kubectl top nodes
kubectl top pods -A
```

`kubectl top` requires a working metrics pipeline. Use node allocatable resources, scheduled requests, and Pod events to diagnose scheduling failures.

### Step 6: Check namespace quotas

```bash
kubectl get resourcequota -A
kubectl get limitrange -A
```

Confirm that namespace policies permit the workload.

## 11. Practical Lab Notes

A real Pod Overhead lab requires a cluster with a configured runtime handler that supports the intended runtime behavior.

A standard cluster using its default runtime may not provide a suitable sandboxed handler. Creating a RuntimeClass YAML alone does not install or configure the runtime.

### Recommended lab sequence

1. Check existing RuntimeClasses.
2. Confirm which runtime handlers are supported by the cluster.
3. Configure a suitable runtime and its overhead through the cluster administrator.
4. Create a Pod referencing that RuntimeClass.
5. Inspect `spec.overhead`.
6. Compare the Pod's container requests with the total scheduling request.
7. Review scheduling events and quota accounting.

### Commands

```bash
kubectl get runtimeclass
kubectl get nodes
kubectl describe node <node-name>
kubectl get pod overhead-demo -o yaml
kubectl describe pod overhead-demo
```

Use a test cluster for runtime experiments. Avoid changing production runtime configuration without an approved rollout plan.

## 12. OpenShift Notes

Pod Overhead is a Kubernetes feature, but actual runtime support depends on the OpenShift version and configured runtime.

In OpenShift, inspect available RuntimeClasses:

```bash
oc get runtimeclass
oc get pods -n <project-name>
oc describe pod <pod-name> -n <project-name>
oc get pod <pod-name> -n <project-name> -o yaml
```

If a specialized runtime is being used, verify its installation, supported nodes, runtime handler, and resource accounting with the cluster administrator.

Do not assume that every OpenShift cluster has a sandboxed runtime or a custom RuntimeClass configured by default.

## 13. Interview Questions and Answers

**Q1. What is Pod Overhead?**
Pod Overhead is the additional CPU and memory resource cost associated with running a Pod's runtime sandbox, beyond the resources requested by its containers.

**Q2. Which Kubernetes resource configures Pod Overhead?**
RuntimeClass, using the `overhead.podFixed` field.

**Q3. How does Pod Overhead affect scheduling?**
Kubernetes accounts for container requests plus Pod overhead when determining whether a node has enough allocatable resources for the Pod.

**Q4. Is Pod Overhead the same as container memory usage?**
No. Container memory describes the application's declared resources or actual usage. Pod Overhead accounts for additional runtime resources associated with the Pod.

**Q5. Is Pod Overhead enabled for every Pod?**
No. It is typically associated with Pods that use a RuntimeClass configured with overhead.

**Q6. Does Pod Overhead increase the node's capacity?**
No. It accounts for additional resource requirements; it does not create extra capacity.

**Q7. Can a RuntimeClass be created without installing its runtime handler?**
A RuntimeClass object can be created, but its handler must be configured on suitable nodes for workloads to run successfully.

**Q8. What happens if a Pod's combined resource request exceeds available node capacity?**
The scheduler cannot place it on that node. The Pod may remain Pending or be considered for another suitable node.

**Q9. Is Pod Overhead the same as PriorityClass?**
No. Pod Overhead affects resource accounting; PriorityClass affects scheduling order and potential preemption.

**Q10. How do you inspect Pod Overhead?**
Use:

```bash
kubectl get pod <pod-name> -o yaml
kubectl get pod <pod-name> -o jsonpath='{.spec.overhead}'
```

## 14. Quick Revision

- Pod Overhead: Additional resource cost for running a Pod's runtime sandbox.
- Configured through: RuntimeClass.
- Field: `overhead.podFixed`.
- Pod reference: `runtimeClassName`.
- Scheduling impact: Container requests plus Pod overhead.
- Not the same as: Container limits, QoS, PriorityClass, or ResourceQuota.
- Runtime requirement: The specified runtime handler must be configured on suitable nodes.
- Troubleshooting: Check Pod events, RuntimeClass, node allocatable resources, and ResourceQuota.

Final takeaway: Pod Overhead allows Kubernetes to account for the extra CPU and memory needed by a Pod's runtime environment, making scheduling and resource accounting more accurate for supported runtimes.
