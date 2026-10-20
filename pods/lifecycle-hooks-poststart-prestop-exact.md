# Kubernetes Notes: PreStart, PostStart and PreStop Lifecycle Hooks

## 1. What Are Container Lifecycle Hooks?

Kubernetes container lifecycle hooks allow you to execute actions at specific points in a container's lifecycle.

The two main lifecycle hooks are:

1. PostStart — executes after a container is created.
2. PreStop — executes immediately before a container is terminated.

These hooks are configured inside the container's `lifecycle` section in a Pod YAML file.

Important: Kubernetes has a PostStart hook, not a PreStart hook.

## 2. PostStart Hook

### Definition

The PostStart hook is executed after a container is created. However, Kubernetes does not guarantee that the container's entrypoint starts before the hook runs.

The hook and the container entrypoint can execute concurrently.

### Use cases

- Perform initialization tasks.
- Create required directories.
- Execute a small setup script.
- Trigger an application initialization endpoint.

### Example YAML

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: poststart-pod
spec:
  containers:
    - name: nginx
      image: nginx:latest
      lifecycle:
        postStart:
          exec:
            command:
              - /bin/sh
              - -c
              - "echo 'PostStart hook executed' >> /tmp/lifecycle.log"
```

### Create and verify

```bash
kubectl apply -f poststart-pod.yaml
kubectl get pod poststart-pod
kubectl exec poststart-pod -- cat /tmp/lifecycle.log
```

Expected output:

```
PostStart hook executed
```

Important points:

- The hook runs after container creation, not necessarily after the application is ready.
- If the hook fails, the container can be terminated.
- The application may start before or after the hook finishes.
- Do not use PostStart as a substitute for a readiness probe.

## 3. PreStop Hook

### Definition

The PreStop hook executes immediately before Kubernetes sends the normal termination signal to the container.

It is commonly used to prepare an application for shutdown.

### Use cases

- Stop accepting new requests.
- Start application-specific connection draining.
- Notify the application to shut down.
- Perform cleanup before termination.

### Example YAML

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
            command:
              - /bin/sh
              - -c
              - "echo 'PreStop hook executed' >> /tmp/lifecycle.log; sleep 5"
```

### Create and verify

```bash
kubectl apply -f prestop-pod.yaml
kubectl get pod prestop-pod
```

Delete the Pod:

```bash
kubectl delete pod prestop-pod
```

The hook runs during termination. Because the Pod is deleted afterward, you may not be able to inspect its log file once it has disappeared. For a real shutdown investigation, collect logs through your logging system or use an application that records shutdown events externally.

Important points:

- The hook runs before the normal termination signal.
- The hook's execution time counts toward `terminationGracePeriodSeconds`.
- If the hook fails, Kubernetes still proceeds with container termination.
- A delay such as `sleep 5` does not automatically drain traffic or guarantee that ongoing requests finish.

## 4. PostStart vs PreStop

| Feature | PostStart | PreStop |
|---|---|---|
| When it runs | After container creation | Before normal container termination signalling |
| Main purpose | Initialization | Shutdown preparation |
| Typical use | Setup or initialization tasks | Cleanup or connection draining |
| Runs during startup? | Yes | No |
| Runs during termination? | No | Yes |
| Can delay progress? | Yes, if the hook takes time | Yes, within the termination grace period |
| Failure impact | Can cause container termination | Does not prevent termination from proceeding |

## 5. Lifecycle Flow

1. **Container created**
2. **PostStart hook** — May run concurrently with the application entrypoint.
3. **Container runs** — Readiness and liveness probes can monitor application behavior.
4. **Pod termination requested**
5. **PreStop hook** — Runs before the normal termination signal.
6. **SIGTERM → graceful exit** — If the process does not exit in time, it is typically forcefully terminated.

## 6. Types of Lifecycle Hook Handlers

Kubernetes supports two main handler types for container lifecycle hooks.

### A. exec

Executes a command inside the container.

```yaml
lifecycle:
  postStart:
    exec:
      command: ["/bin/sh", "-c", "echo started"]
```

### B. httpGet

Sends an HTTP request to a specified endpoint.

```yaml
lifecycle:
  postStart:
    httpGet:
      path: /initialize
      port: 8080
```

The HTTP endpoint must be implemented by the application and accessible from the container's network context.

Note: Lifecycle hooks also support a sleep handler in Kubernetes versions that support it. The handler options available depend on the Kubernetes version.

## 7. Real-Time Scenario: CoffeeShop Application

Suppose your CoffeeShop backend is a Spring Boot application.

### PostStart example

You may call an application initialization endpoint after container creation:

```yaml
lifecycle:
  postStart:
    httpGet:
      path: /internal/initialize
      port: 8080
```

The endpoint must exist and be safe to call at that point. Because PostStart may run before the application is ready, this approach can fail if the application has not started listening yet.

### PreStop example

Before shutting down the backend, the hook could invoke an application-specific endpoint to begin draining requests:

```yaml
lifecycle:
  preStop:
    httpGet:
      path: /internal/shutdown
      port: 8080
```

The application must implement the endpoint and handle the shutdown process correctly. Kubernetes then sends the normal termination signal after the hook completes.

For Spring Boot, also configure the application to handle graceful shutdown appropriately. A lifecycle hook alone does not guarantee zero downtime.

## 8. OpenShift Commands

The same lifecycle hooks work in OpenShift because OpenShift uses Kubernetes Pod specifications.

```bash
oc apply -f poststart-pod.yaml
oc get pods
oc describe pod poststart-pod
oc logs poststart-pod
oc exec poststart-pod -- cat /tmp/lifecycle.log
```

To delete the test Pod and trigger PreStop:

```bash
oc delete pod prestop-pod
```

## 9. Interview Questions and Answers

**Q1. What are container lifecycle hooks in Kubernetes?**
They allow actions to run at specific points in a container's lifecycle. The main hooks are PostStart and PreStop.

**Q2. What is the difference between PostStart and PreStop?**
PostStart runs after container creation. PreStop runs before the normal termination signal is sent.

**Q3. Does PostStart run after the application is ready?**
No. It may run concurrently with the container entrypoint, so application readiness must be checked separately.

**Q4. What happens if a PostStart hook fails?**
Kubernetes can terminate the container, and the container may be restarted according to the Pod's restart policy.

**Q5. Does PreStop get extra time beyond the termination grace period?**
Normally, no. The hook's execution time is included in the grace period, although Kubernetes may grant a small one-off extension in certain circumstances.

**Q6. Can lifecycle hooks guarantee zero-downtime deployments?**
No. They can help with initialization and shutdown, but reliable deployments also require readiness checks, suitable rollout settings, application-level graceful shutdown, and appropriate traffic draining.

**Q7. Is there a PreStart hook?**
No. The standard Kubernetes container lifecycle hooks are PostStart and PreStop.

## 10. Quick Revision

- PostStart: runs after container creation; it may run concurrently with the entrypoint.
- PreStop: runs before the normal termination signal.
- Both are configured under `spec.containers[].lifecycle`.
- Supported common handlers include `exec` and `httpGet`.
- Hook failure does not mean Kubernetes will leave the container running indefinitely.
- PreStop execution counts toward `terminationGracePeriodSeconds`.
- Use readiness probes for readiness and application-level logic for reliable graceful shutdown.
- The same concepts apply to Kubernetes and OpenShift.
