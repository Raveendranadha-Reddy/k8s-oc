# Kubernetes Projected Volumes – Complete Notes

## 1. What Is a Projected Volume?

A Projected Volume allows Kubernetes to combine data from multiple sources into a single directory inside a Pod.

Instead of mounting several volumes separately, you can project multiple sources into one volume.

Supported sources include:

1. `secret` — sensitive configuration data.
2. `configMap` — application configuration.
3. `downwardAPI` — Pod metadata and resource information.
4. `serviceAccountToken` — a token for authenticating to the Kubernetes API.

Example: A container needs an application configuration file, a TLS certificate, Pod metadata, and a ServiceAccount token. A projected volume can make these available under one directory.

## 2. Why Are Projected Volumes Used?

Projected volumes are useful when an application needs configuration or identity information from multiple Kubernetes resources.

Common use cases:

- Combining ConfigMaps and Secrets.
- Providing certificates and application configuration in one directory.
- Exposing Pod name, namespace, labels, and annotations.
- Providing a ServiceAccount token to an application that calls the Kubernetes API.
- Simplifying the container's volume mount configuration.

## 3. How Projected Volumes Work

1. **Projected Volume** — One volume mounted inside the container.
2. **Sources combined into the volume:**
   - **ConfigMap** — Application settings.
   - **Secret** — Credentials or certificates.
   - **Downward API** — Pod metadata.
   - **ServiceAccount token** — API authentication.
3. **Container directory:** `/etc/app-config`

Kubernetes presents the selected data sources as files under the mounted directory. Each source can use a different filename and file permission mode.

## 4. Basic Projected Volume YAML

File: `projected-volume.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: projected-volume-demo
spec:
  containers:
    - name: app
      image: nginx:1.28
      volumeMounts:
        - name: combined-config
          mountPath: /etc/projected
          readOnly: true
  volumes:
    - name: combined-config
      projected:
        sources:
          - configMap:
              name: app-config
          - secret:
              name: app-secret
```

This YAML combines a ConfigMap and a Secret into one volume.

Prerequisite: The ConfigMap `app-config` and Secret `app-secret` must exist in the same namespace as the Pod. Create them before deploying this Pod.

## 5. Create ConfigMap and Secret

### Step 1: Create the ConfigMap

```bash
kubectl create configmap app-config \
  --from-literal=APP_MODE=production \
  --from-literal=LOG_LEVEL=info
```

Verify:

```bash
kubectl get configmap app-config -o yaml
```

### Step 2: Create the Secret

```bash
kubectl create secret generic app-secret \
  --from-literal=username=admin \
  --from-literal=password='ChangeMe-123'
```

Verify its metadata:

```bash
kubectl describe secret app-secret
```

Avoid putting real credentials directly in shell history or YAML files. The example password is only for lab use.

### Step 3: Create the Pod

```bash
kubectl apply -f projected-volume.yaml
```

### Step 4: Inspect the files

```bash
kubectl get pod projected-volume-demo
kubectl exec projected-volume-demo -- ls -la /etc/projected
```

Because no custom paths are specified, the projected filenames are normally the ConfigMap keys and Secret keys:

```bash
kubectl exec projected-volume-demo -- cat /etc/projected/APP_MODE
kubectl exec projected-volume-demo -- cat /etc/projected/username
```

The files contain the corresponding values.

## 6. Customize Projected Volume Paths

You can use `items` to choose which keys to project and where each file appears.

### Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: projected-path-demo
spec:
  containers:
    - name: app
      image: nginx:1.28
      volumeMounts:
        - name: app-files
          mountPath: /etc/app
          readOnly: true
  volumes:
    - name: app-files
      projected:
        sources:
          - configMap:
              name: app-config
              items:
                - key: APP_MODE
                  path: config/application-mode
          - secret:
              name: app-secret
              items:
                - key: username
                  path: credentials/username
```

Read the files:

```bash
kubectl exec projected-path-demo -- \
  cat /etc/app/config/application-mode
kubectl exec projected-path-demo -- \
  cat /etc/app/credentials/username
```

The resulting paths are:

```
/etc/app/
├── config/
│   └── application-mode
└── credentials/
    └── username
```

The referenced keys must exist in their respective ConfigMap and Secret.

## 7. Projected Volume with Downward API

A Pod can combine application configuration with its own metadata.

### Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: projected-downward-demo
  labels:
    app: nginx
    environment: dev
spec:
  containers:
    - name: app
      image: nginx:1.28
      volumeMounts:
        - name: pod-info
          mountPath: /etc/pod-info
          readOnly: true
  volumes:
    - name: pod-info
      projected:
        sources:
          - downwardAPI:
              items:
                - path: pod-name
                  fieldRef:
                    fieldPath: metadata.name
                - path: namespace
                  fieldRef:
                    fieldPath: metadata.namespace
                - path: labels
                  fieldRef:
                    fieldPath: metadata.labels
```

Apply and inspect:

```bash
kubectl apply -f projected-downward.yaml
kubectl exec projected-downward-demo -- \
  cat /etc/pod-info/pod-name
kubectl exec projected-downward-demo -- \
  cat /etc/pod-info/namespace
kubectl exec projected-downward-demo -- \
  cat /etc/pod-info/labels
```

This is useful when an application needs to identify its own Pod without querying the Kubernetes API.

## 8. Projected Volume with ServiceAccount Token

A projected ServiceAccount token can provide an application with a time-limited token for Kubernetes API authentication.

### Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: projected-token-demo
spec:
  serviceAccountName: default
  automountServiceAccountToken: false
  containers:
    - name: app
      image: nginx:1.28
      volumeMounts:
        - name: api-identity
          mountPath: /var/run/secrets/kubernetes.io/serviceaccount
          readOnly: true
  volumes:
    - name: api-identity
      projected:
        sources:
          - serviceAccountToken:
              path: token
              expirationSeconds: 3600
```

The token is made available at:

```
/var/run/secrets/kubernetes.io/serviceaccount/token
```

### Important points

- `expirationSeconds` requests a token lifetime; the API server may return a different lifetime according to its configuration.
- Kubernetes can refresh projected ServiceAccount tokens. Applications must reread the token file rather than assume the token never changes.
- The token's permissions depend on the ServiceAccount and its RBAC bindings.
- Disabling automatic token mounting does not disable the explicitly configured projected token volume.
- For applications that need API access, use a dedicated ServiceAccount with least-privilege permissions instead of relying on the default ServiceAccount.

## 9. Projected Volume with Multiple Sources

The following example combines four source types in one directory.

Prerequisites: Create the ConfigMap `app-config` and Secret `app-secret` as shown earlier.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: all-sources-demo
  labels:
    app: demo
spec:
  containers:
    - name: app
      image: nginx:1.28
      volumeMounts:
        - name: combined-data
          mountPath: /etc/combined
          readOnly: true
  volumes:
    - name: combined-data
      projected:
        sources:
          - configMap:
              name: app-config
              items:
                - key: APP_MODE
                  path: config/mode
          - secret:
              name: app-secret
              items:
                - key: username
                  path: credentials/username
          - downwardAPI:
              items:
                - path: metadata/pod-name
                  fieldRef:
                    fieldPath: metadata.name
          - serviceAccountToken:
              path: identity/token
              expirationSeconds: 3600
```

Inspect the directory:

```bash
kubectl apply -f all-sources-demo.yaml
kubectl exec all-sources-demo -- \
  find /etc/combined -type f
```

Expected file layout:

```
/etc/combined/
├── config/
│   └── mode
├── credentials/
│   └── username
├── metadata/
│   └── pod-name
└── identity/
    └── token
```

The example illustrates the supported source types. Use only the sources and paths required by your application.

## 10. Projected Volume vs Separate Volumes

| Feature | Projected volume | Separate volumes |
|---|---|---|
| Number of mount points | Can use one mount point for several sources | Often uses one mount point per source |
| Data sources | Combines supported sources | Each volume has its own source |
| File organization | Can place files under custom relative paths | Paths are controlled by each volume and mount |
| Updates | Supported projected data can be refreshed | Refresh behavior depends on volume type |
| Access control | Still subject to source permissions and Kubernetes security controls | Same general principle |

Projected volumes simplify file organization. They do not merge the underlying Kubernetes resources into one ConfigMap or Secret.

## 11. Projected Volumes vs Downward API Volumes

| Feature | Projected volume | Downward API volume |
|---|---|---|
| Purpose | Combines data from multiple supported sources | Exposes supported Pod metadata and resource information |
| ConfigMaps | Supported | Not directly |
| Secrets | Supported | Not directly |
| ServiceAccount token | Supported | Not directly |
| Pod metadata | Supported through a `downwardAPI` source | Directly supported |
| YAML field | `projected` | `downwardAPI` |

A projected volume can include a Downward API source, so you can use both capabilities together.

## 12. Important Limitations and Best Practices

1. **Sources must be available.** ConfigMaps and Secrets referenced by the Pod must exist in the same namespace.
2. **Mounts are read-only.** Projected volume files are intended to be provided by Kubernetes, not edited by the application.
3. **Updates are not necessarily instantaneous.** Kubernetes refreshes supported projected data, but applications should account for propagation delays.
4. **Environment variables do not automatically refresh.** If updated configuration must be read dynamically, use an appropriate file-based approach or restart the application.
5. **Avoid overlapping file paths.** Two sources that project to the same relative path can cause conflicts.
6. **Protect Secret data.** A Secret projected as a file is still sensitive data; restrict access to the mounted directory and use appropriate cluster encryption and access controls.
7. **Handle token rotation.** Applications using projected ServiceAccount tokens should reread the token file when necessary.
8. **Use least privilege.** A mounted token does not automatically grant permissions; RBAC determines what the identity can do.

## 13. Troubleshooting Projected Volumes

### Problem 1: Pod is stuck in ContainerCreating

Check:

```bash
kubectl describe pod <pod-name>
kubectl get events --sort-by=.metadata.creationTimestamp
```

Look for missing ConfigMaps or Secrets, mount failures, and other volume setup errors.

### Problem 2: Expected file is missing

Check:

```bash
kubectl get pod <pod-name> -o yaml
kubectl exec <pod-name> -- ls -la <mount-path>
```

Verify the volume name, mount path, source names, key names, and `items.path`.

### Problem 3: Permission denied

Check the projected file permissions and the container's security context:

```bash
kubectl exec <pod-name> -- ls -lR <mount-path>
kubectl describe pod <pod-name>
```

Remember that OpenShift security policies can affect the container's user identity and filesystem access.

### Problem 4: Updated configuration is not visible

Check whether the data comes from a ConfigMap, Secret, or Downward API source. Confirm that the application rereads the projected file and does not rely on a cached copy or environment variable.

### Problem 5: Kubernetes API authentication fails

Check that:

- The token file exists.
- The application rereads the token after rotation.
- The API server address and CA certificate are configured correctly.
- The ServiceAccount has the required RBAC permissions.
- The token audience and expiry are appropriate for the target API.

## 14. OpenShift Commands

Projected volumes use the same core Kubernetes API in OpenShift.

```bash
oc get pods -n <project-name>
oc describe pod <pod-name> -n <project-name>
oc get pod <pod-name> -n <project-name> -o yaml
oc get configmap -n <project-name>
oc get secret -n <project-name>
oc get serviceaccount -n <project-name>
```

Inspect the mounted files:

```bash
oc exec <pod-name> -n <project-name> -- ls -la <mount-path>
```

Use a dedicated ServiceAccount and least-privilege RBAC when an application needs Kubernetes API access.

## 15. Interview Questions and Answers

**Q1. What is a projected volume?**
A projected volume combines data from multiple supported sources into a single volume mounted inside a Pod.

**Q2. Which sources can be projected?**
ConfigMaps, Secrets, Downward API data, and ServiceAccount tokens.

**Q3. What is the main advantage of projected volumes?**
They simplify mounting related configuration, metadata, and identity files through one volume mount.

**Q4. Can a projected volume combine a ConfigMap and a Secret?**
Yes. Both can be listed under `volumes[].projected.sources`.

**Q5. Can projected volumes expose Pod metadata?**
Yes. Add a `downwardAPI` source.

**Q6. Can projected volumes provide Kubernetes API credentials?**
Yes. A `serviceAccountToken` source can provide a projected token. The application's permissions are still controlled by RBAC.

**Q7. Are projected volume files writable by the application?**
They are provided as read-only volume data. Applications should not attempt to modify the projected files.

**Q8. Do projected volume updates happen instantly?**
No. Supported updates propagate eventually, and applications should account for delays.

**Q9. Does mounting a ServiceAccount token grant cluster-admin access?**
No. The token authenticates as a ServiceAccount; RBAC determines the permitted actions.

**Q10. How do you troubleshoot a missing projected file?**
Inspect the Pod specification and events, then verify the source resource, key, relative path, and volume mount.

## 16. Quick Revision

- Projected volume: Combines multiple supported data sources into one volume.
- YAML field: `projected`.
- ConfigMap source: Application configuration.
- Secret source: Sensitive data.
- Downward API source: Pod metadata and supported resource information.
- ServiceAccount token source: Time-limited API authentication token.
- `items.path`: Controls where a source key appears inside the volume.
- Mount behavior: Read-only; supported data may be refreshed.
- Security: Protect Secrets and use least-privilege ServiceAccounts.
- Troubleshooting: Check events, source resources, keys, paths, and mount configuration.

Final takeaway: Use projected volumes when an application needs files from multiple Kubernetes sources in one directory without creating a separate mount for every source.
