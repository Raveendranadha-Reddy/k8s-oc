# Kubernetes and OpenShift Notes: imagePullPolicy

## 1. What is imagePullPolicy?

imagePullPolicy is a Kubernetes Pod configuration field that tells the container runtime when to pull a container image from a registry.

In simple words, it controls whether Kubernetes should download the container image or use an image already available on the worker node.

Kubernetes and OpenShift use the same imagePullPolicy field.

## 2. Types of imagePullPolicy

There are three values:

### A. Always

Kubernetes attempts to pull or check the image from the registry whenever it launches a container.

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
    - name: nginx
      image: nginx:latest
      imagePullPolicy: Always
```

Use case:

Useful when deploying images with frequently updated tags, although immutable image tags or digests are safer for reliable deployments.

Important: `Always` does not necessarily download every image layer again. The runtime can reuse cached layers when appropriate.

### B. IfNotPresent

Kubernetes pulls the image only if it is not already present on the worker node.

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
    - name: nginx
      image: nginx:1.28
      imagePullPolicy: IfNotPresent
```

Use case:

Useful when deploying stable, versioned images and reducing unnecessary registry requests.

Important: Different worker nodes may have different cached images. Reusing a mutable tag can therefore lead to inconsistent application versions across nodes.

### C. Never

Kubernetes does not pull the image from the registry. The required image must already exist on the worker node.

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
    - name: nginx
      image: nginx:1.28
      imagePullPolicy: Never
```

Use case:

Useful in controlled environments where images are loaded onto nodes in advance.

If the image is missing on the selected node, the container cannot start.

## 3. Difference Between the Three Policies

**Always:**
Attempts to pull or check the image from the registry whenever the container starts.

**IfNotPresent:**
Uses the locally available image if present; otherwise, pulls it.

**Never:**
Uses only a locally available image and never pulls it.

## 4. Default imagePullPolicy Behavior

If you omit `imagePullPolicy`, Kubernetes sets a default based on the image reference when the Pod is first created.

- If the image tag is omitted or is `latest`, the default is `Always`.
- If you specify a tag other than `latest`, the default is `IfNotPresent`.
- If you specify an image digest without a tag, the default is `IfNotPresent`.

Important: Kubernetes sets this field when the object is first created. Changing the image reference later does not automatically recalculate the existing imagePullPolicy.

## 5. How to Check imagePullPolicy

Kubernetes:

```bash
kubectl get pod nginx-pod -o yaml
```

OpenShift:

```bash
oc get pod nginx-pod -o yaml
```

Look for:

```yaml
image: nginx:1.28
imagePullPolicy: IfNotPresent
```

## 6. Common Image Pull Errors

**ErrImagePull:**
The image pull attempt failed.

**ImagePullBackOff:**
Kubernetes is delaying further pull attempts after a failure.

Common causes:

- Incorrect image name or tag.
- Private registry authentication failure.
- Registry unavailable.
- Network or DNS problems.
- Image does not exist.

Troubleshooting commands:

```bash
kubectl describe pod nginx-pod
kubectl get events --sort-by=.metadata.creationTimestamp
```

OpenShift:

```bash
oc describe pod nginx-pod
oc get events --sort-by=.metadata.creationTimestamp
```

Check the Pod events for the exact failure reason.

## 7. Important Production Recommendation

For production deployments, prefer immutable image references, such as a versioned image tag that cannot be overwritten or an image digest.

Example:

```yaml
image: nginx:1.28
imagePullPolicy: IfNotPresent
```

For stronger reproducibility, use a verified image digest:

```yaml
image: nginx@sha256:<verified-image-digest>
imagePullPolicy: IfNotPresent
```

Replace the placeholder with the actual digest of the intended image.

## 8. Interview Questions

**Q1. What is imagePullPolicy?**
It defines when Kubernetes pulls a container image from a registry.

**Q2. What is the difference between Always and IfNotPresent?**
`Always` checks the registry when launching a container, while `IfNotPresent` uses a local image when available.

**Q3. What happens when imagePullPolicy is Never and the image is missing?**
The container cannot start because Kubernetes will not pull the image.

**Q4. What is the default policy for the latest tag?**
`Always`.

**Q5. Is imagePullPolicy different in OpenShift?**
No. OpenShift uses the same Kubernetes imagePullPolicy behavior.

## 9. Quick Revision

- Always = Attempt to pull or check the registry on container launch.
- IfNotPresent = Pull only when the image is absent locally.
- Never = Use only the local image.

Remember: imagePullPolicy controls image retrieval; it does not control how frequently the application restarts.
