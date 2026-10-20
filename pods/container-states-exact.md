# Kubernetes Notes: Container States

## 1. What Are Container States?

Container states describe the current condition of a container running inside a Kubernetes Pod.

Kubernetes uses the kubelet and container runtime to manage containers and report their status to the Kubernetes API server.

Kubernetes defines three main container states:

1. Waiting
2. Running
3. Terminated

## 2. Waiting State

### Definition

A container is in the Waiting state when it has not started running yet or is waiting for a condition before it can start.

### Common reasons

- Container image is being downloaded.
- Image name or tag is incorrect.
- Registry authentication failed.
- Required configuration is unavailable.
- Container startup is waiting for another prerequisite.

### Example

```bash
kubectl get pods
```

Output:

```
NAME         READY   STATUS             RESTARTS
nginx-pod    0/1     ImagePullBackOff   0
```

Here, `ImagePullBackOff` indicates that Kubernetes is repeatedly failing to pull the image and is delaying further attempts.

### Troubleshooting

```bash
kubectl describe pod nginx-pod
kubectl get events
```

Check the Events section for the exact reason.

Remember: `ImagePullBackOff` and `CrashLoopBackOff` are common waiting reasons, not separate container states.

## 3. Running State

### Definition

A container is in the Running state when the container has been started and is running.

This does not automatically mean that the application is healthy or ready to receive traffic.

### Example

```bash
kubectl run nginx-pod --image=nginx
```

Verify:

```bash
kubectl get pods
```

Output:

```
NAME         READY   STATUS    RESTARTS
nginx-pod    1/1     Running   0
```

Inspect the container status:

```bash
kubectl describe pod nginx-pod
kubectl get pod nginx-pod -o yaml
```

Example status fragment:

```yaml
containerStatuses:
  - name: nginx
    state:
      running:
        startedAt: "2026-10-18T10:00:00Z"
```

### Important point

A container can be Running while its readiness probe is failing. In that situation, the container is running, but the Pod may not be ready to receive Service traffic.

## 4. Terminated State

### Definition

A container is in the Terminated state when its process has finished executing or has stopped.

The container may have exited successfully or unsuccessfully.

### Common reasons

- Application completed its task.
- Application crashed.
- Process exited with an error.
- Container received a termination signal.
- Container was stopped during a Pod termination.

### Example: Successful completion

Create a Pod that runs a command and exits:

```bash
kubectl run completed-task \
  --image=busybox \
  --restart=Never \
  -- /bin/sh -c 'echo Task completed'
```

Check:

```bash
kubectl get pod completed-task
kubectl describe pod completed-task
```

The container should terminate with exit code 0, assuming the command runs successfully.

### Example: Inspect the exit code

```bash
kubectl get pod completed-task -o yaml
```

Example status fragment:

```yaml
containerStatuses:
  - name: completed-task
    state:
      terminated:
        exitCode: 0
        reason: Completed
```

Exit codes:

- `0` — normally indicates successful execution.
- Non-zero — normally indicates an error or unsuccessful execution.

A terminated container may be restarted depending on the Pod's `restartPolicy`. For example, a container in a Pod with `restartPolicy: Always` is normally restarted even if it exits with code 0.

## 5. Comparison of Container States

| Feature | Waiting | Running | Terminated |
|---|---|---|---|
| Meaning | Not running yet or waiting for a condition | Container process is running | Container process has stopped |
| Application executing? | No | Yes | No |
| Common example | ImagePullBackOff | Application running | Process exited |
| Useful diagnostic | kubectl describe pod | kubectl logs | kubectl logs --previous |
| Can transition to another state? | Yes | Yes | Yes, if restarted |

## 6. Container State vs Pod Phase

These are different concepts and are frequently asked about in interviews.

| Container state | Pod phase example |
|---|---|
| Waiting | Pending or Running |
| Running | Running |
| Terminated | Succeeded or Failed; a restarting container may leave the Pod Running |

The mapping is not one-to-one. A Pod's phase depends on the status of its containers and other lifecycle conditions.

For example, a Pod may have phase Running while one of its containers is in Waiting because it is restarting.

## 7. Practical Troubleshooting Commands

**Check Pod status**

```bash
kubectl get pods -o wide
```

**View detailed container state and events**

```bash
kubectl describe pod nginx-pod
```

**View current container logs**

```bash
kubectl logs nginx-pod
```

**View logs from the previous container instance**

```bash
kubectl logs nginx-pod --previous
```

**Inspect container state in YAML**

```bash
kubectl get pod nginx-pod -o yaml
```

OpenShift equivalents:

```bash
oc get pods
oc describe pod nginx-pod
oc logs nginx-pod
oc logs nginx-pod --previous
oc get pod nginx-pod -o yaml
```

## 8. Interview Questions and Answers

**Q1. What are the three container states in Kubernetes?**
Waiting, Running, and Terminated.

**Q2. What is the Waiting state?**
The container has not started running or is waiting for a prerequisite, such as successful image retrieval.

**Q3. What is the difference between ImagePullBackOff and CrashLoopBackOff?**
ImagePullBackOff means Kubernetes cannot successfully pull the container image. CrashLoopBackOff means the container repeatedly exits and Kubernetes is delaying restart attempts.

**Q4. Does Running mean the application is healthy?**
No. Running indicates that the container process is running. Readiness and liveness probes help assess application readiness and health.

**Q5. What does exit code 0 mean?**
It normally indicates that the container process completed successfully.

**Q6. How do you check why a container terminated?**
Use `kubectl describe pod`, inspect the container's terminated state and exit code, and check its logs. Use `kubectl logs --previous` if it restarted and the previous container logs are available.

**Q7. What happens after a container terminates?**
The result depends on the Pod's `restartPolicy` and whether a controller manages the Pod. Kubernetes may restart the container or the controller may create a replacement Pod.

## 9. Quick Revision

- Waiting: Container has not started or is waiting for a condition.
- Running: Container process is running.
- Terminated: Container process has stopped.
- ImagePullBackOff is associated with image-pull failures.
- CrashLoopBackOff is associated with repeated container failures and delayed restarts.
- Use `kubectl describe pod` for state details and events.
- Use `kubectl logs --previous` to inspect a previous container instance.
- Container state and Pod phase are not the same.
