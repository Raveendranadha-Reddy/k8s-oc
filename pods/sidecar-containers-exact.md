# Kubernetes Notes: Sidecar Containers

## 1. What Is a Sidecar Container?

A sidecar container is a supporting container that runs alongside the main application container in the same Pod to provide additional functionality.

The main application handles the business logic, while the sidecar supports it with tasks such as logging, monitoring, proxying, or configuration updates.

Real-time example: In your CoffeeShop application, the main container runs the application, while a sidecar container collects application logs from a shared volume.

## 2. Sidecar Architecture

- **Kubernetes Pod:** Shared network namespace and Pod IP.
- **Main Container:** Runs the application.
- **Sidecar Container:** Collects logs or provides proxy services.
- Containers can exchange data through shared volumes or localhost networking.

The containers share the Pod's network namespace and can share storage through explicitly mounted volumes. They do not automatically share their filesystems.

## 3. Why Do We Use Sidecar Containers?

Common use cases include:

| Use case | Sidecar responsibility |
|---|---|
| Logging | Reads application log files and forwards logs to a logging system |
| Monitoring | Exposes or collects application metrics |
| Service mesh | Handles traffic through a proxy such as Envoy |
| Security | Provides supporting security or identity functions |
| Configuration | Retrieves or refreshes configuration files |
| Traffic management | Handles proxying, retries, or connection management |

Important: A sidecar is useful when supporting functionality should run alongside the application without being built into the main application container.

## 4. Practical Lab: Application + Logging Sidecar

In this lab, one container writes logs to a file, and a sidecar reads the same file.

### YAML: sidecar-pod.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-demo
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["/bin/sh", "-c"]
      args:
        - |
          while true; do
            echo "$(date): Application is running" >> /var/log/app/app.log
            sleep 5
          done
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log/app
    - name: log-sidecar
      image: busybox:1.36
      command: ["/bin/sh", "-c"]
      args:
        - "touch /var/log/app/app.log; tail -n+1 -F /var/log/app/app.log"
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log/app
  volumes:
    - name: shared-logs
      emptyDir: {}
```

### Step 1: Create the Pod

```bash
kubectl apply -f sidecar-pod.yaml
```

### Step 2: Check the Pod

```bash
kubectl get pod sidecar-demo
kubectl describe pod sidecar-demo
```

Expected: both containers should be running.

### Step 3: Check the main application's logs

```bash
kubectl logs sidecar-demo -c app
```

### Step 4: Check the sidecar's logs

```bash
kubectl logs sidecar-demo -c log-sidecar
```

The sidecar displays the log entries written by the main application because both containers mount the same volume.

### Step 5: Clean up

```bash
kubectl delete pod sidecar-demo
```

Note: `emptyDir` data is removed when the Pod is removed. This example demonstrates the sidecar pattern; it does not forward logs to an external logging platform.

## 5. Sidecar Container vs Init Container

| Feature | Sidecar Container | Init Container |
|---|---|---|
| Main purpose | Supports the application | Performs initialization |
| Execution | Commonly runs alongside the application | Runs before regular application containers start |
| Lifetime | Often runs for the application's lifetime | Normally completes before application startup |
| Example | Log collector | Database readiness check or configuration preparation |
| Restart behavior | Depends on container type and Pod configuration | Depends on its configuration and Pod restart policy |

### Important Kubernetes detail

Kubernetes supports native sidecars using restartable init containers. These are defined under `initContainers` with `restartPolicy: Always` and start before regular application containers. They can continue running alongside the application containers.

Therefore, not every sidecar must be defined under the regular `containers` list.

## 6. Sidecar Container vs Separate Pod

| Sidecar in the same Pod | Separate Pod |
|---|---|
| Shares the Pod IP and network namespace | Has its own Pod IP and network namespace |
| Can share Pod volumes | Requires an appropriate storage or network mechanism to share data |
| Lifecycle is closely coupled to the Pod | Can be managed independently |
| Useful for tightly coupled support functions | Useful for independently scalable services |

Real-time recommendation: A CoffeeShop frontend and Spring Boot backend should generally use separate Deployments and Pods because they need independent scaling and updates. A tightly coupled logging agent or proxy may be suitable as a sidecar.

## 7. Troubleshooting Sidecars

### Check all containers

```bash
kubectl get pod sidecar-demo
kubectl describe pod sidecar-demo
```

### Check a specific container's logs

```bash
kubectl logs sidecar-demo -c log-sidecar
```

### Check the container's previous logs after a restart

```bash
kubectl logs sidecar-demo -c log-sidecar --previous
```

### Check the shared volume configuration

```bash
kubectl get pod sidecar-demo -o yaml
```

Verify that both containers mount the same volume at the intended path.

Common issues:

- One container has an incorrect volume mount path.
- The main container writes to a different file.
- The sidecar exits because its command fails.
- The Pod has insufficient CPU or memory.
- Both containers attempt to listen on the same port.

## 8. OpenShift Commands

The same sidecar architecture works in OpenShift.

```bash
oc apply -f sidecar-pod.yaml
oc get pods
oc describe pod sidecar-demo
oc logs sidecar-demo -c app
oc logs sidecar-demo -c log-sidecar
oc delete pod sidecar-demo
```

OpenShift security policies and container image requirements may affect which images and commands are permitted.

## 9. Interview Questions and Answers

**Q1. What is a sidecar container?**
A supporting container that runs alongside the main application container in the same Pod.

**Q2. What are common sidecar use cases?**
Logging, monitoring, service-mesh proxies, configuration management, and supporting security functions.

**Q3. How do containers in a Pod communicate?**
They can communicate through localhost networking and exchange files through shared volumes.

**Q4. What happens if a regular sidecar container crashes?**
Its restart behavior depends on the Pod's `restartPolicy`. If the Pod is managed by a controller, the controller can also replace the Pod under the appropriate conditions.

**Q5. What is the difference between a sidecar and an init container?**
A conventional sidecar runs alongside the application, while a traditional init container completes its initialization task before regular application containers start. Native restartable sidecars are implemented as restartable init containers.

**Q6. Should the frontend and backend run as sidecars in the same Pod?**
Usually not. Separate Deployments allow independent scaling, updates, and recovery.

**Q7. Does a sidecar have its own Pod IP?**
No. A sidecar in the same Pod shares the Pod's network namespace and IP address.

## 10. Quick Revision

- Sidecar = supporting container alongside an application.
- Common uses: logging, monitoring, proxies, and configuration.
- Containers in the same Pod share networking.
- Shared files require a shared volume mounted into both containers.
- Conventional sidecars commonly run for the application's lifetime.
- Native sidecars use restartable init containers.
- Independently scalable services should generally run in separate Pods.
- Kubernetes and OpenShift both support sidecar patterns.
