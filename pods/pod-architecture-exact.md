# Kubernetes and OpenShift Notes: Pod Architecture and Components

## 1. What Is Pod Architecture?

Pod architecture explains how a Pod is structured, which components it contains, and how it runs an application inside a Kubernetes or OpenShift cluster.

A Pod is the smallest deployable unit in Kubernetes. It can contain one or more containers.

Example:
An Nginx Pod contains an Nginx container that runs a web server.

Kubernetes and OpenShift use the same basic Pod architecture. OpenShift adds platform-specific security and management features.

## 2. Pod Architecture Workflow

The basic workflow is:

Developer → YAML File → API Server → Scheduler → Worker Node → Kubelet → Container Runtime → Pod → Application

Step-by-step explanation:

1. Developer creates a Pod YAML file.
2. API Server validates the request and processes the resource.
3. Scheduler selects a suitable worker node.
4. Kubelet on the selected node receives the Pod specification.
5. Container Runtime creates and starts the container.
6. Container Networking Interface (CNI) provides Pod networking through the cluster networking implementation.
7. The application starts inside the container.
8. Readiness checks determine whether the application is ready to receive traffic.

Note: The Scheduler assigns the Pod to a node. The Kubelet and container runtime then work together to start its containers.

## 3. Main Components of a Pod

### 3.1 Containers

Containers run applications inside the Pod.

Example:
An Nginx container runs a web server.

A Pod can contain:

- One application container.
- Multiple closely related containers.
- A main container and a supporting sidecar container.

### 3.2 Shared Network Namespace

Containers in the same Pod share the Pod's network namespace.

This means:

- They share the Pod IP address.
- They can communicate using localhost.
- They share the same port space, so port conflicts must be avoided.

Example:
If Nginx listens on port 80, another container in the same Pod cannot independently bind to the same IP and port combination.

### 3.3 Volumes

Volumes provide storage to containers inside a Pod.

Common examples:

- emptyDir: Temporary storage associated with the Pod.
- PersistentVolumeClaim (PVC): Requests persistent storage.
- Secret volume: Provides sensitive configuration data to a container.

Containers can share files by mounting the same volume.

Important: A volume's persistence depends on its type. An emptyDir is deleted when the Pod is removed, while persistent storage can outlive a Pod.

### 3.4 Pod Specification (spec)

The spec section defines the desired Pod configuration.

It can contain:

- Containers and images.
- Ports.
- Environment variables.
- Volumes and volume mounts.
- Resource requests and limits.
- Readiness, liveness, and startup probes.
- Security settings and scheduling rules.

### 3.5 Pod Metadata

Metadata identifies and organizes the Pod.

Common fields:

- name: Pod name.
- namespace: Namespace in which the Pod exists.
- labels: Key-value pairs used to identify and select resources.
- annotations: Additional descriptive information.

In OpenShift, the user-facing term project is commonly used for a namespace.

### 3.6 Pod IP Address

Each Pod normally receives its own IP address from the cluster networking system.

The Pod IP is generally used for communication within the cluster. A Service provides a stable endpoint when Pods are replaced and their IP addresses change.

### 3.7 Init Containers and Sidecars

Init containers run setup tasks before the application containers start.

Sidecar containers provide supporting functionality alongside the main application, such as log collection or proxy services.

Modern Kubernetes also supports native sidecar containers using restartable init-container semantics.

## 4. Pod Architecture YAML Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
    - name: nginx
      image: nginx:1.28
      ports:
        - containerPort: 80
      resources:
        requests:
          cpu: "100m"
          memory: "64Mi"
        limits:
          cpu: "500m"
          memory: "128Mi"
      readinessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 10
```

Explanation

- apiVersion: v1 — Pod API version.
- kind: Pod — Resource type.
- metadata — Pod identification and labels.
- spec — Desired configuration.
- containers — Application containers.
- image — Container image to run.
- containerPort — Documents the application port.
- resources.requests — CPU and memory requested for scheduling.
- resources.limits — Resource usage limits.
- readinessProbe — Checks whether the application is ready to receive traffic.

## 5. Kubernetes and OpenShift Commands

### Kubernetes

```bash
kubectl apply -f pod.yaml
kubectl get pods -o wide
kubectl describe pod nginx-pod
kubectl logs nginx-pod
kubectl exec -it nginx-pod -- /bin/sh
```

### OpenShift

```bash
oc apply -f pod.yaml
oc get pods -o wide
oc describe pod nginx-pod
oc logs nginx-pod
oc exec -it nginx-pod -- /bin/sh
```

Both platforms support the same basic Pod operations.

## 6. Pod vs Worker Node Components

| Component | Responsibility |
|---|---|
| API Server | Receives and processes Kubernetes API requests |
| Scheduler | Selects a worker node for an unscheduled Pod |
| Kubelet | Ensures assigned Pods are running as specified |
| Container Runtime | Creates and runs containers |
| CNI plugin | Provides Pod networking |
| Container | Runs the application process |
| Pod | Groups containers and provides shared resources |
| Volume | Provides storage to containers |
| Readiness probe | Checks whether the application is ready |

## 7. Important Interview Questions

**Q1. What are the main components of a Pod?**
Containers, Pod networking, optional volumes, metadata, and the Pod specification.

**Q2. Do containers inside a Pod share the same IP?**
Yes. They share the Pod's network namespace and IP address.

**Q3. What is the role of the Kubelet?**
The Kubelet ensures that the containers in Pods assigned to its node run according to the Pod specification.

**Q4. What is the difference between a Pod and a node?**
A Pod is a deployable unit that runs containers. A node is the machine that provides the resources to run Pods.

**Q5. What is the role of a CNI plugin?**
It implements Pod networking, including network connectivity and IP configuration, according to the cluster's network design.

**Q6. What is the purpose of a readiness probe?**
It checks whether an application is ready to receive traffic. A failed readiness probe normally marks the container as not ready without automatically restarting it.

**Q7. Is Pod architecture different in OpenShift?**
The core architecture is the same. OpenShift adds platform capabilities such as integrated developer workflows and additional cluster management features.

## 8. Quick Revision

- A Pod is the smallest deployable unit in Kubernetes.
- A Pod contains one or more containers.
- Containers share the Pod's network namespace.
- Volumes provide storage to containers.
- The API Server processes requests.
- The Scheduler selects a node.
- The Kubelet manages assigned Pods.
- The container runtime runs containers.
- CNI provides Pod networking.

Kubernetes and OpenShift use the same basic Pod architecture.
