# Kubernetes and OpenShift Notes: Container Name and Image

## 1. What is a Container Name?

A container name identifies a specific container inside a Pod.

Example:

```yaml
containers:
  - name: nginx-container
    image: nginx:1.28
```

Here, `nginx-container` is the container name.

Why do we use a container name?

- To identify a container inside a Pod.
- To view logs for a specific container.
- To execute commands inside a specific container.
- To distinguish containers in a multi-container Pod.

Example command:

```bash
kubectl logs nginx-pod -c nginx-container
```

In OpenShift:

```bash
oc logs nginx-pod -c nginx-container
```

The `-c` option specifies the container name.

## 2. What is a Container Image?

A container image is a packaged template used to create a container.

It contains the application, required libraries, dependencies, and filesystem content needed to run the application.

Example:

```yaml
image: nginx:1.28
```

Here:

- `nginx` is the image name.
- `1.28` is the image tag.

The image can be downloaded from a container registry, such as Docker Hub or a private registry.

Examples:

```yaml
image: nginx:1.28
image: redis:7
image: busybox:1.36
```

## 3. Difference Between Container Name and Image

Container name identifies the container inside the Pod.

Image specifies the package used to create that container.

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
    - name: web-server
      image: nginx:1.28
```

In this example:

- Pod name: nginx-pod
- Container name: web-server
- Container image: nginx:1.28
- Application: Nginx web server

Remember: The container name and image name do not have to be the same.

## 4. How Does It Work?

1. The developer specifies the container name and image in the YAML file.
2. The API Server validates the Pod configuration.
3. The Scheduler assigns the Pod to a suitable worker node.
4. The Kubelet asks the container runtime to create the container.
5. The runtime obtains the image if necessary.
6. The runtime starts the container using that image.
7. The application runs inside the container.

## 5. Complete YAML Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
    - name: nginx-container
      image: nginx:1.28
      ports:
        - containerPort: 80
```

## 6. Kubernetes Commands

Create the Pod:

```bash
kubectl apply -f pod.yaml
```

Check the Pod:

```bash
kubectl get pods
```

Check container details:

```bash
kubectl describe pod nginx-pod
```

Check container logs:

```bash
kubectl logs nginx-pod -c nginx-container
```

Access the container:

```bash
kubectl exec -it nginx-pod -c nginx-container -- /bin/sh
```

## 7. OpenShift Commands

Create the Pod:

```bash
oc apply -f pod.yaml
```

Check the Pod:

```bash
oc get pods
```

Check container details:

```bash
oc describe pod nginx-pod
```

Check container logs:

```bash
oc logs nginx-pod -c nginx-container
```

Access the container:

```bash
oc exec -it nginx-pod -c nginx-container -- /bin/sh
```

## 8. Common Errors

**ImagePullBackOff:** The cluster cannot pull the image successfully.

Possible causes:

- Incorrect image name or tag.
- Private registry authentication failure.
- Network connectivity problems.

**CrashLoopBackOff:** The container repeatedly exits and Kubernetes retries starting it.

Possible causes:

- Application startup failure.
- Incorrect command or configuration.
- Missing dependencies.

**Invalid container name:** The name does not meet Kubernetes naming requirements or conflicts with another container name in the same Pod.

## 9. Interview Questions

**Q1. What is a container name?**
It identifies a container inside a Pod.

**Q2. What is a container image?**
It is a packaged template used to create a container.

**Q3. Can the container name and image name be different?**
Yes. For example, the container name can be `web-server` and the image can be `nginx:1.28`.

**Q4. What does nginx:1.28 mean?**
It identifies the Nginx image with the tag 1.28.

**Q5. Can two containers inside one Pod use the same image?**
Yes. They can use the same image but must have different container names.

**Q6. Are container names and images different in OpenShift?**
No. Both Kubernetes and OpenShift use the same basic container name and image fields.

## 10. Quick Revision

- Container name = Identifies the container.
- Container image = Provides the packaged application and dependencies.

Example:

```yaml
name: web-server
image: nginx:1.28
```

The name identifies the container, while the image determines what software is used to create it.
