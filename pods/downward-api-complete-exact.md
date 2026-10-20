# Kubernetes Downward API – Complete Notes

## 1. What Is the Downward API?

The Downward API allows a container to access information about its own Pod and container configuration without needing to query the Kubernetes API server.

It can expose selected Pod metadata and resource information to applications through:

1. Environment variables
2. Volume files

For example, an application can read its Pod name, namespace, Pod IP address, labels, annotations, or configured resource requests.

### Why is it useful?

- Applications can identify which Pod instance they are running in.
- Logs can include Pod name and namespace.
- Monitoring agents can read Pod metadata.
- Applications can access configured CPU and memory requests.
- Containers can consume selected labels and annotations without Kubernetes API credentials.

Important: The Downward API does not provide unrestricted access to the entire Kubernetes API. It exposes only supported fields.

## 2. How the Downward API Works

1. **Pod metadata and configuration:** Name, namespace, labels, annotations, resource requests.
2. **Delivery method:**
   - **Environment variables:** Read when the container starts.
   - **Volume files:** Read from mounted files.
3. **Application inside the container** reads the values.

The kubelet makes the supported information available to the container through the configured environment variables or mounted volume.

## 3. Downward API: Environment Variables

Use `fieldRef` to expose supported Pod fields and `resourceFieldRef` to expose container resource information.

### Example YAML

File: `downward-env.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: downward-env-demo
  namespace: default
  labels:
    app: nginx
    environment: dev
spec:
  containers:
    - name: nginx
      image: nginx:1.28
      env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: APP_LABELS
          valueFrom:
            fieldRef:
              fieldPath: metadata.labels
        - name: CPU_REQUEST
          valueFrom:
            resourceFieldRef:
              resource: requests.cpu
        - name: MEMORY_LIMIT
          valueFrom:
            resourceFieldRef:
              resource: limits.memory
      resources:
        requests:
          cpu: "250m"
          memory: "128Mi"
        limits:
          cpu: "500m"
          memory: "256Mi"
```

Apply the Pod:

```bash
kubectl apply -f downward-env.yaml
```

Check the status:

```bash
kubectl get pod downward-env-demo -o wide
```

Display the values inside the container:

```bash
kubectl exec downward-env-demo -- env
```

Display selected values:

```bash
kubectl exec downward-env-demo -- printenv POD_NAME
kubectl exec downward-env-demo -- printenv POD_NAMESPACE
kubectl exec downward-env-demo -- printenv POD_IP
kubectl exec downward-env-demo -- printenv NODE_NAME
kubectl exec downward-env-demo -- printenv CPU_REQUEST
kubectl exec downward-env-demo -- printenv MEMORY_LIMIT
```

The actual values depend on the cluster and the Pod's configuration.

### Important environment variable fields

| Field | Information exposed |
|---|---|
| metadata.name | Pod name |
| metadata.namespace | Namespace name |
| metadata.uid | Pod UID |
| metadata.labels | Pod labels |
| metadata.annotations | Pod annotations |
| status.podIP | Pod IP address |
| spec.nodeName | Node hosting the Pod |
| spec.serviceAccountName | Pod's service account name |

Only supported field paths can be used. For the full supported list, consult the Kubernetes Downward API documentation.

## 4. fieldRef vs resourceFieldRef

These are the two key references used by the Downward API.

| Feature | fieldRef | resourceFieldRef |
|---|---|---|
| Purpose | Exposes supported Pod fields | Exposes container resource information |
| Examples | Pod name, namespace, IP | CPU request, memory limit |
| Main field | fieldPath | resource |
| Resource container selection | Not applicable | Can use containerName when necessary |

### fieldRef example

```yaml
env:
  - name: MY_POD_NAME
    valueFrom:
      fieldRef:
        fieldPath: metadata.name
```

### resourceFieldRef example

```yaml
env:
  - name: MY_MEMORY_LIMIT
    valueFrom:
      resourceFieldRef:
        containerName: nginx
        resource: limits.memory
        divisor: 1Mi
```

Here, `divisor: 1Mi` specifies the unit used when presenting the resource quantity. The value is a numeric representation, not a string with an Mi suffix.

Remember:

- `fieldRef` → Pod fields.
- `resourceFieldRef` → Container resource requests and limits.

## 5. Downward API Using a Volume

The Downward API can also write supported information into files in a mounted volume.

This is useful when applications expect configuration files instead of environment variables.

### Example YAML

File: `downward-volume.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: downward-volume-demo
  labels:
    app: nginx
    environment: dev
  annotations:
    owner: platform-team
spec:
  containers:
    - name: nginx
      image: nginx:1.28
      volumeMounts:
        - name: podinfo
          mountPath: /etc/podinfo
          readOnly: true
  volumes:
    - name: podinfo
      downwardAPI:
        items:
          - path: pod-name
            fieldRef:
              fieldPath: metadata.name
          - path: pod-namespace
            fieldRef:
              fieldPath: metadata.namespace
          - path: pod-labels
            fieldRef:
              fieldPath: metadata.labels
          - path: pod-annotations
            fieldRef:
              fieldPath: metadata.annotations
```

Apply:

```bash
kubectl apply -f downward-volume.yaml
```

Read the files:

```bash
kubectl exec downward-volume-demo -- ls -l /etc/podinfo
kubectl exec downward-volume-demo -- cat /etc/podinfo/pod-name
kubectl exec downward-volume-demo -- cat /etc/podinfo/pod-namespace
kubectl exec downward-volume-demo -- cat /etc/podinfo/pod-labels
kubectl exec downward-volume-demo -- cat /etc/podinfo/pod-annotations
```

The Pod name and namespace files contain their respective values. Labels and annotations are written as text representations of the metadata.

## 6. Environment Variables vs Downward API Volume

| Feature | Environment variables | Downward API volume |
|---|---|---|
| Access method | Read through process environment | Read through files |
| Configuration | `env` | `volumes.downwardAPI` |
| Good for | Startup configuration | Applications expecting files |
| Metadata updates | Environment values do not update automatically | Supported projected metadata files can be updated |
| Resource values | Supported through `resourceFieldRef` | Supported through volume items using `resourceFieldRef` |

Important: A volume-backed Downward API can reflect updates to supported metadata fields, such as labels and annotations. Environment variables are fixed when the container starts. Applications reading volume files must handle file updates appropriately.

## 7. Downward API for CPU and Memory

The Downward API can expose the container's configured resource requests and limits.

### Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-info-demo
spec:
  containers:
    - name: app
      image: nginx:1.28
      env:
        - name: CPU_REQUEST
          valueFrom:
            resourceFieldRef:
              resource: requests.cpu
        - name: CPU_LIMIT
          valueFrom:
            resourceFieldRef:
              resource: limits.cpu
        - name: MEMORY_REQUEST
          valueFrom:
            resourceFieldRef:
              resource: requests.memory
        - name: MEMORY_LIMIT
          valueFrom:
            resourceFieldRef:
              resource: limits.memory
      resources:
        requests:
          cpu: "500m"
          memory: "128Mi"
        limits:
          cpu: "1"
          memory: "256Mi"
```

Check the values:

```bash
kubectl apply -f resource-info.yaml
kubectl exec resource-info-demo -- printenv CPU_REQUEST
kubectl exec resource-info-demo -- printenv CPU_LIMIT
kubectl exec resource-info-demo -- printenv MEMORY_REQUEST
kubectl exec resource-info-demo -- printenv MEMORY_LIMIT
```

The default resource divisor affects the representation of the quantity. If you require a particular unit, configure `divisor` explicitly.

Important: These variables expose configured resource quantities, not actual live CPU or memory usage. Use monitoring tools such as `kubectl top` for usage metrics.

## 8. Downward API vs ConfigMap vs Secret

These features have different purposes.

| Feature | Downward API | ConfigMap | Secret |
|---|---|---|---|
| Main purpose | Expose Pod metadata and resource configuration | Store application configuration | Store sensitive configuration |
| Example | Pod name, namespace, labels | Application settings | Passwords, tokens |
| Typical source | Pod fields | User-defined configuration | User-defined sensitive data |
| API access credentials needed inside the container | No, for supported Downward API fields | No, when injected as environment variables or mounted files | No, when injected as environment variables or mounted files |

A Downward API volume does not replace ConfigMaps or Secrets. Use the feature that matches the source and sensitivity of the data.

## 9. Downward API vs Kubernetes API Access

These are not the same.

### Downward API

- Exposes a defined set of fields about the current Pod.
- Does not require the application to call the Kubernetes API server.
- Does not provide arbitrary access to other Pods or cluster resources.

### Kubernetes API access

- Allows an application to query or modify permitted resources.
- Normally uses a ServiceAccount identity and appropriate RBAC permissions.
- Is necessary when an application needs broader cluster information.

For example, an application can read its own Pod name using the Downward API. To list all Pods in a namespace, it generally needs Kubernetes API access with the appropriate permissions.

## 10. Common Troubleshooting

### Problem 1: Environment variable is empty

Check the Pod YAML:

```bash
kubectl get pod downward-env-demo -o yaml
kubectl describe pod downward-env-demo
```

Verify:

- `valueFrom` is configured correctly.
- `fieldPath` or `resource` is supported and correctly written.
- The container was recreated if the environment variable configuration changed.

### Problem 2: File does not exist

Check the volume and mount configuration:

```bash
kubectl describe pod downward-volume-demo
kubectl exec downward-volume-demo -- ls -la /etc/podinfo
```

Verify that:

- The volume is defined under `spec.volumes`.
- The volume name matches `volumeMounts.name`.
- The expected file path matches the configured `items.path`.

### Problem 3: Wrong resource quantity

Check the actual Pod resource configuration:

```bash
kubectl get pod resource-info-demo -o yaml
kubectl describe pod resource-info-demo
```

Verify the resource name, request, limit, and divisor.

### Problem 4: Application expects changing metadata

If the application needs to react to changing labels or annotations, consider a Downward API volume instead of environment variables. Ensure the application rereads the files as needed.

## 11. OpenShift Notes

The Downward API is a Kubernetes feature and is also available in OpenShift, subject to supported field paths and the platform version.

Useful commands:

```bash
oc get pods -n <project-name>
oc describe pod <pod-name> -n <project-name>
oc get pod <pod-name> -n <project-name> -o yaml
oc exec <pod-name> -n <project-name> -- printenv POD_NAME
```

When using OpenShift, remember that a Pod's namespace is the OpenShift project in which it runs. The Downward API does not bypass project security, quota, or admission policies.

## 12. Interview Questions and Answers

**Q1. What is the Downward API?**
It exposes selected Pod metadata and container resource information to containers through environment variables or volume files.

**Q2. What are the two ways to use the Downward API?**
Environment variables and a Downward API volume.

**Q3. What is the difference between fieldRef and resourceFieldRef?**
`fieldRef` exposes supported Pod fields. `resourceFieldRef` exposes configured container resource requests and limits.

**Q4. Can the Downward API provide the Pod IP address?**
Yes. Use `status.podIP` with `fieldRef`.

**Q5. Can the Downward API show actual CPU usage?**
No. It can expose configured resource requests and limits. Actual usage requires a metrics source.

**Q6. Does the Downward API require Kubernetes API credentials inside the container?**
No, not for the supported fields it exposes.

**Q7. Can an application read its Pod labels through the Downward API?**
Yes. Labels can be exposed through an environment variable or a volume file.

**Q8. What is the difference between a Downward API volume and an environment variable?**
Environment variables are set for the container process at startup. Supported metadata in a Downward API volume can be updated in the projected files.

**Q9. Can the Downward API list all Pods in a namespace?**
No. That requires Kubernetes API access with suitable RBAC permissions.

**Q10. How do you debug Downward API configuration?**
Inspect the Pod YAML, check environment variables or mounted files inside the container, and review Pod events.

## 13. Quick Revision

- Downward API: Exposes selected Pod metadata and resource configuration.
- Environment variables: Use `env`, `fieldRef`, and `resourceFieldRef`.
- Volume files: Use `volumes.downwardAPI`.
- Pod name: `metadata.name`
- Namespace: `metadata.namespace`
- Pod IP: `status.podIP`
- Node name: `spec.nodeName`
- CPU request: `requests.cpu`
- Memory limit: `limits.memory`
- Actual resource usage: Use metrics, not the Downward API.
- Broader cluster information: Use Kubernetes API access with appropriate RBAC.

Final takeaway: Use the Downward API when a container needs information about its own Pod or configured resources without making Kubernetes API calls.
