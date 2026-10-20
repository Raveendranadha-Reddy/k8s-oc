# Kubernetes Notes: terminationGracePeriodSeconds

## 1. What Is terminationGracePeriodSeconds?

terminationGracePeriodSeconds is a Pod specification field that defines how many seconds Kubernetes allows a Pod's containers to terminate gracefully before forceful termination.

Default value: 30 seconds

During this period, the application should finish ongoing requests, close connections, release resources, and exit cleanly.

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  terminationGracePeriodSeconds: 60
  containers:
    - name: nginx
      image: nginx:latest
```

Here, Kubernetes allows up to 60 seconds for graceful termination under normal conditions.

## 2. How Does It Work?

1. **Pod deletion requested** — Example: `kubectl delete pod nginx-pod`.
2. **Graceful termination begins** — The grace-period countdown starts.
3. **PreStop hook runs, if configured** — Its execution time counts toward the grace period.
4. **SIGTERM is sent** — The application is asked to exit gracefully.
5. **Grace period expires** — Any container still running is typically forcefully terminated using SIGKILL.

The PreStop hook and container shutdown share the same grace-period budget. Kubernetes may allow a small one-off extension if the hook is still running, but you should not rely on it.

## 3. Different Values and Their Use Cases

| Value | Meaning / Use case |
|---|---|
| 0 | No normal grace period; termination can be immediate. |
| 10 | Short-lived or simple applications with fast shutdown. |
| 30 | Default value; suitable for many applications. |
| 60 | Applications that need more time to drain requests or close connections. |
| 120 | Applications with longer shutdown requirements, if justified. |

Choose the value based on the application's actual shutdown time. A larger value does not itself make the application shut down gracefully.

## 4. Practical Lab

### Step 1: Create a Pod

Create `graceful-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: graceful-pod
spec:
  terminationGracePeriodSeconds: 30
  containers:
    - name: nginx
      image: nginx:latest
      lifecycle:
        preStop:
          exec:
            command: ["/bin/sh", "-c", "sleep 10"]
```

### Step 2: Create the Pod

```bash
kubectl apply -f graceful-pod.yaml
```

### Step 3: Verify the configuration

```bash
kubectl get pod graceful-pod -o yaml
kubectl describe pod graceful-pod
```

### Step 4: Delete the Pod

```bash
kubectl delete pod graceful-pod
```

The PreStop hook waits approximately 10 seconds before normal termination signalling proceeds. The remaining grace-period time is available for the container to exit.

## 5. Configure It in a Deployment

In production, this field is commonly configured in the Pod template of a Deployment.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: coffee-frontend
  namespace: coffeeshop
spec:
  replicas: 2
  selector:
    matchLabels:
      app: coffee-frontend
  template:
    metadata:
      labels:
        app: coffee-frontend
    spec:
      terminationGracePeriodSeconds: 60
      containers:
        - name: coffee-frontend
          image: raveendradudyala/coffee-shop-frontend:5.0.0
          ports:
            - containerPort: 80
```

Apply and verify:

```bash
kubectl apply -f coffee-frontend.yaml -n coffeeshop
kubectl rollout status deployment/coffee-frontend -n coffeeshop
kubectl get pods -n coffeeshop
```

Important: Changing this field in a Deployment's Pod template affects newly created Pods and Pods replaced during a rollout. It does not retroactively change the grace period of an existing Pod already terminating.

## 6. Common Mistakes

- **Setting a long grace period without shutdown handling:** The application may still be forcefully terminated when the timer expires.
- **Using sleep as a complete shutdown solution:** A delay alone does not drain requests or close connections.
- **Forgetting the PreStop duration:** The hook consumes part of the configured grace period.
- **Confusing container restart with Pod termination:** This setting primarily governs graceful Pod termination; it is not a general delay for every container restart.
- **Using force deletion unnecessarily:** Force deletion can bypass normal graceful shutdown behavior.

## 7. Interview Questions

**Q1. What is the default value of terminationGracePeriodSeconds?**
30 seconds.

**Q2. When does the grace-period countdown start?**
When Pod termination begins after a deletion request is processed.

**Q3. Does the PreStop hook get extra time beyond the configured grace period?**
Normally, no. Its execution time counts toward the grace period, although Kubernetes may grant a small one-off extension in certain circumstances.

**Q4. What happens if a container does not exit before the grace period expires?**
Kubernetes typically forcefully terminates the container using SIGKILL.

**Q5. Does this field apply to Kubernetes and OpenShift?**
Yes. OpenShift uses the same Kubernetes Pod termination mechanism.

## 8. Quick Revision

- Field: `spec.terminationGracePeriodSeconds`
- Default: 30
- Purpose: Allow time for graceful shutdown.
- PreStop hook runs before normal termination signalling.
- SIGTERM requests graceful termination.
- SIGKILL forcefully terminates a process that has not exited in time.
- Configure the value based on the application's real shutdown requirements.
