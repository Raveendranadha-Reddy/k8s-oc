# Kubernetes Notes: Pod Health Checks

The key concept is that Kubernetes uses probes to determine whether a container has started, is healthy, and is ready to receive traffic. Pod health checks are especially important for production deployments, rolling updates, and troubleshooting applications such as your CoffeeShop frontend and Spring Boot backend.

## 1. What Are Pod Health Checks?

Pod health checks are mechanisms Kubernetes uses to check the state of an application running inside a container.

Kubernetes provides three main probe types:

1. Startup Probe — checks whether the application has started.
2. Readiness Probe — checks whether the application is ready to receive traffic.
3. Liveness Probe — checks whether the application is still healthy enough to continue running.

These probes are configured inside the container specification.

Important: Probes are configured for containers, not directly for the Pod as a whole.

## 2. Why Are Health Checks Required?

Health checks help Kubernetes:

- Detect applications that fail during startup.
- Avoid sending normal Service traffic to unready containers.
- Restart containers that become unresponsive.
- Support safer Deployment rollouts.
- Improve application availability.

Example: In your CoffeeShop project, the frontend may be running while the Spring Boot backend is still initializing. A readiness probe prevents the backend from receiving normal Service traffic until it is ready.

## 3. Three Types of Probes

- **Startup Probe** — Has the application started? Protects slow-starting applications from premature liveness and readiness checks until startup succeeds.
- **Readiness Probe** — Can it receive traffic? When readiness fails, the Pod is marked unready and normally excluded from Service traffic.
- **Liveness Probe** — Should the container restart? Repeated liveness failures cause kubelet to restart the affected container.

## 4. Startup Probe

### Definition

A startup probe determines whether an application has completed its startup process.

It is useful for applications that take a long time to initialize, such as Java Spring Boot applications.

### Example YAML

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: startup-demo
spec:
  containers:
    - name: app
      image: nginx:latest
      startupProbe:
        httpGet:
          path: /
          port: 80
        periodSeconds: 5
        failureThreshold: 12
```

In this example, Kubernetes checks every five seconds. After 12 consecutive failures, the startup probe fails and the container is restarted.

The approximate startup failure window is 60 seconds, subject to probe timing and execution overhead.

Important: While a startup probe is configured and has not succeeded, Kubernetes does not run the container's liveness and readiness probes.

## 5. Readiness Probe

### Definition

A readiness probe determines whether a container is ready to receive normal network traffic.

If the readiness probe fails:

- The container is not automatically restarted just because of that failure.
- The Pod is marked unready.
- Normal Service traffic is directed away from that unready Pod.

### Example YAML

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: readiness-demo
spec:
  containers:
    - name: app
      image: nginx:latest
      ports:
        - containerPort: 80
      readinessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 10
        timeoutSeconds: 2
        failureThreshold: 3
```

Explanation:

- `httpGet` — sends an HTTP request.
- `path: /` — checks the root path.
- `port: 80` — checks port 80.
- `initialDelaySeconds: 5` — delays the first check for five seconds.
- `periodSeconds: 10` — performs checks every ten seconds.
- `timeoutSeconds: 2` — allows two seconds for each probe.
- `failureThreshold: 3` — marks the container unready after three consecutive failures.

## 6. Liveness Probe

### Definition

A liveness probe determines whether a container should continue running or be restarted.

If the liveness probe fails repeatedly, kubelet restarts the affected container according to the Pod's restart policy.

### Example YAML

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-demo
spec:
  containers:
    - name: app
      image: nginx:latest
      ports:
        - containerPort: 80
      livenessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 10
        periodSeconds: 10
        timeoutSeconds: 2
        failureThreshold: 3
```

In this example, Kubernetes checks whether NGINX responds to HTTP requests. Three consecutive failures trigger a container restart.

Production tip: Do not make liveness depend unnecessarily on external services such as a database. A temporary database outage could cause repeated application restarts. Use readiness checks for dependencies that determine whether an application can serve traffic.

## 7. Probe Mechanisms

Kubernetes supports several probe mechanisms.

| Mechanism | How it works | Example |
|---|---|---|
| HTTP GET | Sends an HTTP request to a path and port | Web application health endpoint |
| TCP Socket | Attempts to establish a TCP connection | Database or application port |
| Exec | Executes a command inside the container | Check a local process or file |
| gRPC | Calls a gRPC health-check endpoint | gRPC microservice |

### HTTP probe

```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 8080
```

### TCP probe

```yaml
livenessProbe:
  tcpSocket:
    port: 8080
  periodSeconds: 10
```

### Exec probe

```yaml
readinessProbe:
  exec:
    command:
      - cat
      - /tmp/healthy
```

The exec probe succeeds if the command exits with status code 0.

### gRPC probe

```yaml
livenessProbe:
  grpc:
    port: 50051
  periodSeconds: 10
```

The application must implement the appropriate gRPC health-check protocol for the probe to work correctly.

## 8. Important Probe Parameters

| Parameter | Purpose |
|---|---|
| initialDelaySeconds | Delay before the first probe, except where startup-probe behavior governs the startup sequence |
| periodSeconds | Interval between probes |
| timeoutSeconds | Maximum time allowed for one probe |
| failureThreshold | Consecutive failures required to consider the probe failed |
| successThreshold | Consecutive successes required to recover from a failed probe; for liveness and startup probes, it must be 1 |
| terminationGracePeriodSeconds | Grace period used when terminating a container because of a liveness or startup probe failure, if configured at probe level; otherwise the Pod-level setting applies |

Probe timing is approximate because execution and scheduling add overhead. Avoid treating the parameters as an exact timer.

## 9. Combined Example: All Three Probes

This example uses NGINX and checks its HTTP endpoint. For production applications, use endpoints designed for the actual startup, readiness, and liveness semantics.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: health-check-demo
spec:
  containers:
    - name: nginx
      image: nginx:latest
      ports:
        - containerPort: 80
      startupProbe:
        httpGet:
          path: /
          port: 80
        periodSeconds: 5
        failureThreshold: 12
      readinessProbe:
        httpGet:
          path: /
          port: 80
        periodSeconds: 10
        timeoutSeconds: 2
        failureThreshold: 3
      livenessProbe:
        httpGet:
          path: /
          port: 80
        periodSeconds: 10
        timeoutSeconds: 2
        failureThreshold: 3
```

Apply the manifest:

```bash
kubectl apply -f health-check-demo.yaml
kubectl get pods -w
kubectl describe pod health-check-demo
```

## 10. Real-Time Example: CoffeeShop Spring Boot Backend

Suppose your backend exposes these application health endpoints:

- `/actuator/health/liveness`
- `/actuator/health/readiness`

These endpoints must be enabled and configured correctly in Spring Boot Actuator before using them.

Example container configuration:

```yaml
containers:
  - name: coffee-backend
    image: raveendradudyala/coffee-shop-backend:1.0.0
    ports:
      - containerPort: 8080
    startupProbe:
      httpGet:
        path: /actuator/health/liveness
        port: 8080
      periodSeconds: 5
      failureThreshold: 30
    readinessProbe:
      httpGet:
        path: /actuator/health/readiness
        port: 8080
      periodSeconds: 10
      timeoutSeconds: 2
      failureThreshold: 3
    livenessProbe:
      httpGet:
        path: /actuator/health/liveness
        port: 8080
      periodSeconds: 10
      timeoutSeconds: 2
      failureThreshold: 3
```

Here, the startup probe allows approximately 150 seconds of failed startup checks before the startup probe fails, subject to execution overhead.

Before using this configuration, confirm that the backend exposes the endpoints, returns successful HTTP status codes when healthy, and uses appropriate health semantics. Do not expose sensitive management endpoints publicly.

## 11. Troubleshooting Health Checks

### Scenario 1: Pod is Running but not Ready

```bash
kubectl get pods
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

Possible causes:

- Application is still initializing.
- Readiness endpoint returns a non-success status.
- Wrong port or path.
- A required dependency is unavailable.

### Scenario 2: Container repeatedly restarts

```bash
kubectl describe pod <pod-name>
kubectl logs <pod-name> --previous
```

Possible causes:

- Liveness probe is failing.
- Application is crashing.
- Probe timeout is too short.
- The application needs more startup time.

### Scenario 3: Startup probe fails

Check the startup duration, application logs, probe path, port, and timeout. Increase the startup window only when the application genuinely needs more time.

### OpenShift commands

```bash
oc get pods
oc describe pod <pod-name>
oc logs <pod-name>
oc logs <pod-name> --previous
oc get events --sort-by=.metadata.creationTimestamp
```

The same probe behavior applies in OpenShift.

## 12. Interview Questions and Answers

**Q1. What are the three Kubernetes probes?**
Startup, readiness, and liveness probes.

**Q2. What is the difference between readiness and liveness?**
Readiness determines whether the Pod should receive normal Service traffic. Liveness determines whether the container should be restarted after repeated failures.

**Q3. What is the purpose of a startup probe?**
It gives a slow-starting application time to initialize before liveness and readiness checks begin.

**Q4. Does a failed readiness probe restart the container?**
No. Readiness failure alone marks the Pod unready and normally removes it from Service traffic.

**Q5. What happens when a liveness probe fails repeatedly?**
Kubelet restarts the affected container, subject to the Pod's restart policy and termination settings.

**Q6. Can all three probes be configured together?**
Yes. A startup probe can protect slow startup, while readiness and liveness probes check traffic readiness and continued health.

**Q7. What is the difference between a startup probe and initialDelaySeconds?**
A startup probe checks actual application startup and suppresses liveness and readiness probes until it succeeds. `initialDelaySeconds` simply delays when a particular probe begins.

## 13. Quick Revision

- Startup: Has the application started?
- Readiness: Can it receive traffic?
- Liveness: Should the container be restarted?
- Probes are configured inside the container specification.
- Supported mechanisms include HTTP, TCP, exec, and gRPC.
- Readiness failure does not itself restart the container.
- Liveness and startup probe failures can trigger a restart.
- Use suitable health endpoints and avoid overly aggressive probes.
- Kubernetes and OpenShift share the same core probe behavior.

For your CoffeeShop backend, the most important production practice is to configure startup, readiness, and liveness endpoints according to Spring Boot's actual health behavior—not just whether port 8080 is open.

Your earlier CoffeeShop project uses a Spring Boot backend, so the Actuator endpoint example is relevant to that setup.
