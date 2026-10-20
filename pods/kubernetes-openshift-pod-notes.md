# ☸️ Kubernetes and OpenShift – Pod Notes

**Complete Pod fundamentals, workflow, YAML, commands, troubleshooting, and platform comparison**

---

## 🟦 1. What is a Pod?

A Pod is the smallest deployable unit in Kubernetes. It is used to run application containers.

A Pod can contain one or more containers. Usually, one application container runs inside a Pod. Sometimes, supporting containers such as sidecars are also used.

**Example: In the CoffeeShop project**

- **a)** Frontend application runs in a frontend Pod.
- **b)** Backend application runs in a backend Pod.

---

## 🟩 2. What is a Pod called in Kubernetes and OpenShift?

**Kubernetes:** Pod

**OpenShift:** Pod

The name is the same in both platforms because OpenShift is built on Kubernetes.

Both platforms use similar Pod YAML files and Kubernetes resources.

- **a)** Kubernetes CLI: `kubectl`
- **b)** OpenShift CLI: `oc`

OpenShift also supports `kubectl` for common Kubernetes operations.

---

## 🟪 3. How Does a Pod Work?

**Workflow**

Developer → YAML File → API Server → Scheduler → Worker Node → Kubelet → Container Runtime → Container

**Step-by-step explanation**

1. **a)** The developer creates a Pod YAML file.
2. **b)** The API Server validates the request and stores the desired cluster state.
3. **c)** The Scheduler selects a suitable worker node for the Pod.
4. **d)** The Kubelet on that node works with the container runtime to create the container.
5. **e)** The container image is used to start the application.
6. **f)** The Pod enters the Running phase when its containers are running. The application becomes Ready when its readiness conditions are satisfied.

OpenShift follows the same basic workflow. It also provides security features such as Security Context Constraints (SCC) to control how Pods can run.

---

## 🟧 4. Pod YAML Example

The following YAML works on both Kubernetes and OpenShift, subject to cluster policies and image availability.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
    - name: nginx
      image: nginx:1.28
      ports:
        - containerPort: 80
```

### Important YAML Fields

- **a) `apiVersion: v1`** – Specifies the API version for the Pod resource.
- **b) `kind: Pod`** – Specifies the resource type.
- **c) `metadata.name`** – Defines the Pod name.
- **d) `spec`** – Defines the desired Pod configuration.
- **e) `containers`** – Lists the containers inside the Pod.
- **f) `name`** – Defines the container name.
- **g) `image`** – Specifies the container image.
- **h) `containerPort: 80`** – Documents the port the container is expected to use.

---

## 🟦 5. How to Create a Pod in Kubernetes

**Apply the YAML file**

```bash
kubectl apply -f pod.yaml
```

**Check the Pods**

```bash
kubectl get pods
```

**Get detailed information**

```bash
kubectl describe pod nginx-pod
```

**Check application logs**

```bash
kubectl logs nginx-pod
```

**Access the container shell**

```bash
kubectl exec -it nginx-pod -- /bin/sh
```

**Delete the Pod**

```bash
kubectl delete pod nginx-pod
```

---

## 🟩 6. How to Create a Pod in OpenShift

**Log in to the OpenShift cluster**

```bash
oc login <cluster-url>
```

**Create a project**

```bash
oc new-project pod-demo
```

**Apply the YAML file**

```bash
oc apply -f pod.yaml
```

**Check the Pods**

```bash
oc get pods
```

**Get detailed information**

```bash
oc describe pod nginx-pod
```

**Check application logs**

```bash
oc logs nginx-pod
```

**Access the container shell**

```bash
oc exec -it nginx-pod -- /bin/sh
```

**Delete the Pod**

```bash
oc delete pod nginx-pod
```

**Note:** If the YAML does not specify a namespace, the Pod is created in the current namespace or OpenShift project.

---

## 🟪 7. Pod Lifecycle and Troubleshooting

### Pending

The Pod has not finished scheduling or setup. Possible reasons include insufficient resources or scheduling constraints.

### ContainerCreating

The container is being prepared. Image downloads, volume mounts, and networking setup may be in progress.

### Running

The Pod has been assigned to a node, and its containers are running or starting. Running does not necessarily mean the application is ready.

### CrashLoopBackOff

The container repeatedly crashes and Kubernetes waits before trying to restart it again.

**Possible causes**

- **a)** Application errors
- **b)** Incorrect configuration
- **c)** Missing environment variables
- **d)** Incorrect startup commands

### ImagePullBackOff

The cluster cannot pull the container image successfully.

**Possible causes**

- **a)** Incorrect image name or tag
- **b)** Private registry authentication issues
- **c)** Network connectivity problems

### Succeeded

All containers have completed successfully. This is common for Jobs and other short-lived workloads.

### Unknown

The control plane cannot reliably determine the Pod's state, often because of communication problems with the node.

---

## 🟧 8. Pod Troubleshooting Commands

### Kubernetes

```bash
kubectl get pods -o wide
kubectl describe pod nginx-pod
kubectl logs nginx-pod --previous
kubectl get events --sort-by=.metadata.creationTimestamp
```

### OpenShift

```bash
oc get pods -o wide
oc describe pod nginx-pod
oc logs nginx-pod --previous
oc get events --sort-by=.metadata.creationTimestamp
```

The `--previous` option displays logs from the previous container instance, if available.

---

## 🟥 9. Important Interview Point

A Pod is not a permanent resource. If a standalone Pod fails or is deleted, Kubernetes does not automatically recreate it.

In production, we generally use a Deployment to manage application Pods. If a managed Pod fails or is deleted, the Deployment controller helps maintain the desired number of replicas by creating replacement Pods.

Replacement Pods may have different names and IP addresses.

---

## 🟦 10. Kubernetes vs OpenShift Summary

| Feature | Kubernetes | OpenShift |
|---|---|---|
| Smallest deployable unit | Pod | Pod |
| CLI | `kubectl` | `oc` |
| Pod YAML | Kubernetes YAML | Kubernetes YAML |
| Workload management | Deployment | Deployment |
| Security | Kubernetes security controls | Kubernetes controls plus OpenShift features such as SCC |
| Namespace terminology | Namespace | Project is the common user-facing term |

---

## 🟩 Final Summary

A Pod is the smallest deployable unit in Kubernetes and OpenShift. It runs one or more containers and provides their shared execution environment.

The basic Pod creation and lifecycle processes are the same in both platforms. OpenShift adds platform-specific security and developer capabilities.

For production applications, use Deployments or other appropriate workload controllers instead of relying on standalone Pods.
