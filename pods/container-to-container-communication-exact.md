# Kubernetes Notes: Container-to-Container Communication

## 1. What Is Container-to-Container Communication?

Container-to-container communication means how two containers exchange data or communicate with each other.

In Kubernetes, communication depends on whether the containers are running inside the same Pod or in different Pods.

There are two main scenarios:

1. Containers inside the same Pod.
2. Containers running in different Pods.

## 2. Communication Between Containers in the Same Pod

Containers inside the same Pod share the same network namespace and Pod IP address.

They can communicate using `localhost` or `127.0.0.1`, provided the destination application is listening on the requested port.

Example layout:

- **Kubernetes Pod:** Example Pod IP: 10.244.1.10
- **Container A:** Web application, Port 8080
- **Container B:** Helper service, Port 9000
- **localhost communication:** Container A can reach Container B at `localhost:9000`.

### Example

Suppose Container B runs an HTTP service on port 9000.

From Container A, send a request to:

```bash
curl http://localhost:9000
```

Because both containers share the Pod network namespace, the request reaches the service listening on port 9000 in Container B.

Important: Both containers must not attempt to bind to the same IP address and port combination. Otherwise, a port conflict can occur.

## 3. YAML Example: Two Containers in One Pod

Create a file named `same-pod.yaml`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: communication-pod
spec:
  containers:
    - name: web
      image: nginx:latest
      ports:
        - containerPort: 80
    - name: helper
      image: busybox:1.36
      command: ["/bin/sh", "-c"]
      args:
        - |
          while true; do
            wget -q -O- http://localhost:80
            sleep 10
          done
```

### Explanation

- `web` runs NGINX on port 80.
- `helper` sends HTTP requests to `localhost:80`.
- Both containers share the Pod network namespace.
- The helper can communicate with NGINX without a Kubernetes Service.

### Deploy and verify

```bash
kubectl apply -f same-pod.yaml
kubectl get pods
kubectl logs communication-pod -c helper
```

The helper's logs should contain NGINX HTML output if the requests succeed.

This example assumes the NGINX container starts successfully and its HTTP server is listening before the helper's requests run.

## 4. Communication Between Containers in Different Pods

Containers in different Pods normally communicate through the Kubernetes network using Pod IP addresses or, preferably, Kubernetes Services.

Example:

- Frontend Pod: 10.244.1.10
- Backend Pod: 10.244.2.20
- Backend port: 8080

The frontend can call the backend using its Service DNS name:

```bash
curl http://coffee-backend:8080
```

A Service provides a stable endpoint even when backend Pod IP addresses change.

Request flow:

1. **Frontend Pod** sends an HTTP request.
2. **Backend Service** — Stable DNS name and virtual IP.
3. **Backend Pod** — Application listening on port 8080.

## 5. Shared Volumes for Container Communication

Containers in the same Pod can exchange files using a shared volume.

Example:

- Container A writes application logs to `/shared/app.log`.
- Container B reads `/shared/app.log`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-volume-pod
spec:
  containers:
    - name: writer
      image: busybox:1.36
      command: ["/bin/sh", "-c"]
      args:
        - |
          while true; do
            date >> /shared/app.log
            sleep 5
          done
      volumeMounts:
        - name: shared-data
          mountPath: /shared
    - name: reader
      image: busybox:1.36
      command: ["/bin/sh", "-c"]
      args:
        - "touch /shared/app.log; tail -f /shared/app.log"
      volumeMounts:
        - name: shared-data
          mountPath: /shared
  volumes:
    - name: shared-data
      emptyDir: {}
```

Deploy and verify:

```bash
kubectl apply -f shared-volume-pod.yaml
kubectl logs shared-volume-pod -c reader
```

The reader displays the timestamps written by the writer.

Important: Shared volumes exchange files, not network messages. Use network communication when one container exposes an application endpoint.

## 6. Which Communication Method Should You Use?

| Scenario | Recommended method |
|---|---|
| Containers in the same Pod, one exposing an HTTP service | `localhost:PORT` |
| Containers in different Pods | Kubernetes Service DNS |
| Containers exchanging files in the same Pod | Shared volume |
| Frontend calling backend | Backend Service DNS |
| Service-to-service communication in OpenShift | Service DNS and cluster networking |

## 7. Troubleshooting Container Communication

### Check Pod status

```bash
kubectl get pods -o wide
kubectl describe pod communication-pod
```

### Check container logs

```bash
kubectl logs communication-pod -c web
kubectl logs communication-pod -c helper
```

### Execute a command inside a container

```bash
kubectl exec -it communication-pod -c helper -- /bin/sh
```

Then test:

```bash
wget -q -O- http://localhost:80
```

### Troubleshooting checklist

- Confirm the destination application is running and listening on the expected port.
- Verify the correct hostname and port.
- Check container logs and Pod events.
- Confirm Service selectors match the backend Pod labels.
- Check Service endpoints or EndpointSlices.
- Check NetworkPolicies if connectivity is blocked.
- Verify that the required debugging tools exist in the container image.

For a Service:

```bash
kubectl get svc
kubectl get endpoints
kubectl get endpointslices
```

For OpenShift, use the corresponding `oc` commands.

## 8. Interview Questions and Answers

**Q1. How do containers in the same Pod communicate?**
They can communicate over localhost because they share the Pod's network namespace. They can also exchange files through shared volumes.

**Q2. Do containers in the same Pod have separate IP addresses?**
Normally, no. They share the Pod IP address and network namespace.

**Q3. How do containers in different Pods communicate?**
They generally communicate over the cluster network, preferably through Kubernetes Services and DNS.

**Q4. Do containers in the same Pod need a Service to communicate?**
No. They can communicate directly using localhost when the destination service is listening on the correct port.

**Q5. Can two containers in the same Pod listen on port 8080?**
Not on the same Pod IP and protocol combination. Because they share the network namespace, this would normally cause a port conflict.

**Q6. What is the difference between localhost communication and Service communication?**
localhost addresses a service in the same Pod network namespace. A Service provides a stable endpoint for accessing a group of Pods or a particular application endpoint across the cluster.

## 9. Quick Revision

- Same Pod: use `localhost:PORT` for network communication.
- Different Pods: use a Kubernetes Service and DNS.
- Shared files: use a shared volume.
- Containers in the same Pod share a network namespace and Pod IP.
- Containers in different Pods normally have different Pod IP addresses.
- Kubernetes Services provide stable access when Pod IPs change.
- The same core networking concepts apply to Kubernetes and OpenShift.
