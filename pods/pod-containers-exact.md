# Kubernetes Notes: Pod Containers

## 1. What Is a Container in Kubernetes?

A container is a running instance of an application packaged with its code, libraries, dependencies, and runtime requirements.

In Kubernetes, containers run inside Pods. A Pod is the smallest deployable unit in Kubernetes and can contain one or more containers.

Example: In your CoffeeShop project, the frontend can run in one container and the Spring Boot backend can run in another container, usually in separate Pods.

## 2. Types of Pod Container Patterns

Kubernetes supports several common container patterns.

| Pattern | Description | Example |
|---|---|---|
| Single-container Pod | One main container in a Pod | NGINX web server |
| Multi-container Pod | Multiple containers sharing a Pod | Application + logging sidecar |
| Sidecar pattern | Supporting container assists the main application | Log collector |
| Init container | Container runs setup tasks before application containers start | Download configuration |
| Ephemeral container | Temporary container used for troubleshooting | Debug a running Pod |

Important: Init containers and ephemeral containers have different lifecycle rules from regular application containers.

## 3. Single-Container Pod

A single-container Pod contains one application container.

### Example YAML

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
    - name: nginx
      image: nginx:latest
      ports:
        - containerPort: 80
```

### Commands

```bash
kubectl apply -f nginx-pod.yaml
kubectl get pods
kubectl describe pod nginx-pod
kubectl logs nginx-pod
```

Here, the NGINX container serves web content on port 80 inside the Pod's network namespace.

## 4. Multi-Container Pod

A multi-container Pod contains two or more containers that are designed to work closely together.

Containers in the same Pod share:

- The same network namespace and Pod IP address.
- The same port space, so they must avoid conflicting listening ports.
- Volumes explicitly mounted by the relevant containers.

They do not automatically share their filesystems or process namespaces.

### Example: Application and Sidecar

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-pod
spec:
  containers:
    - name: app
      image: busybox
      command: ["/bin/sh", "-c"]
      args: ["while true; do date >> /var/log/app.log; sleep 5; done"]
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log
    - name: log-reader
      image: busybox
      command: ["/bin/sh", "-c"]
      args: ["tail -f /var/log/app.log"]
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log
  volumes:
    - name: shared-logs
      emptyDir: {}
```

In this example:

- `app` writes timestamps to a log file.
- `log-reader` reads the same file.
- `emptyDir` provides shared storage for both containers.
- The volume exists for the lifetime of the Pod and is removed when the Pod is removed.

### Verify

```bash
kubectl apply -f multi-container-pod.yaml
kubectl get pods
kubectl logs multi-container-pod -c app
kubectl logs multi-container-pod -c log-reader
```

Use `-c` to specify which container's logs you want to inspect.

## 5. Container Ports and Networking

Containers in the same Pod share one IP address.

For example:

```yaml
containers:
  - name: frontend
    image: nginx:latest
    ports:
      - containerPort: 80
  - name: helper
    image: busybox
    command: ["/bin/sh", "-c", "sleep 3600"]
```

The frontend can listen on port 80, and another container can communicate with it through `localhost:80` because both share the Pod network namespace.

Important: Declaring `containerPort` documents a port in the Pod specification; it does not create a Service or expose the application externally.

## 6. Container Commands and Arguments

You can override the image's default startup command using `command` and `args`.

```yaml
containers:
  - name: app
    image: busybox
    command: ["/bin/sh", "-c"]
    args: ["echo Hello Kubernetes; sleep 3600"]
```

- `command` overrides the image's entrypoint.
- `args` supplies arguments to the command.

This is useful for testing, utility containers, and running simple tasks.

## 7. Container Resources

CPU and memory requests and limits help Kubernetes schedule workloads and constrain container resource use.

```yaml
containers:
  - name: nginx
    image: nginx:latest
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "500m"
        memory: "256Mi"
```

- `requests` are used primarily for scheduling and resource allocation decisions.
- CPU limits constrain CPU usage.
- Memory limits can cause a container to be terminated with an out-of-memory condition if it exceeds its limit.

## 8. Container Environment Variables

Environment variables pass configuration into a container.

```yaml
containers:
  - name: app
    image: nginx:latest
    env:
      - name: APP_ENV
        value: "development"
```

For passwords, tokens, and other sensitive values, use Kubernetes Secrets or an appropriate external secret-management system rather than hardcoding them in a manifest.

## 9. Troubleshooting Containers

### Check Pod and container status

```bash
kubectl get pods
kubectl describe pod multi-container-pod
```

### View logs from a specific container

```bash
kubectl logs multi-container-pod -c app
kubectl logs multi-container-pod -c log-reader
```

### View previous container logs after a restart

```bash
kubectl logs multi-container-pod -c app --previous
```

### Execute a command inside a container

```bash
kubectl exec -it multi-container-pod -c app -- /bin/sh
```

The image must contain the shell you request. Minimal or distroless images may not include `/bin/sh`.

### OpenShift equivalents

```bash
oc get pods
oc describe pod multi-container-pod
oc logs multi-container-pod -c app
oc exec -it multi-container-pod -c app -- /bin/sh
```

## 10. Interview Questions

**Q1. Can a Pod contain multiple containers?**
Yes. A Pod can contain multiple containers that work closely together and share networking and explicitly mounted volumes.

**Q2. Do containers in the same Pod have different IP addresses?**
Normally, no. They share the Pod's network namespace and IP address.

**Q3. How do you view logs from a particular container?**
Use `kubectl logs POD_NAME -c CONTAINER_NAME`.

**Q4. What is a sidecar container?**
A supporting container that runs alongside the main application container to provide functionality such as logging, proxying, or monitoring.

**Q5. Do containers in the same Pod share their filesystems?**
Not automatically. They can share files through volumes mounted into each container.

**Q6. What happens if one regular container in a multi-container Pod fails?**
The result depends on the Pod's restart policy, the container's exit status, and any controller managing the Pod. Other containers in the Pod may continue running.

**Q7. Should frontend and backend containers always be placed in the same Pod?**
No. Frontend and backend applications usually run in separate Deployments and Pods so they can scale, update, and recover independently. Multi-container Pods are best suited to tightly coupled components.

## 11. Quick Revision

- Containers run inside Pods.
- A Pod can have one or more regular application containers.
- Containers in the same Pod share networking but not automatically their filesystems.
- Volumes provide shared files when mounted into multiple containers.
- Use `kubectl logs -c` and `kubectl exec -c` to target a specific container.
- Use resource requests and limits to manage CPU and memory.
- Use separate Deployments for independently scalable application components.
- The same core container concepts apply to Kubernetes and OpenShift.
