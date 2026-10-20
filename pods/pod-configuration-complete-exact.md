# Kubernetes Pod Configuration – Complete Notes

## 1. What Is Pod Configuration?

Pod Configuration defines how containers inside a Pod should run.

It includes:

- Container image and image pull policy
- Commands and arguments
- Environment variables
- ConfigMaps and Secrets
- Ports
- Resource requests and limits
- Security context
- Service account
- DNS settings
- Restart policy
- Volume mounts
- Scheduling preferences

Pod configuration is generally declared in a YAML manifest and submitted to the Kubernetes API server.

## 2. Basic Pod YAML Structure

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-demo
  namespace: default
  labels:
    app: config-demo
spec:
  restartPolicy: Always
  containers:
    - name: app
      image: nginx:stable
      ports:
        - containerPort: 80
```

### Field explanation

| Field | Purpose |
|---|---|
| apiVersion | Kubernetes API version |
| kind | Type of resource |
| metadata.name | Pod name |
| metadata.namespace | Namespace containing the Pod |
| metadata.labels | Key-value metadata used for selection and organization |
| spec | Desired Pod configuration |
| containers | Containers to run inside the Pod |
| image | Container image |
| containerPort | Documents a port the container listens on; it does not expose the Pod externally |
| restartPolicy | Controls container restart behavior within the Pod |

## 3. Environment Variables

Environment variables pass configuration values into a container.

### Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-demo
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "env; sleep 3600"]
      env:
        - name: APP_ENV
          value: "development"
        - name: APP_PORT
          value: "8080"
```

### Deploy and verify

```bash
kubectl apply -f env-demo.yaml
kubectl exec env-demo -- printenv APP_ENV
kubectl exec env-demo -- printenv APP_PORT
```

Expected values:

```
development
8080
```

Important: Environment variables are strings. Do not put passwords or tokens directly in a manifest committed to Git.

## 4. ConfigMap Configuration

A ConfigMap stores non-sensitive configuration separately from the container image.

### Create a ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: "production"
  APP_PORT: "8080"
```

### Use ConfigMap values as environment variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-demo
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "env; sleep 3600"]
      envFrom:
        - configMapRef:
            name: app-config
```

Deploy and verify:

```bash
kubectl apply -f configmap.yaml
kubectl apply -f configmap-pod.yaml
kubectl exec configmap-demo -- printenv APP_ENV
kubectl exec configmap-demo -- printenv APP_PORT
```

You can also reference a single ConfigMap key:

```yaml
env:
  - name: APP_ENV
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: APP_ENV
```

Important: Environment variables populated from a ConfigMap do not update automatically in a running container when the ConfigMap changes. The Pod or application must be restarted or reconfigured to pick up the new value.

## 5. Secret Configuration

A Secret stores sensitive configuration, such as passwords, tokens, or credentials.

### Create a Secret

```bash
kubectl create secret generic db-secret \
  --from-literal=DB_USERNAME=appuser \
  --from-literal=DB_PASSWORD='change-me'
```

### Inject Secret values into a Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-demo
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      env:
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: DB_USERNAME
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: DB_PASSWORD
```

Verify without displaying the password:

```bash
kubectl exec secret-demo -- printenv DB_USERNAME
kubectl get secret db-secret
```

Security note: Kubernetes Secret data is base64-encoded, not automatically encrypted by virtue of being a Secret. Use RBAC, encryption at rest, and secure secret-management practices.

## 6. Command and Args

Kubernetes supports `command` and `args` to control how a container process starts.

| Field | Meaning |
|---|---|
| command | Overrides the image's default ENTRYPOINT |
| args | Overrides the image's default CMD arguments |

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: command-demo
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c"]
      args:
        - echo "Application started"; sleep 3600
```

Verify:

```bash
kubectl apply -f command-demo.yaml
kubectl logs command-demo
kubectl describe pod command-demo
```

Interview point: `command` is not simply a shell command string. It replaces the image entrypoint. If you need shell operators such as `&&`, pipes, or variable expansion, invoke a shell explicitly.

## 7. Image Pull Policy

The `imagePullPolicy` controls when Kubernetes attempts to pull the container image.

```yaml
containers:
  - name: app
    image: nginx:stable
    imagePullPolicy: IfNotPresent
```

| Policy | Behavior |
|---|---|
| Always | Kubelet asks the container runtime to resolve the image reference on each container launch |
| IfNotPresent | Pulls only when the image is not already present locally |
| Never | Never pulls; the image must already be available locally |

Kubernetes sets a default when the Pod is initially created. That default is not automatically recalculated if you later change the image tag.

For production, prefer controlled image versions or digests rather than relying on a mutable `latest` tag.

## 8. Container Ports

Ports document the ports the container listens on.

```yaml
ports:
  - name: http
    containerPort: 8080
    protocol: TCP
```

`containerPort` does not create a Service, expose the application to the internet, or automatically open a firewall.

For network access, configure a Service and, where required, an Ingress.

## 9. Resource Requests and Limits

Resource configuration controls the CPU and memory resources requested by a container.

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

- `250m` CPU means 0.25 CPU core.
- Requests influence scheduling.
- CPU limits can cause throttling.
- Exceeding a memory limit can result in an OOM kill.

Inspect resources:

```bash
kubectl describe pod config-demo
kubectl top pod config-demo
kubectl describe node <node-name>
```

`kubectl top` requires metrics support.

## 10. Security Context

A security context configures security-related properties for a Pod or container.

### Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-demo
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 10001
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "id; sleep 3600"]
      securityContext:
        allowPrivilegeEscalation: false
        capabilities:
          drop:
            - ALL
```

This example requires an image that can run as the specified non-root UID.

Inspect the result:

```bash
kubectl apply -f security-demo.yaml
kubectl exec security-demo -- id
kubectl describe pod security-demo
```

Security settings can be constrained by cluster policies, which may restrict user IDs, privileges, and volume access.

## 11. Service Account

A service account provides an identity for a Pod when it accesses the Kubernetes API.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: serviceaccount-demo
spec:
  serviceAccountName: app-service-account
  automountServiceAccountToken: false
  containers:
    - name: app
      image: nginx:stable
```

Create the service account first:

```bash
kubectl create serviceaccount app-service-account
```

Setting `automountServiceAccountToken: false` disables automatic token mounting for the Pod. Use it when the application does not need Kubernetes API access.

For workloads that need API access, grant only the required permissions through RBAC.

## 12. Restart Policy

A Pod's `restartPolicy` controls whether failed containers are restarted.

Supported values:

- Always
- OnFailure
- Never

Example:

```yaml
spec:
  restartPolicy: OnFailure
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "exit 1"]
```

For Pods managed by a Deployment, the restart policy must be `Always`.

A Deployment is normally preferred over a standalone Pod for long-running applications because it maintains the desired replica count.

## 13. DNS Policy

DNS configuration controls how a Pod resolves hostnames.

Common policies:

| Policy | Meaning |
|---|---|
| ClusterFirst | Uses cluster DNS for names that are not resolved through the cluster's DNS setup |
| Default | Inherits DNS configuration from the node |
| None | Uses explicitly configured DNS settings |

Example:

```yaml
spec:
  dnsPolicy: ClusterFirst
  containers:
    - name: app
      image: nginx:stable
```

For custom DNS configuration, `dnsConfig` can specify nameservers, search domains, and resolver options.

Avoid changing DNS policy without understanding the cluster's DNS architecture.

## 14. Pod-Level Configuration vs Container-Level Configuration

| Pod-level settings | Container-level settings |
|---|---|
| nodeSelector | image |
| affinity | command and args |
| tolerations | env |
| volumes | volumeMounts |
| serviceAccountName | ports |
| restartPolicy | resources |
| dnsPolicy | Container securityContext |

Some security and resource settings can be defined at either the Pod or container level, depending on the field.

## 15. Complete Pod Configuration Example

The following manifest combines environment variables, ConfigMap, Secret, resources, security settings, and a volume.

Create the ConfigMap and Secret first:

```bash
kubectl create configmap web-config \
  --from-literal=APP_ENV=production

kubectl create secret generic web-secret \
  --from-literal=API_TOKEN='replace-with-a-real-token'
```

Save the Pod as `complete-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: complete-config-pod
  labels:
    app: complete-config
spec:
  restartPolicy: Always
  automountServiceAccountToken: false
  securityContext:
    runAsNonRoot: true
    runAsUser: 10001
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      image: nginx:stable
      imagePullPolicy: IfNotPresent
      ports:
        - name: http
          containerPort: 80
      env:
        - name: APP_ENV
          valueFrom:
            configMapKeyRef:
              name: web-config
              key: APP_ENV
        - name: API_TOKEN
          valueFrom:
            secretKeyRef:
              name: web-secret
              key: API_TOKEN
      resources:
        requests:
          cpu: "100m"
          memory: "128Mi"
        limits:
          cpu: "500m"
          memory: "256Mi"
      volumeMounts:
        - name: temp-data
          mountPath: /tmp/app
  volumes:
    - name: temp-data
      emptyDir: {}
```

Lab note: This manifest demonstrates configuration fields, but it may not run unchanged in every environment. Verify that the image supports the configured non-root UID, that the selected image and security policy permit execution, and that the application actually uses the supplied variables. The stock NGINX image does not automatically use `APP_ENV` or `API_TOKEN` to configure application behavior.

Deploy and inspect:

```bash
kubectl apply -f complete-pod.yaml
kubectl get pod complete-config-pod
kubectl describe pod complete-config-pod
kubectl get pod complete-config-pod -o yaml
```

Clean up after the lab:

```bash
kubectl delete pod complete-config-pod
kubectl delete configmap web-config
kubectl delete secret web-secret
```

## 16. Common Troubleshooting Scenarios

### Pod stuck in CreateContainerConfigError

Possible causes:

- Missing ConfigMap
- Missing Secret
- Missing referenced key

Commands:

```bash
kubectl describe pod <pod-name>
kubectl get configmap
kubectl get secret
```

### Pod stuck in ImagePullBackOff

Possible causes:

- Incorrect image name or tag
- Registry authentication failure
- Network or registry availability issues

Commands:

```bash
kubectl describe pod <pod-name>
kubectl get events --sort-by=.metadata.creationTimestamp
```

### Container exits repeatedly

Check the command, arguments, logs, and exit code:

```bash
kubectl logs <pod-name>
kubectl logs <pod-name> --previous
kubectl describe pod <pod-name>
```

### Permission denied

Check the effective user, filesystem ownership, security context, and platform policies:

```bash
kubectl exec <pod-name> -- id
kubectl describe pod <pod-name>
```

### Configuration change not reflected

If environment variables came from a ConfigMap or Secret, changing the source does not update the environment of an already running process. Restart the Pod or use an application-specific reload mechanism.

For volume-mounted configuration, update behavior depends on the volume type and mount method. `subPath` mounts do not receive normal automatic updates.

## 17. Kubernetes and OpenShift Commands

| Task | Kubernetes | OpenShift |
|---|---|---|
| Create resources | `kubectl apply -f file.yaml` | `oc apply -f file.yaml` |
| Inspect Pod | `kubectl describe pod <name>` | `oc describe pod <name>` |
| Read logs | `kubectl logs <name>` | `oc logs <name>` |
| Execute command | `kubectl exec <name> -- <cmd>` | `oc exec <name> -- <cmd>` |
| View events | `kubectl get events` | `oc get events` |
| View YAML | `kubectl get pod <name> -o yaml` | `oc get pod <name> -o yaml` |

## 18. Interview Questions

**Q1. What is Pod Configuration?**
It defines how containers run, including images, environment variables, resources, security, volumes, and runtime settings.

**Q2. What is the difference between ConfigMap and Secret?**
ConfigMaps store non-sensitive configuration. Secrets store sensitive values, but require appropriate access controls and encryption practices.

**Q3. What is the difference between command and args?**
`command` overrides the image ENTRYPOINT, while `args` overrides the image CMD arguments.

**Q4. What is imagePullPolicy?**
It determines whether the container runtime should attempt to pull the image when launching a container.

**Q5. Does containerPort expose a Pod externally?**
No. A Service or another networking mechanism is needed for access.

**Q6. What is a security context?**
It defines security-related settings such as user identity, privilege escalation, capabilities, and seccomp configuration.

**Q7. Why use automountServiceAccountToken: false?**
To prevent automatic mounting of a service account token when the workload does not need Kubernetes API access.

**Q8. What happens if a ConfigMap referenced by a Pod does not exist?**
The Pod may fail during container configuration or volume setup, depending on how the ConfigMap is referenced.

**Q9. Can environment variables update automatically after a ConfigMap change?**
No. Existing container environments do not automatically change. A restart or application-specific configuration reload is needed.

**Q10. What is the default restart policy for a Pod?**
Always, unless specified otherwise.

**Q11. What is the difference between Pod-level and container-level security contexts?**
Pod-level settings apply to the Pod's containers where supported; container-level settings configure an individual container and can override applicable Pod-level defaults.

**Q12. Why might a Pod fail after setting runAsNonRoot: true?**
The image may require root privileges, or its configured user identity may not satisfy the non-root requirement.

## 19. Quick Revision

| Configuration | Purpose |
|---|---|
| env | Set environment variables |
| envFrom | Import values from a ConfigMap or Secret |
| command | Override image ENTRYPOINT |
| args | Override image CMD arguments |
| imagePullPolicy | Control image pull behavior |
| resources | Define CPU and memory requests and limits |
| securityContext | Configure container and Pod security |
| serviceAccountName | Select the Pod's service account |
| restartPolicy | Define container restart behavior |
| dnsPolicy | Control Pod DNS behavior |
| volumeMounts | Mount volumes into containers |
| containerPort | Document a container's listening port |

Key takeaway: Keep application configuration separate from container images, use Secrets and least-privilege access for sensitive values, declare realistic resource requests, and validate security settings against the cluster's policies.
