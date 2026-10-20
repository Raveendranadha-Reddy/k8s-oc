# Kubernetes Notes: Pod Phases vs Container States

## 1. Introduction

In Kubernetes, Pod phases and container states describe two different levels of an application's lifecycle.

- Pod phase: Describes the overall lifecycle phase of a Pod.
- Container state: Describes the current state of an individual container inside that Pod.

A Pod can contain one or more containers, and each container has its own state.

## 2. Pod Phases

Kubernetes defines five Pod phases.

| Pod Phase | Meaning |
|---|---|
| Pending | Pod has been accepted, but its containers are not all running yet. |
| Running | Pod is bound to a node, and at least one container is running, starting, or restarting. |
| Succeeded | All containers have terminated successfully and will not be restarted. |
| Failed | All containers have terminated, and at least one terminated unsuccessfully. |
| Unknown | Kubernetes cannot determine the Pod's state, often because of communication problems with the node. |

Command:

```bash
kubectl get pods
```

Example output:

```
NAME          READY   STATUS      RESTARTS
nginx-pod     1/1     Running     0
job-pod       0/1     Completed   0
```

Note: `Completed` is a human-readable status commonly shown for a Pod whose phase is `Succeeded`. It is not a separate Pod phase.

## 3. Container States

Kubernetes defines three main container states.

| Container State | Meaning |
|---|---|
| Waiting | Container has not started running or is waiting for a condition. |
| Running | Container process is running. |
| Terminated | Container process has stopped, either successfully or unsuccessfully. |

Example container status:

```yaml
containerStatuses:
  - name: nginx
    state:
      running:
        startedAt: "2026-10-18T10:00:00Z"
```

To inspect the actual state:

```bash
kubectl get pod nginx-pod -o yaml
kubectl describe pod nginx-pod
```

## 4. Main Differences

| Feature | Pod Phases | Container States |
|---|---|---|
| Level | Pod level | Individual container level |
| Number of defined states | 5 phases | 3 states |
| Describes | Overall Pod lifecycle | Individual container lifecycle |
| Examples | Pending, Running, Succeeded | Waiting, Running, Terminated |
| Where to inspect | `status.phase` | `status.containerStatuses[].state` |
| Main purpose | Understand the Pod's overall condition | Understand what each container is doing |

## 5. How They Work Together

Illustrative example: a Pod can have phase Running while one container is Running and another is Waiting, such as during a restart.

### Scenario 1: Pod is Pending

A Pod has been created but is waiting for a suitable node or another prerequisite. Its container may not have been created yet, so a container state may not be available.

### Scenario 2: Pod is Running, but a container is Waiting

One container may be running while another is waiting for an image pull or restarting. The Pod can still have phase Running.

### Scenario 3: Pod is Succeeded

All containers have terminated successfully, and the Pod's restart policy will not restart them. This commonly occurs with completed Jobs.

### Scenario 4: Pod is Failed

All containers have terminated, and at least one terminated unsuccessfully. This commonly occurs when a Job's Pod fails permanently.

## 6. Practical Example

Create a Pod that runs a command and exits successfully:

```bash
kubectl run test-job \
  --image=busybox \
  --restart=Never \
  -- /bin/sh -c 'echo Hello Kubernetes'
```

Check its Pod phase:

```bash
kubectl get pod test-job
```

Inspect the container state and exit code:

```bash
kubectl get pod test-job -o yaml
kubectl describe pod test-job
kubectl logs test-job
```

Expected result after successful completion:

- Pod phase: Succeeded
- Container state: Terminated
- Exit code: 0

## 7. Interview Questions

**Q1. What is the difference between a Pod phase and a container state?**
A Pod phase describes the overall lifecycle of a Pod, while a container state describes the current condition of an individual container inside it.

**Q2. Can a Pod be Running while a container is Waiting?**
Yes. For example, one container may be running while another container in the same Pod is waiting or restarting.

**Q3. Is CrashLoopBackOff a Pod phase?**
No. It is a waiting/restart condition associated with a container. The Pod's phase may still be Running.

**Q4. What is the container state when a Job completes successfully?**
The container is Terminated with exit code 0, and the Pod normally reaches phase Succeeded.

**Q5. How do you check the Pod phase and container state?**

```bash
kubectl get pod nginx-pod -o jsonpath='{.status.phase}'
kubectl get pod nginx-pod -o jsonpath='{.status.containerStatuses[*].state}'
```

## 8. Quick Revision

- Pod phases: Pending, Running, Succeeded, Failed, Unknown.
- Container states: Waiting, Running, Terminated.
- Pod phase describes the Pod; container state describes each container.
- `Completed` is a displayed status commonly associated with the Succeeded phase.
- CrashLoopBackOff is not a Pod phase or one of the three main container states.
- The same concepts apply to Kubernetes and OpenShift.
