# Kubernetes and OpenShift Notes: Ports and Environment Variables

## 1. What are Ports in Kubernetes?

A port specifies the network port through which an application inside a container is expected to communicate.

Example: An Nginx web server normally listens on port 80 for HTTP traffic.

In Kubernetes and OpenShift, the container's application must actually listen on the configured port. Declaring a port in YAML does not start a listener or automatically expose the application.

Example:

```yaml
ports:
  - containerPort: 80
```

Here, `containerPort: 80` documents the port used by the application inside the container.

## 2. Types of Ports

**containerPort:**
The port on which the application inside the container is expected to listen.

**port:**
The port exposed by a Kubernetes Service.

**targetPort:**
The destination port on the selected Pod.

**nodePort:**
A port opened on cluster nodes by a NodePort Service, when configured.

Example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

Explanation:

- `port: 80` — Service port.
- `targetPort: 80` — Application destination port.
- `nodePort: 30080` — Node-level access port.

A NodePort Service requires matching Pods with the label `app: nginx`. The cluster networking configuration must also permit access.

## 3. What Are Environment Variables?

Environment variables provide configuration values to an application inside a container.

They help configure applications without changing the container image.

Examples:

- `APP_ENV=production`
- `PORT=8080`
- `DB_HOST=mysql-service`

Environment variables are commonly used to configure application behavior, service endpoints, and runtime settings.

## 4. Environment Variables in Pod YAML

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-demo
spec:
  containers:
    - name: nginx
      image: nginx:1.28
      env:
        - name: APP_ENV
          value: production
        - name: APP_PORT
          value: "8080"
```

Explanation:

- `env` defines environment variables.
- `name` specifies the variable name.
- `value` specifies the variable value.
- `APP_ENV` identifies the environment.
- `APP_PORT` provides a port value to the application if it is configured to use it.

Important: Setting `APP_PORT=8080` does not automatically change Nginx's listening port. The application must read and use that variable.

## 5. Using ConfigMap for Environment Variables

A ConfigMap stores non-sensitive configuration separately from the Pod YAML.

Create a ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: production
  APP_PORT: "8080"
```

Reference it in a Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-demo
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["/bin/sh", "-c"]
      args: ["echo $APP_ENV; echo $APP_PORT; sleep 3600"]
      envFrom:
        - configMapRef:
            name: app-config
```

The container receives `APP_ENV` and `APP_PORT` from the ConfigMap.

## 6. Using Secret for Sensitive Environment Variables

A Secret is used to store sensitive configuration, such as credentials.

Example reference:

```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: password
```

The Secret named `db-secret` must exist and contain the `password` key.

Do not hardcode real passwords in application YAML. Kubernetes Secrets also require appropriate access controls and encryption-at-rest configuration to protect sensitive data.

## 7. Complete Pod YAML Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-demo
  labels:
    app: nginx
spec:
  containers:
    - name: nginx
      image: nginx:1.28
      ports:
        - containerPort: 80
      env:
        - name: APP_ENV
          value: production
```

This defines a container port and an environment variable in the same Pod.

## 8. Kubernetes Commands

Create the Pod:

```bash
kubectl apply -f pod.yaml
```

Check the Pod:

```bash
kubectl get pods
```

Inspect the Pod configuration:

```bash
kubectl describe pod nginx-demo
```

Check environment variables inside the container:

```bash
kubectl exec nginx-demo -- printenv APP_ENV
```

Check the Pod logs:

```bash
kubectl logs nginx-demo
```

## 9. OpenShift Commands

Create the Pod:

```bash
oc apply -f pod.yaml
```

Check the Pod:

```bash
oc get pods
```

Inspect the configuration:

```bash
oc describe pod nginx-demo
```

Check an environment variable:

```bash
oc exec nginx-demo -- printenv APP_ENV
```

Check the logs:

```bash
oc logs nginx-demo
```

These fields and commands work similarly in Kubernetes and OpenShift, subject to cluster policies.

## 10. Difference Between Ports and Environment Variables

Ports define network communication endpoints.

Environment variables provide configuration values to applications.

Example:

- `containerPort: 80` — Documents the expected application port.
- `APP_ENV=production` — Configures the application's environment.
- Service `port: 80` — Provides a stable service endpoint.
- `NodePort: 30080` — Provides node-level access through a NodePort Service.

## 11. Interview Questions

**Q1. What is containerPort?**
It documents the port on which the container application is expected to listen.

**Q2. Does containerPort expose the application outside the cluster?**
No. A Service or another suitable networking configuration is needed.

**Q3. What are environment variables?**
They provide runtime configuration values to applications inside containers.

**Q4. What is the difference between port and targetPort?**
`port` is the Service port, while `targetPort` is the destination port on the selected Pod.

**Q5. What is a ConfigMap?**
A ConfigMap stores non-sensitive configuration data separately from application code and Pod definitions.

**Q6. What is a Secret?**
A Secret stores sensitive configuration data and can provide it to containers.

**Q7. Are ports and environment variables different in OpenShift?**
No. The basic Kubernetes fields are the same in both platforms.

## 12. Quick Revision

- containerPort = Expected application port inside the container.
- port = Service port.
- targetPort = Destination port on the Pod.
- nodePort = Node-level Service port.
- env = Defines environment variables.
- ConfigMap = Stores non-sensitive configuration.
- Secret = Stores sensitive configuration.

Remember: Declaring a port does not expose the application, and defining an environment variable does not automatically change application behavior.
