# Kubernetes Advanced Pod Topics – Graceful Application Shutdown

## 1. What Is Graceful Application Shutdown?

Graceful shutdown is the process of stopping an application in a controlled manner without unnecessarily interrupting active requests or corrupting application state.

When Kubernetes terminates a Pod, it normally gives the container a termination grace period to shut down before forcefully killing it.

Why is graceful shutdown required?

- Complete ongoing HTTP requests.
- Stop accepting new requests.
- Close database connections safely.
- Finish or safely stop background jobs.
- Flush application logs and telemetry.
- Release locks and other resources.
- Reduce errors during rolling updates and node maintenance.

Without graceful shutdown, users may experience HTTP 5xx errors, interrupted uploads, incomplete transactions, or dropped connections.

## 2. Kubernetes Pod Termination Workflow

When a Pod is deleted through the normal API deletion process, Kubernetes initiates termination.

A simplified workflow is:

1. The Pod is marked for deletion.
2. The Pod begins terminating, and its endpoint readiness and termination state are updated as appropriate.
3. The kubelet runs the container's PreStop hook if configured.
4. The container runtime sends the container's configured stop signal, normally SIGTERM.
5. The application performs its graceful shutdown.
6. The container exits.
7. If the grace period expires before it exits, Kubernetes attempts to forcefully terminate it, normally using SIGKILL.

```
             Pod Deletion
                  |
                  v
       Pod enters Terminating
                  |
                  v
      Endpoint / traffic updates
          occur concurrently
                  |
                  v
          PreStop hook
          if configured
                  |
                  v
             SIGTERM
                  |
                  v
      Application graceful stop
                  |
                  v
        Flush / cleanup / exit
                  |
            +-----+------+
            |            |
            v            v
       Exits in time   Grace period
            |          expires
            v            |
        Container        v
         stopped       SIGKILL
```

Important: Traffic removal and application shutdown are not a perfectly sequential process. Endpoint updates and shutdown can overlap, and existing connections may continue during the transition. Applications should be designed to handle this correctly.

## 3. What Is terminationGracePeriodSeconds?

`terminationGracePeriodSeconds` specifies how long Kubernetes allows a Pod to terminate gracefully before forceful termination is attempted.

The default is 30 seconds when not otherwise configured.

### YAML example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: graceful-demo
spec:
  terminationGracePeriodSeconds: 45
  containers:
    - name: app
      image: nginx:stable
```

Here, Kubernetes allows up to 45 seconds for termination.

Important: The grace period includes the execution time of the PreStop hook. It is not 45 seconds for the hook plus another 45 seconds for application shutdown.

If the hook consumes most of the grace period, the application may have very little time left to handle SIGTERM.

## 4. What Is SIGTERM?

SIGTERM is a termination signal that normally asks a process to stop.

An application can handle it and perform cleanup before exiting.

Examples:

- Node.js: handle SIGTERM and close the HTTP server.
- Java/Spring Boot: use JVM shutdown hooks and application lifecycle callbacks.
- Python: register a signal handler and stop workers.
- NGINX: use its graceful shutdown behavior through the appropriate signal.

SIGKILL cannot be caught or handled. If a process is forcibly killed, its own graceful cleanup code cannot run.

Interview point: Kubernetes does not automatically know how to complete application requests. The application must implement appropriate shutdown behavior.

## 5. What Is a PreStop Hook?

A PreStop lifecycle hook runs before the container's normal termination signal is sent.

It can be used to prepare an application for shutdown.

### Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: prestop-demo
spec:
  terminationGracePeriodSeconds: 40
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "trap 'echo SIGTERM received; exit 0' TERM; while true; do sleep 1; done"]
      lifecycle:
        preStop:
          exec:
            command:
              - sh
              - -c
              - echo "PreStop hook executed"
```

### What happens?

1. The Pod begins terminating.
2. The PreStop hook executes.
3. The hook completes.
4. Kubernetes sends the normal termination signal.
5. The container handles SIGTERM and exits.

### Important points

- The hook executes inside the container's context.
- The hook's execution time counts toward the termination grace period.
- A hook failure does not guarantee that termination will be cancelled.
- Use a hook for a meaningful shutdown operation, such as calling a local drain endpoint.
- Avoid unnecessary fixed sleeps; they consume the available shutdown budget.

## 6. Graceful Shutdown in Node.js

A Node.js HTTP server should stop accepting new connections and allow existing requests to finish before the process exits.

### Example application

```javascript
const http = require("http");

const server = http.createServer((req, res) => {
  if (req.url === "/health") {
    res.writeHead(200, { "Content-Type": "text/plain" });
    return res.end("OK");
  }

  setTimeout(() => {
    res.writeHead(200, { "Content-Type": "text/plain" });
    res.end("Request completed");
  }, 2000);
});

server.listen(8080, "0.0.0.0", () => {
  console.log("Server listening on port 8080");
});

let shuttingDown = false;

process.on("SIGTERM", () => {
  if (shuttingDown) return;
  shuttingDown = true;

  console.log("SIGTERM received; starting graceful shutdown");

  server.close((err) => {
    if (err) {
      console.error("HTTP server close failed", err);
      process.exitCode = 1;
    } else {
      console.log("HTTP server closed");
    }
  });
});
```

### How it works

- SIGTERM initiates shutdown.
- `server.close()` stops accepting new connections.
- Existing requests are allowed to finish.
- The process exits when the server and other active handles are closed.

A production application should also close database pools, message consumers, and background workers. It should enforce an overall shutdown deadline so that a stuck connection cannot keep the process alive indefinitely.

## 7. Graceful Shutdown in Java / Spring Boot

Spring Boot applications should stop accepting new work and close application resources cleanly.

Common mechanisms include:

- JVM shutdown hooks.
- Spring application lifecycle callbacks.
- Graceful web-server shutdown.
- Database connection pool shutdown.
- Message-consumer shutdown.

### Spring Boot configuration

In `application.properties`:

```properties
server.shutdown=graceful
spring.lifecycle.timeout-per-shutdown-phase=25s
```

This enables Spring Boot's graceful web-server shutdown and configures the timeout for each lifecycle shutdown phase.

### Kubernetes Pod configuration

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: spring-app
spec:
  terminationGracePeriodSeconds: 35
  containers:
    - name: app
      image: my-spring-app:1.0
      ports:
        - containerPort: 8080
```

The application must be built with a Spring Boot version that supports these properties and must correctly manage its own background tasks and external resources.

Important: The Spring lifecycle timeout is not the same as the Pod termination grace period. Configure the application shutdown budget to fit within the Kubernetes grace period, leaving time for hooks and other cleanup.

## 8. Readiness Probe and Graceful Shutdown

A readiness probe determines whether a Pod should receive normal Service traffic.

When an application begins shutting down, it should stop advertising readiness or otherwise coordinate traffic draining.

Example:

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  periodSeconds: 5
  timeoutSeconds: 2
  failureThreshold: 2
```

The application should return success only when it can accept new work.

### Important distinction

- Readiness failure removes the Pod from normal Service traffic when endpoint conditions update.
- Liveness failure can cause the container to restart.
- A readiness probe alone does not guarantee that every existing connection or external load balancer has stopped sending traffic.

Use the application's shutdown lifecycle and appropriate load-balancer or ingress draining behavior together.

## 9. preStop Hook vs SIGTERM

| Feature | PreStop hook | SIGTERM |
|---|---|---|
| Type | Lifecycle hook | Operating-system signal |
| Timing | Before normal stop signal | Sent after the hook completes |
| Purpose | Prepare for termination | Ask the process to terminate |
| Common use | Drain endpoint or application-specific preparation | Stop accepting work and clean up |
| Uses grace period | Yes | Yes |

A PreStop hook is optional. Many applications can shut down gracefully using their SIGTERM handler without a hook.

## 10. Graceful Shutdown During Rolling Updates

A Deployment can replace Pods during a rolling update.

### Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      terminationGracePeriodSeconds: 30
      containers:
        - name: app
          image: nginx:stable
          ports:
            - containerPort: 80
```

### What happens during an update?

1. The Deployment creates replacement capacity according to its rollout strategy.
2. New Pods start and become Ready.
3. Old Pods are selected for termination according to rollout progress.
4. Kubernetes initiates graceful termination of those Pods.
5. The application shuts down within its configured grace period if possible.

`maxSurge: 1` permits one extra Pod above the desired replica count during the rollout. `maxUnavailable: 0` prevents the Deployment rollout strategy from intentionally reducing the number of available replicas below the desired count.

These settings do not guarantee zero downtime by themselves. Application readiness, capacity, connection draining, and shutdown behavior still matter.

## 11. Common Graceful Shutdown Problems

### Problem 1: Requests fail during deployment

Possible causes:

- Application exits immediately on SIGTERM.
- Readiness changes too late.
- Load balancer or ingress still sends traffic.
- Existing requests exceed the shutdown grace period.

Investigate readiness handling, application logs, request duration, and traffic-draining behavior.

### Problem 2: Pod repeatedly reaches SIGKILL

Possible causes:

- Shutdown code hangs.
- Database pool never closes.
- Background threads remain active.
- PreStop consumes too much time.
- Grace period is too short.

Investigate shutdown logs and measure the time needed for the slowest legitimate request and cleanup operation.

### Problem 3: Pod remains Terminating

Possible causes:

- Process does not exit.
- Node is unreachable.
- Finalizer blocks deletion completion.
- Volume unmount or detach is delayed.

Commands:

```bash
kubectl describe pod <pod-name>
kubectl get pod <pod-name> -o yaml
kubectl get events --sort-by=.metadata.creationTimestamp
```

Inspect the deletion timestamp, finalizers, grace period, node condition, and storage events.

### Problem 4: Background jobs are interrupted

Possible causes:

- Worker does not stop accepting jobs on SIGTERM.
- In-progress work cannot finish within the grace period.
- Message acknowledgements or checkpoints are missing.

Implement a controlled worker shutdown: stop accepting new jobs, finish or safely checkpoint active work, release resources, and exit before the deadline.

## 12. Recommended Production Shutdown Sequence

A robust application shutdown usually follows this order:

1. Detect the termination signal.
2. Mark the application as not ready to receive new work.
3. Stop accepting new requests or jobs.
4. Allow in-flight work to finish within a defined deadline.
5. Close database connections, consumers, and other resources.
6. Flush logs and telemetry where feasible.
7. Exit before the Kubernetes grace period expires.

The exact order may vary with the application framework and load-balancing architecture. Ensure that traffic draining and endpoint updates have enough time to propagate.

### Practical design rules

- Set a realistic `terminationGracePeriodSeconds`.
- Use a PreStop hook only when necessary.
- Handle SIGTERM in the application.
- Keep shutdown operations bounded and observable.
- Configure readiness and load-balancer draining appropriately.
- Test rolling updates and node drains in a non-production environment.
- Make batch jobs and message processing safe to retry when interruption is possible.

## 13. Troubleshooting Commands

```bash
# Watch Pods during termination
kubectl get pods -w

# Inspect lifecycle hooks and grace period
kubectl get pod <pod-name> -o yaml

# Check application logs
kubectl logs <pod-name>

# Check previous container logs
kubectl logs <pod-name> --previous

# Inspect termination events
kubectl describe pod <pod-name>

# Observe Deployment rollout
kubectl rollout status deployment/<deployment-name>

# Inspect available endpoints
kubectl get endpointslices
```

Use the appropriate namespace with `-n <namespace>` when needed.

## 14. Interview Questions

**Q1. What is graceful application shutdown?**
It is the controlled stopping of an application so ongoing work can finish and resources can be released safely.

**Q2. What is terminationGracePeriodSeconds?**
It is the time Kubernetes allows for Pod termination before forceful termination is attempted. The default is 30 seconds.

**Q3. What is the purpose of SIGTERM?**
It requests that a process terminate and gives the application an opportunity to clean up.

**Q4. Can an application handle SIGKILL?**
No. SIGKILL cannot be caught or handled.

**Q5. What is a PreStop hook?**
A lifecycle hook that runs before the container's normal termination signal.

**Q6. Does the PreStop hook get additional time?**
No. Its execution time counts toward the Pod termination grace period.

**Q7. Does readiness failure stop all existing connections?**
No. It affects normal Service endpoint eligibility, but existing connections and external traffic paths may need separate draining.

**Q8. Why might a Pod get stuck in Terminating?**
Possible causes include slow shutdown, finalizers, node issues, and volume cleanup problems.

**Q9. How do you configure Spring Boot graceful shutdown?**
Use `server.shutdown=graceful` and configure `spring.lifecycle.timeout-per-shutdown-phase` appropriately.

**Q10. What is the relationship between PDB and graceful shutdown?**
A PDB constrains supported voluntary disruptions; graceful shutdown controls how an individual application handles termination. They solve different parts of the availability problem.

## 15. Quick Revision

| Topic | Key point |
|---|---|
| Graceful shutdown | Finish work and clean up before exiting |
| SIGTERM | Normal termination request |
| SIGKILL | Uncatchable forceful termination signal |
| PreStop | Runs before the normal stop signal |
| terminationGracePeriodSeconds | Total termination grace budget |
| Readiness probe | Controls normal Service traffic eligibility |
| Liveness probe | Can trigger container restarts |
| Rolling update | Replaces Pods according to rollout strategy |
| PDB | Constrains supported voluntary evictions |
| Application shutdown hook | Performs application-specific cleanup |

Key takeaway: Graceful shutdown depends on coordination between Kubernetes, the application, readiness handling, and traffic draining. Implement signal handling, set a realistic termination deadline, and test actual request behavior during rolling updates and node maintenance.
