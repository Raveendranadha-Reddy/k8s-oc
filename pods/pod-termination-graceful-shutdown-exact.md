# Kubernetes Notes: Pod Termination and Graceful Shutdown

## 1. What Is Pod Termination?

Pod termination is the process of stopping a Pod and its containers when the Pod is deleted, replaced, evicted, or otherwise terminated.

Kubernetes normally attempts to stop containers gracefully before forcefully terminating them.

Example: In a CoffeeShop application, when you update the frontend Deployment, Kubernetes may terminate an old frontend Pod and create a new one with the updated image.

## 2. What Is Graceful Shutdown?

Graceful shutdown means allowing an application enough time to finish ongoing work, stop accepting new requests, close connections, and exit cleanly before it is forcibly stopped.

Benefits:

- Reduces interrupted requests.
- Helps prevent incomplete transactions.
- Allows applications to release resources.
- Supports safer rolling updates.
- Reduces disruption to users.

## 3. Pod Termination Workflow

1. **Pod deletion requested** — For example, `kubectl delete pod`.
2. **Termination begins** — Deletion timestamp is set; the Pod enters termination.
3. **Endpoint and shutdown processing** — Traffic routing updates and container shutdown can proceed concurrently.
4. **PreStop hook, if configured** — The hook runs before the container's termination signal.
5. **SIGTERM signal** — The container runtime signals the container's main process.
6. **Grace period expires** — If the process has not exited, it is forcefully terminated, typically using SIGKILL.
7. **Pod removed** — The API server eventually removes the Pod object.

Important: Endpoint updates, PreStop, and signal delivery are not strictly sequential across the entire cluster. Endpoint propagation and container shutdown can overlap.

## 4. What Is terminationGracePeriodSeconds?

terminationGracePeriodSeconds specifies how long Kubernetes allows a Pod to terminate gracefully before forcefully stopping its containers.

The default is 30 seconds.

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: graceful-pod
spec:
  terminationGracePeriodSeconds: 60
  containers:
    - name: nginx
      image: nginx:latest
      ports:
        - containerPort: 80
```

Here, Kubernetes allows up to 60 seconds for graceful termination under normal conditions.

Remember: The grace period includes the time needed for a configured PreStop hook to finish. It is not an additional 60 seconds after the hook.

## 5. What Is a PreStop Hook?

A PreStop hook is a lifecycle hook executed before Kubernetes sends the normal termination signal to the container.

It can be used to:

- Notify an application to stop accepting new work.
- Trigger an application-specific shutdown endpoint.
- Perform a short cleanup operation.
- Allow a process to drain connections.

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: prestop-pod
spec:
  terminationGracePeriodSeconds: 30
  containers:
    - name: nginx
      image: nginx:latest
      lifecycle:
        preStop:
          exec:
            command: ["/bin/sh", "-c", "sleep 5"]
```

In this example, the hook waits five seconds before normal termination signalling proceeds.

Production caution: A sleep command only delays shutdown. It does not itself drain traffic or ensure that requests finish. Use an application-specific shutdown procedure where possible.

## 6. What Is SIGTERM?

SIGTERM is a signal that asks a process to terminate gracefully.

During normal Pod termination, Kubernetes asks the container runtime to send the termination signal to the container's main process, commonly SIGTERM.

An application that handles this signal correctly can:

1. Stop accepting new requests.
2. Finish in-flight requests.
3. Close database connections.
4. Flush logs or pending writes.
5. Exit cleanly.

The application must implement suitable shutdown handling. Kubernetes cannot guarantee that the application completes its work.

## 7. What Is SIGKILL?

SIGKILL is a forceful termination signal that a process cannot catch or ignore.

If a container has not stopped when its termination grace period expires, Kubernetes typically forcefully terminates it.

Consequences can include:

- Requests interrupted before completion.
- Cleanup handlers not running.
- Pending application work being lost.
- Incomplete writes, depending on the application and storage system.

This is why applications should handle graceful shutdown correctly.

## 8. Practical Lab: Observe Pod Termination

### Step 1: Create a Pod

Save as `graceful-pod.yaml`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: graceful-pod
spec:
  terminationGracePeriodSeconds: 30
  containers:
    - name: app
      image: nginx:latest
      lifecycle:
        preStop:
          exec:
            command: ["/bin/sh", "-c", "sleep 10"]
```

### Step 2: Create the Pod

```bash
kubectl apply -f graceful-pod.yaml
```

### Step 3: Verify the Pod

```bash
kubectl get pod graceful-pod
kubectl describe pod graceful-pod
```

### Step 4: Delete the Pod

```bash
kubectl delete pod graceful-pod
```

The hook adds a delay of approximately ten seconds before the normal termination signal is sent, subject to runtime and scheduling behavior. The Pod's total termination time must fit within its grace period.

### Step 5: Watch termination

In another terminal, run:

```bash
kubectl get pod graceful-pod -w
```

The Pod may disappear quickly after it terminates, so the watch command is useful for observing changes.

### OpenShift commands

The same behavior applies in OpenShift.

```bash
oc apply -f graceful-pod.yaml
oc get pod graceful-pod -w
oc describe pod graceful-pod
oc delete pod graceful-pod
```

## 9. Pod Termination During a Deployment Update

Suppose the CoffeeShop frontend Deployment runs two replicas and you update its container image.

```bash
kubectl set image deployment/coffee-frontend \
  coffee-frontend=raveendradudyala/coffee-shop-frontend:5.0.0 \
  -n coffeeshop
```

Monitor the rollout:

```bash
kubectl rollout status deployment/coffee-frontend -n coffeeshop
kubectl get pods -n coffeeshop -w
```

During a rolling update, Kubernetes generally creates replacement Pods and terminates old Pods according to the Deployment's rollout strategy.

For a smooth update:

- Use readiness probes so unready Pods do not receive normal Service traffic.
- Configure appropriate `maxSurge` and `maxUnavailable` values.
- Ensure the application handles SIGTERM.
- Set an appropriate termination grace period.
- Implement connection draining if the application needs it.

A readiness probe alone does not guarantee that every existing connection completes before termination.

## 10. Troubleshooting

### Problem 1: Pod takes too long to terminate

```bash
kubectl get pod graceful-pod -o yaml
kubectl describe pod graceful-pod
```

Check:

- terminationGracePeriodSeconds
- PreStop hook behavior
- Application shutdown logic
- Node and container runtime health

### Problem 2: Application requests fail during deployment

Check:

- Readiness probe configuration.
- Application handling of SIGTERM.
- Connection draining and load balancer behavior.
- Deployment rollout settings.
- Whether the shutdown process finishes within the grace period.

### Problem 3: Pod is stuck in Terminating

```bash
kubectl describe pod graceful-pod
kubectl get nodes
kubectl get events --sort-by=.metadata.creationTimestamp
```

Investigate node connectivity, volume unmounting, finalizers, and kubelet or runtime issues.

Caution: Force-deleting a Pod removes the API object without waiting for normal graceful termination. It does not guarantee that the process has stopped on an unreachable node. Use force deletion only when you understand the risks.

## 11. Interview Questions and Answers

**Q1. What is graceful shutdown in Kubernetes?**
It is the process of allowing an application to finish work and exit cleanly before Kubernetes forcefully terminates its container.

**Q2. What is the default termination grace period?**
30 seconds, unless configured differently.

**Q3. What is the purpose of a PreStop hook?**
It performs an operation before the container receives its normal termination signal, for example, initiating application-specific shutdown or draining.

**Q4. What is the difference between SIGTERM and SIGKILL?**
SIGTERM requests graceful termination and can be handled by the application. SIGKILL forcibly terminates the process and cannot be caught or ignored.

**Q5. Does PreStop run after SIGTERM?**
No. The PreStop hook runs before the normal termination signal is sent to the container.

**Q6. What happens when the termination grace period expires?**
If the container has not exited, Kubernetes typically forcefully terminates it.

**Q7. Why is graceful shutdown important during rolling updates?**
It helps reduce interrupted requests and allows old application instances to finish work while replacement instances become available.

## 12. Quick Revision

- Pod termination begins when Kubernetes is asked to terminate a Pod.
- The default `terminationGracePeriodSeconds` is 30 seconds.
- A PreStop hook runs before normal termination signalling.
- SIGTERM requests graceful termination.
- SIGKILL forcefully terminates a process.
- The grace period includes the time spent executing the PreStop hook.
- Readiness probes, connection draining, and application shutdown handling help reduce disruption.
- Kubernetes and OpenShift use the same core Pod termination behavior.
