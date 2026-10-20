# Kubernetes Notes: Pod Resources and Performance

## 1. What Are Pod Resources?

Pod resources are the CPU, memory, storage, and other computing resources required by containers running inside a Pod.

Kubernetes uses resource configuration to schedule workloads onto suitable worker nodes and control container resource consumption.

The main resource types are:

- CPU — processing capacity.
- Memory — RAM used by applications.
- Ephemeral storage — temporary local storage used for writable container layers, logs, and temporary files.
- Extended resources — specialized resources such as GPUs.

Resource requests and limits are configured for individual containers. A Pod's total resource requirement is calculated from its containers and applicable Pod overhead.

## 2. CPU Requests and Limits

### CPU request

A CPU request specifies the amount of CPU capacity Kubernetes uses when scheduling a container.

For example:

```yaml
resources:
  requests:
    cpu: "250m"
```

`250m` means 250 millicores, or one-quarter of a CPU core.

### CPU limit

A CPU limit specifies the maximum CPU time a container can consume over time. When the container reaches its CPU limit, it can be throttled.

```yaml
resources:
  limits:
    cpu: "500m"
```

This allows up to half a CPU core's worth of CPU time, subject to CPU scheduling and throttling.

### CPU units

| CPU value | Meaning |
|---|---|
| 100m | 0.1 CPU |
| 250m | 0.25 CPU |
| 500m | 0.5 CPU |
| 1000m | 1 CPU |
| 2 | 2 CPUs |

Important: CPU limits generally cause throttling, not an automatic container restart.

## 3. Memory Requests and Limits

### Memory request

A memory request tells the scheduler how much memory to account for when placing a container on a node.

```yaml
resources:
  requests:
    memory: "256Mi"
```

### Memory limit

A memory limit restricts how much memory a container can use. If the container exceeds its enforced memory limit, it can be terminated by the kernel's out-of-memory mechanism.

```yaml
resources:
  limits:
    memory: "512Mi"
```

### Memory units

| Value | Meaning |
|---|---|
| 128Mi | 128 mebibytes |
| 256Mi | 256 mebibytes |
| 512Mi | 512 mebibytes |
| 1Gi | 1 gibibyte |
| 2Gi | 2 gibibytes |

Mi and Gi are binary units. They are not exactly the same as MB and GB.

Important: A memory request is not a hard usage ceiling. A container may use more memory than its request when resources are available, provided it stays within its enforced limit and the node can support the workload.

## 4. Complete Pod YAML Example

Create `resource-pod.yaml`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
spec:
  containers:
    - name: nginx
      image: nginx:latest
      ports:
        - containerPort: 80
      resources:
        requests:
          cpu: "250m"
          memory: "128Mi"
        limits:
          cpu: "500m"
          memory: "256Mi"
```

### Explanation

- CPU request: 250m
- CPU limit: 500m
- Memory request: 128Mi
- Memory limit: 256Mi

Kubernetes uses the requests for scheduling decisions and enforces the applicable limits at runtime.

### Deploy and verify

```bash
kubectl apply -f resource-pod.yaml
kubectl get pods
kubectl describe pod resource-demo
kubectl get pod resource-demo -o yaml
```

## 5. Pod Resource Calculation

Suppose a Pod contains two regular containers.

| Resource | Frontend | Sidecar | Total request |
|---|---|---|---|
| CPU | 250m | 100m | 350m |
| Memory | 256Mi | 64Mi | 320Mi |

For ordinary application containers without init-container scheduling complications, the combined regular-container requests are added together.

The Pod requests approximately:

- CPU: 350m
- Memory: 320Mi

Actual scheduling calculations can also account for init containers, restartable init sidecars, Pod overhead, and other Kubernetes rules.

## 6. What Happens When Resources Are Insufficient?

### Scenario 1: Insufficient CPU or memory on worker nodes

If no eligible node has enough allocatable resources to satisfy a Pod's requests, the Pod may remain Pending.

Check:

```bash
kubectl get pods
kubectl describe pod <pod-name>
kubectl get nodes
kubectl describe node <node-name>
```

Look for scheduling messages such as:

```
Insufficient cpu
Insufficient memory
```

### Scenario 2: CPU throttling

If a container repeatedly reaches its CPU limit, its execution can be throttled.

Possible symptoms:

- Slow API responses.
- Increased application latency.
- Reduced throughput.
- Delayed background jobs.

Check metrics from your monitoring system, such as CPU usage and throttling metrics.

### Scenario 3: Out-of-memory termination

If a container exceeds its enforced memory limit, it may be terminated with an OOM-related status.

Check:

```bash
kubectl describe pod <pod-name>
kubectl get pod <pod-name> \
  -o jsonpath='{.status.containerStatuses[*].lastState.terminated.reason}'
```

A returned value of `OOMKilled` indicates that the previous container instance was terminated because of an out-of-memory condition.

## 7. Kubernetes QoS Classes

Kubernetes assigns Pods to Quality of Service (QoS) classes based on their resource configuration.

| QoS class | General rule |
|---|---|
| Guaranteed | Every regular container has CPU and memory requests and limits configured, with each request equal to its corresponding limit. Init containers must also satisfy the applicable Guaranteed criteria. |
| Burstable | The Pod does not qualify as Guaranteed, but at least one container has a CPU or memory request or limit configured. |
| BestEffort | No containers in the Pod specify CPU or memory requests or limits. |

Check a Pod's QoS class:

```bash
kubectl get pod <pod-name> \
  -o jsonpath='{.status.qosClass}'
```

Important: QoS affects how Kubernetes treats Pods under resource pressure, especially memory pressure. It does not guarantee application performance or prevent all evictions and failures.

## 8. Ephemeral Storage Resources

Containers also consume node-local storage through writable container layers, logs, and temporary files.

You can set requests and limits for ephemeral storage:

```yaml
resources:
  requests:
    ephemeral-storage: "1Gi"
  limits:
    ephemeral-storage: "2Gi"
```

These values help Kubernetes account for local storage needs and enforce applicable ephemeral-storage limits.

Check node storage and Pod events when containers are evicted because of local storage pressure.

```bash
kubectl describe node <node-name>
kubectl describe pod <pod-name>
kubectl get events --sort-by=.metadata.creationTimestamp
```

## 9. Monitoring Pod Performance

### Check current Pod resource usage

```bash
kubectl top pods
```

For a specific namespace:

```bash
kubectl top pods -n coffeeshop
```

For node usage:

```bash
kubectl top nodes
```

These commands require the Kubernetes Metrics API to be available, commonly through Metrics Server.

### Check resource configuration

```bash
kubectl describe pod <pod-name>
kubectl get pod <pod-name> -o yaml
```

### Useful performance metrics

- CPU utilization.
- Memory working set and usage.
- CPU throttling.
- Application response latency.
- Request throughput.
- Pod restart count.
- Node memory and disk pressure.

For long-term performance analysis, monitoring tools such as Prometheus and Grafana are commonly used.

## 10. Resource Management with LimitRange and ResourceQuota

### LimitRange

A LimitRange can define default resource requests and limits, as well as minimum and maximum allowed resource values for containers or Pods in a namespace.

Inspect it:

```bash
kubectl get limitrange -n coffeeshop
kubectl describe limitrange -n coffeeshop
```

### ResourceQuota

A ResourceQuota can limit aggregate resource requests, limits, and other resource consumption within a namespace.

Inspect it:

```bash
kubectl get resourcequota -n coffeeshop
kubectl describe resourcequota -n coffeeshop
```

These controls help prevent one namespace from consuming an excessive share of cluster resources.

## 11. Real-Time Example: CoffeeShop Application

For a small test environment, you might start with the following illustrative resource settings.

### Frontend container

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
```

### Spring Boot backend container

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "512Mi"
  limits:
    cpu: "1000m"
    memory: "1Gi"
```

These are starting values for testing, not universal production recommendations. Measure actual usage under realistic traffic before finalizing them.

For a Java application, also account for JVM heap settings, non-heap memory, thread stacks, and other native memory overhead. Setting the heap equal to the container's entire memory limit can cause OOM termination.

## 12. Performance Troubleshooting Workflow

When an application becomes slow, use this sequence:

1. Check Pod status and restart count.

```bash
kubectl get pods -n coffeeshop
```

2. Check current CPU and memory usage.

```bash
kubectl top pods -n coffeeshop
```

3. Inspect Pod events and resource settings.

```bash
kubectl describe pod <pod-name> -n coffeeshop
```

4. Review application logs.

```bash
kubectl logs <pod-name> -n coffeeshop
```

5. Check node capacity and pressure.

```bash
kubectl describe node <node-name>
```

6. Review CPU throttling, memory trends, application latency, and traffic volume in your monitoring system.

7. Adjust resource requests and limits based on measured workload behavior, then validate the change under load.

## 13. OpenShift Commands

The same Kubernetes resource concepts apply to OpenShift.

```bash
oc get pods -n coffeeshop
oc describe pod <pod-name> -n coffeeshop
oc adm top pods -n coffeeshop
oc adm top nodes
oc get limitrange,resourcequota -n coffeeshop
```

The availability of metrics commands depends on cluster configuration and permissions.

## 14. Interview Questions and Answers

**Q1. What is the difference between CPU requests and limits?**
CPU requests influence scheduling decisions. CPU limits constrain CPU usage through throttling.

**Q2. What is the difference between memory requests and limits?**
Memory requests help Kubernetes schedule workloads. Memory limits impose a runtime memory ceiling that can lead to OOM termination when exceeded.

**Q3. What is OOMKilled?**
It indicates that a container was terminated by an out-of-memory mechanism, commonly because it exceeded its memory limit or the node experienced memory pressure.

**Q4. What are Kubernetes QoS classes?**
Guaranteed, Burstable, and BestEffort. They are assigned according to the Pod's CPU and memory request/limit configuration.

**Q5. What happens if a Pod's resource requests cannot be satisfied?**
The scheduler cannot place the Pod on an eligible node, so the Pod can remain Pending.

**Q6. How do you check CPU and memory usage?**
Use `kubectl top pods` and `kubectl top nodes` when the Metrics API is available.

**Q7. Can a container use more CPU than its request?**
Yes. A CPU request is not a hard CPU usage limit. The container may use additional CPU when available, up to its applicable limit if one is configured.

**Q8. How do you troubleshoot a slow Spring Boot application in Kubernetes?**
Check CPU and memory usage, CPU throttling, JVM memory configuration, Pod restarts, application logs, node pressure, and application latency metrics.

## 15. Quick Revision

- Requests influence scheduling.
- CPU limits can cause throttling.
- Memory limits can lead to OOM termination.
- Ephemeral storage settings help manage node-local storage consumption.
- QoS classes are Guaranteed, Burstable, and BestEffort.
- `kubectl top` displays current resource metrics when available.
- LimitRange and ResourceQuota help control namespace resource consumption.
- Set production requests and limits using actual measurements and workload testing.
