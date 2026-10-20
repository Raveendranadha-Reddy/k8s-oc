# Kubernetes and OpenShift Notes: restartPolicy

## 1. What is restartPolicy?

restartPolicy is a Pod configuration field that defines what Kubernetes should do when a container inside the Pod terminates.

In simple words, restartPolicy controls whether Kubernetes restarts a container after it exits.

Kubernetes and OpenShift use the same restartPolicy field.

## 2. Types of restartPolicy

Kubernetes supports three values:

### A. Always

Kubernetes restarts a container whenever it terminates, regardless of whether it exits successfully or with an error.

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: always-demo
spec:
  restartPolicy: Always
  containers:
    - name: app
      image: busybox:1.36
      command: ["/bin/sh", "-c"]
      args: ["echo Hello; sleep 5; exit 1"]
```

How it works:

- The container prints `Hello`.
- It waits five seconds.
- It exits with an error.
- Kubernetes attempts to restart the container.

Use case: Long-running applications such as web servers and APIs.

Note: `Always` is the default restart policy for Pods.

### B. OnFailure

Kubernetes restarts the container when it exits with a non-zero exit code.

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: onfailure-demo
spec:
  restartPolicy: OnFailure
  containers:
    - name: app
      image: busybox:1.36
      command: ["/bin/sh", "-c"]
      args: ["echo Processing; exit 1"]
```

How it works:

- The container executes the command.
- It exits with a non-zero exit code.
- Kubernetes attempts to restart the container.

Use case: Tasks that should retry after a failure.

### C. Never

Kubernetes does not automatically restart a terminated container within the Pod.

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: never-demo
spec:
  restartPolicy: Never
  containers:
    - name: app
      image: busybox:1.36
      command: ["/bin/sh", "-c"]
      args: ["echo Task completed; exit 0"]
```

How it works:

- The container prints the message.
- It exits successfully.
- Kubernetes does not restart it.

Use case: One-time tasks where the Pod should not retry a terminated container.

## 3. Difference Between the Three Policies

**Always:** Restart the container whenever it terminates.

**OnFailure:** Restart the container when it terminates unsuccessfully.

**Never:** Do not automatically restart the terminated container.

Important: These policies control container restarts within a Pod. They do not, by themselves, guarantee that a deleted Pod will be recreated.

## 4. How Does restartPolicy Work?

The workflow is:

Container starts → Application runs → Container exits → Kubelet checks restartPolicy → Container restarts or remains terminated.

The restart behavior also depends on the container's exit status and, for `Always`, `OnFailure`, and `Never`, the Pod's configured policy.

When a container repeatedly crashes, Kubernetes may show `CrashLoopBackOff` while delaying restart attempts.

## 5. What Is the Default restartPolicy?

If you do not specify restartPolicy, Kubernetes uses:

```yaml
restartPolicy: Always
```

This is the default for Pods.

## 6. restartPolicy in Kubernetes and OpenShift

| Feature | Kubernetes | OpenShift |
|---|---|---|
| Always | Supported | Supported |
| OnFailure | Supported | Supported |
| Never | Supported | Supported |
| Default | Always | Always |

The behavior is the same in both platforms.

## 7. How to Check Pod Restart Information

Kubernetes:

```bash
kubectl get pods
kubectl describe pod always-demo
kubectl get pod always-demo -o yaml
```

OpenShift:

```bash
oc get pods
oc describe pod always-demo
oc get pod always-demo -o yaml
```

To view the restart count:

```bash
kubectl get pods
```

Look at the RESTARTS column.

To inspect the previous container instance's logs:

```bash
kubectl logs always-demo --previous
```

In OpenShift, use `oc logs always-demo --previous`.

## 8. Important Production Concept: Pod vs Container Restart

A container restart and a Pod recreation are different operations.

**Container restart:**
The container is restarted inside the existing Pod. The Pod name and IP normally remain the same.

**Pod recreation:**
A controller such as a Deployment creates a replacement Pod. The replacement can have a different name and IP address.

For production applications, Deployments commonly manage Pod replicas and replace Pods when needed.

## 9. Important Interview Questions

**Q1. What is restartPolicy?**
It defines whether Kubernetes automatically restarts terminated containers inside a Pod.

**Q2. What are the three restartPolicy values?**
`Always`, `OnFailure`, and `Never`.

**Q3. Which is the default restartPolicy?**
`Always`.

**Q4. What is the difference between Always and OnFailure?**
`Always` restarts the container regardless of its exit status. `OnFailure` restarts it only when it exits unsuccessfully.

**Q5. Does restartPolicy recreate a deleted Pod?**
No. Pod recreation is generally handled by a workload controller, such as a Deployment or Job.

**Q6. Is restartPolicy different in OpenShift?**
No. OpenShift follows the same Kubernetes restartPolicy behavior.

## 10. Quick Revision

- Always = Restart whenever the container terminates.
- OnFailure = Restart after an unsuccessful exit.
- Never = Do not automatically restart the terminated container.

Remember: restartPolicy controls container restart behavior inside a Pod, while workload controllers manage the desired number of Pods.
