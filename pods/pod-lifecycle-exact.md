# Kubernetes Notes: Pod Lifecycle

## 1. What Is Pod Lifecycle?

Pod lifecycle describes the different phases a Pod goes through from creation until it terminates.

Kubernetes manages Pods through the Kubernetes API server, scheduler, controllers, and kubelet running on worker nodes.

Example: In a CoffeeShop application, Kubernetes creates a frontend Pod, schedules it on a worker node, and the kubelet starts its container.

## 2. Pod Lifecycle Phases

Kubernetes defines five main Pod phases:

| Phase | Description |
|---|---|
| Pending | Pod is accepted, but its containers are not all running yet. It may be waiting for scheduling, image downloads, or storage. |
| Running | Pod is bound to a node, and at least one container is running, starting, or restarting. |
| Succeeded | All containers in the Pod terminated successfully, and they will not be restarted. |
| Failed | All containers have terminated, and at least one terminated unsuccessfully. |
| Unknown | The API server cannot determine the Pod's state, often because communication with the node has failed. |

Important: Pod phases are not the same as container states. A container can be Waiting, Running, or Terminated while the Pod has its own phase.

## 3. Pod Lifecycle Flow

```
        Pod Created
             |
             v
          Pending
             |
             v
           Running
          /   |    \
         /    |     \
        v     v      v
   Succeeded Failed  Unknown
```

This is a simplified diagram. A Pod may remain Pending, and an Unknown state does not necessarily mean the Pod has terminated.

## 4. Step-by-Step Pod Lifecycle

### Step 1: Pod Creation

The user submits a Pod manifest to the API server.

```bash
kubectl apply -f pod.yaml
```

The API server validates and stores the Pod specification.

### Step 2: Scheduling

The scheduler selects a suitable worker node based on resource requests, scheduling rules, node selectors, taints, tolerations, and other constraints.

Check the Pod:

```bash
kubectl get pods
kubectl describe pod nginx-pod
```

If no suitable node is available, the Pod may remain Pending.

### Step 3: Container Setup

The kubelet on the selected node works with the container runtime to prepare the Pod.

Typical activities include:

- Setting up the Pod's sandbox and networking.
- Mounting required volumes.
- Pulling container images when necessary.
- Creating and starting containers.

### Step 4: Container Startup

The container runtime starts the application containers.

If configured, startup probes, readiness probes, and liveness probes influence how Kubernetes handles the application after startup.

### Step 5: Running

When the Pod has at least one container running, starting, or restarting, its phase can be Running.

Check:

```bash
kubectl get pods -o wide
kubectl get pods -w
```

### Step 6: Termination

A Pod may terminate because:

- A user deletes it.
- A Deployment rollout replaces its old Pod.
- A Job completes.
- A node failure or eviction affects it.

During graceful termination, Kubernetes normally sends a termination signal to the container and allows time for shutdown before forceful termination if necessary.

## 5. Pod Lifecycle vs Container Lifecycle

| Pod concept | Container concept |
|---|---|
| Pending | Waiting |
| Running | Running |
| Succeeded | Terminated with exit code 0, for all containers |
| Failed | All containers terminated, with at least one unsuccessful |
| Unknown | Pod state cannot be determined by the API server |

These are not exact one-to-one mappings. A Pod's phase is calculated from the state of its containers and other lifecycle conditions.

Container states:

- **Waiting:** Container has not started or is waiting for a condition, such as an image pull.
- **Running:** Container process is running.
- **Terminated:** Container process has exited.

## 6. Pod Restart Policy

The `restartPolicy` field controls whether containers are restarted within the same Pod.

Supported values:

- `Always` — restart a terminated container; default for Pods.
- `OnFailure` — restart a container if it exits unsuccessfully.
- `Never` — do not automatically restart a terminated container.

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-pod
spec:
  restartPolicy: Always
  containers:
    - name: nginx
      image: nginx:latest
```

Important: Restarting a container is different from recreating a Pod. A Deployment or another controller can create a replacement Pod when needed.

## 7. Startup, Readiness, and Liveness Probes

Probes help Kubernetes determine whether an application has started, can receive traffic, or should be restarted.

| Probe | Purpose |
|---|---|
| Startup probe | Checks whether the application has started successfully. |
| Readiness probe | Determines whether the container is ready to receive traffic. |
| Liveness probe | Detects an unhealthy container and can trigger a container restart. |

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: probe-pod
spec:
  containers:
    - name: nginx
      image: nginx:latest
      ports:
        - containerPort: 80
      readinessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 10
      livenessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 10
        periodSeconds: 15
```

Apply and verify:

```bash
kubectl apply -f probe-pod.yaml
kubectl describe pod probe-pod
kubectl get pods
```

A failed readiness probe removes the Pod from normal Service traffic while it is unready. A failed liveness probe can cause the affected container to be restarted.

## 8. Troubleshooting Pod Lifecycle Issues

### Scenario 1: Pod remains Pending

```bash
kubectl describe pod nginx-pod
kubectl get events --sort-by=.metadata.creationTimestamp
```

Possible causes:

- Insufficient CPU or memory.
- No suitable worker node.
- Unbound PersistentVolumeClaim.
- Taints or scheduling constraints.

### Scenario 2: ImagePullBackOff

```bash
kubectl describe pod nginx-pod
```

Possible causes:

- Incorrect image name or tag.
- Registry authentication failure.
- Network or registry access problems.

### Scenario 3: CrashLoopBackOff

```bash
kubectl logs nginx-pod
kubectl logs nginx-pod --previous
kubectl describe pod nginx-pod
```

Possible causes:

- Application crash.
- Incorrect configuration.
- Missing environment variables or dependencies.
- Failed liveness probe.

CrashLoopBackOff is a container waiting/restart condition, not a Pod phase.

### Scenario 4: Pod is Unknown

```bash
kubectl get pods -o wide
kubectl describe pod nginx-pod
kubectl get nodes
```

Investigate node health, kubelet status, and communication between the node and control plane.

## 9. OpenShift Commands

OpenShift uses the same core Pod lifecycle concepts.

```bash
oc get pods
oc get pods -o wide
oc describe pod nginx-pod
oc logs nginx-pod
oc logs nginx-pod --previous
oc get events --sort-by=.metadata.creationTimestamp
oc get nodes
```

To watch changes in Pod status:

```bash
oc get pods -w
```

## 10. Interview Questions and Answers

**Q1. What are the five Pod phases?**
Pending, Running, Succeeded, Failed, and Unknown.

**Q2. What is the difference between Pending and Running?**
Pending means the Pod has not reached the Running phase. It may be waiting for scheduling or container setup. Running means it is bound to a node and at least one container is running, starting, or restarting.

**Q3. What is CrashLoopBackOff?**
It indicates that a container is repeatedly failing and Kubernetes is delaying subsequent restart attempts.

**Q4. What happens when a Pod is deleted?**
Kubernetes terminates the Pod. If a controller such as a Deployment requires that replica, it normally creates a replacement Pod.

**Q5. What is the difference between a readiness probe and a liveness probe?**
Readiness controls whether a Pod is considered ready for Service traffic. Liveness detects a container that should be restarted.

**Q6. Does a Pod always move directly from Pending to Running?**
No. It may remain Pending, or it may fail before reaching Running. Its observed state depends on scheduling, setup, and container execution.

**Q7. Is CrashLoopBackOff a Pod phase?**
No. It is a container waiting/restart condition. The Pod's phase may still be Running.

## 11. Quick Revision

- Pod lifecycle describes the stages of a Pod from creation to termination.
- The five phases are Pending, Running, Succeeded, Failed, and Unknown.
- The scheduler selects a worker node.
- The kubelet coordinates Pod setup and container execution on that node.
- Container states are Waiting, Running, and Terminated.
- restartPolicy controls container restart behavior within a Pod.
- Readiness probes control traffic eligibility; liveness probes can trigger restarts.
- Deployments maintain the desired number of replicas and can recreate deleted Pods.
- The same core lifecycle concepts apply to Kubernetes and OpenShift.
