# Kubernetes Notes: Init Containers

## 1. What Is an Init Container?

An Init Container is a specialized container that runs before the regular application containers in a Pod start.

It performs initialization tasks that must be completed before the application starts.

Example: In your CoffeeShop application, an init container can check whether the backend's required configuration file exists before the main application starts.

Init containers are defined under `spec.initContainers` in a Pod YAML file.

## 2. Why Do We Use Init Containers?

Common use cases include:

- Checking whether a required service is reachable.
- Preparing configuration files.
- Downloading initialization data.
- Setting up shared files or directories.
- Waiting for a prerequisite before application startup.
- Performing initialization tasks that should finish before the application starts.

Important: Init containers are not intended to run continuously as the main application does. Traditional init containers complete their tasks and exit successfully before regular application containers start.

## 3. Init Container Architecture

1. **Pod Created**
2. **Init Container** — Runs initialization tasks. Must complete successfully.
3. **Initialization Complete** — Next init container starts, if configured.
4. **Application Containers Start** — Main workload begins.

If an init container fails, Kubernetes normally retries it according to the Pod's restart policy. The regular application containers do not start until all traditional init containers have completed successfully.

## 4. Init Container YAML Example

Create a file named `init-pod.yaml`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  initContainers:
    - name: init-check
      image: busybox:1.36
      command:
        - /bin/sh
        - -c
        - "echo Initialization started; sleep 5; echo Initialization completed"
  containers:
    - name: nginx
      image: nginx:latest
      ports:
        - containerPort: 80
```

### Explanation

- `initContainers`: Defines containers that must initialize the Pod first.
- `init-check`: Name of the initialization container.
- `busybox:1.36`: Image used to run the initialization commands.
- `command`: Runs the initialization task.
- `containers`: Defines the regular application containers.
- `nginx`: The main application container.

In this example, the init container waits five seconds and exits successfully. Then Kubernetes starts the NGINX container.

## 5. Practical Lab

### Step 1: Create the Pod

```bash
kubectl apply -f init-pod.yaml
```

### Step 2: Watch the Pod

```bash
kubectl get pods -w
```

You may briefly see an initialization status such as:

```
NAME        READY   STATUS
init-demo   0/1     Init:0/1
```

After initialization finishes, the Pod should reach:

```
NAME        READY   STATUS
init-demo   1/1     Running
```

The exact intermediate output depends on how quickly the commands execute.

### Step 3: Check init container logs

```bash
kubectl logs init-demo -c init-check
```

Expected output:

```
Initialization started
Initialization completed
```

### Step 4: Inspect Pod details

```bash
kubectl describe pod init-demo
```

Look for the Init Containers, Containers, and Events sections.

### Step 5: Clean up

```bash
kubectl delete pod init-demo
```

## 6. Multiple Init Containers

A Pod can have multiple traditional init containers. They execute sequentially, in the order specified in the YAML.

The next init container starts only after the previous one completes successfully.

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-init-demo
spec:
  initContainers:
    - name: init-one
      image: busybox:1.36
      command: ["/bin/sh", "-c", "echo First initialization"]
    - name: init-two
      image: busybox:1.36
      command: ["/bin/sh", "-c", "echo Second initialization"]
  containers:
    - name: nginx
      image: nginx:latest
```

Execution order:

1. init-one runs and exits successfully.
2. init-two runs and exits successfully.
3. The NGINX application container starts.

## 7. Init Container for Dependency Checks

Suppose a backend application needs a database service to be reachable before startup.

An init container can wait for the database's DNS name and TCP port to become available.

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: backend-pod
spec:
  initContainers:
    - name: wait-for-db
      image: busybox:1.36
      command:
        - /bin/sh
        - -c
        - |
          until nc -z database-service 5432; do
            echo "Waiting for database..."
            sleep 2
          done
  containers:
    - name: backend
      image: nginx:latest
```

Important: This is an illustrative example. Replace `database-service` and port `5432` with the actual database Service name and port. The image must support the `nc` command. A successful TCP connection does not guarantee that the database is ready to accept application queries.

Also, if the database remains unavailable, this init container can keep the Pod in an initialization state. Ensure the dependency check matches your actual deployment requirements.

## 8. Init Containers vs Sidecar Containers

| Feature | Traditional Init Container | Sidecar Container |
|---|---|---|
| Main purpose | Initialization | Supporting the application |
| Execution | Before regular application containers | Commonly alongside the application |
| Completion | Normally exits successfully | Usually continues running |
| Ordering | Sequential among init containers | Depends on its configuration |
| Example | Wait for a dependency | Collect application logs |

Kubernetes version detail: Kubernetes also supports native sidecars, defined under `initContainers` with `restartPolicy: Always`. These continue running alongside application containers and are different from traditional init containers that complete and exit.

## 9. Troubleshooting Init Containers

### Scenario 1: Pod remains in Init state

```bash
kubectl get pods
kubectl describe pod init-demo
```

Check the Events section and init container status.

Possible causes:

- Image pull failure.
- Initialization command failure.
- Dependency is unavailable.
- Insufficient node resources.

### Scenario 2: Init container repeatedly fails

```bash
kubectl logs init-demo -c init-check
kubectl logs init-demo -c init-check --previous
kubectl describe pod init-demo
```

The `--previous` option can help when a prior init container instance has restarted and its logs remain available.

### Scenario 3: ImagePullBackOff

Check the image name, tag, registry credentials, and network access.

```bash
kubectl describe pod init-demo
```

### OpenShift commands

```bash
oc apply -f init-pod.yaml
oc get pods
oc describe pod init-demo
oc logs init-demo -c init-check
oc logs init-demo -c init-check --previous
oc delete pod init-demo
```

## 10. Interview Questions and Answers

**Q1. What is an init container?**
An init container performs setup tasks before the regular application containers in a Pod start.

**Q2. Can multiple init containers run in parallel?**
Traditional init containers run sequentially, in the order specified in the Pod specification.

**Q3. What happens if an init container fails?**
Kubernetes normally retries it according to the Pod's restart policy. Regular application containers wait until initialization succeeds.

**Q4. Can init containers share files with application containers?**
Yes. They can use a shared volume, provided the volume is mounted into both containers.

**Q5. What is the difference between an init container and a sidecar?**
A traditional init container completes its initialization task before application startup. A sidecar generally runs alongside the application to provide supporting functionality.

**Q6. Can an init container wait for a database?**
Yes. It can check DNS resolution or connectivity to a database Service before the application starts. The check must use the correct Service name and port.

**Q7. Does an init container run every time an application container restarts?**
Not necessarily. A traditional init container normally runs during Pod initialization. Restarting an application container within the same Pod does not ordinarily rerun successfully completed init containers. Creating a replacement Pod triggers initialization again.

## 11. Quick Revision

- Init containers are defined under `spec.initContainers`.
- Traditional init containers run before regular application containers.
- Multiple init containers execute sequentially.
- Each traditional init container must complete successfully before the next one starts.
- A failed init container can prevent the application from starting.
- Init containers can use shared volumes to prepare application data.
- Use `kubectl describe pod` and `kubectl logs -c` to troubleshoot.
- The same core init container behavior applies to Kubernetes and OpenShift.
