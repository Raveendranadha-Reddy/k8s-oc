# Kubernetes and OpenShift Notes: apiVersion, kind, metadata, spec

## 1. What are apiVersion, kind, metadata, and spec?

These are the four important fields commonly found at the top level of a Kubernetes resource YAML file.

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
```

Simple explanation:

- apiVersion = Which API version should be used?
- kind = What resource should be created?
- metadata = What is the resource's name and identity?
- spec = How should the resource be configured?

## 2. apiVersion

Example:

```yaml
apiVersion: v1
```

Meaning:

apiVersion specifies the Kubernetes API group and version used for the resource.

For a Pod, the API version is `v1`.

Other examples:

- Pod: v1
- Service: v1
- Deployment: apps/v1
- ReplicaSet: apps/v1
- StatefulSet: apps/v1
- Job: batch/v1
- CronJob: batch/v1

Interview answer:

apiVersion tells Kubernetes which API version to use when processing the resource definition.

## 3. kind

Example:

```yaml
kind: Pod
```

Meaning:

kind specifies the type of Kubernetes resource you want to create.

Examples:

- Pod
- Service
- Deployment
- ConfigMap
- Secret
- StatefulSet
- DaemonSet

Interview answer:

kind defines which Kubernetes resource we want to create or manage.

## 4. metadata

Example:

```yaml
metadata:
  name: nginx-pod
  labels:
    app: nginx
```

Meaning:

metadata contains identifying information about the resource.

Important fields:

- name: The resource name.
- labels: Key-value pairs used to identify and select resources.
- namespace: The namespace in which the resource exists.
- annotations: Additional descriptive information for tools and integrations.

Example:

```yaml
metadata:
  name: nginx-pod
  namespace: default
  labels:
    app: nginx
  annotations:
    description: "Nginx web server"
```

Interview answer:

metadata provides the resource's name, namespace, labels, annotations, and other identifying information.

## 5. spec

Example:

```yaml
spec:
  containers:
    - name: nginx
      image: nginx:1.28
      ports:
        - containerPort: 80
```

Meaning:

spec defines the desired configuration of the resource.

For a Pod, it can specify:

- Containers and images
- Ports
- Environment variables
- CPU and memory requests and limits
- Volumes and volume mounts
- Readiness and liveness probes
- Scheduling rules
- Security settings

Interview answer:

spec defines the desired state and configuration that Kubernetes should maintain for the resource.

## 6. How do these fields work together?

Consider this example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
    - name: nginx
      image: nginx:1.28
```

Step-by-step:

1. apiVersion tells Kubernetes to use the v1 API.
2. kind tells Kubernetes that the resource is a Pod.
3. metadata gives the Pod its name: nginx-pod.
4. spec defines the container that should run.
5. Kubernetes processes the request and schedules the Pod onto a suitable worker node.
6. The node's Kubelet works with the container runtime to start the container.

## 7. Kubernetes vs OpenShift

| Field | Kubernetes | OpenShift |
|---|---|---|
| apiVersion | Kubernetes API version | Same Kubernetes API version |
| kind | Resource type | Same resource type |
| metadata | Resource identity | Same basic structure |
| spec | Desired configuration | Same basic structure, subject to platform policies |

Both platforms use these fields in their standard Kubernetes resource definitions.

## 8. Quick revision for interviews

- apiVersion: Which API version?
- kind: Which resource?
- metadata: What is its identity?
- spec: What should it look like and how should it work?

Easy memory trick:

API version → Resource type → Resource identity → Desired configuration.

Important: Not every resource has exactly the same spec fields. The fields available under spec depend on the resource kind.
