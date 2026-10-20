# Kubernetes and OpenShift Notes: Pod Creation Workflow

## 1. What Is Pod Creation Workflow?

Pod creation workflow explains how a Pod is created, scheduled onto a worker node, and started inside a Kubernetes or OpenShift cluster.

In simple words, a developer submits a Pod YAML file, Kubernetes processes the request, selects a worker node, and starts the application container.

Kubernetes and OpenShift follow the same basic workflow.

## 2. Pod Creation Workflow

Developer → YAML File → API Server → Authentication and Authorization → Admission Control → etcd → Scheduler → Kubelet → Container Runtime → Container Running

Note: The API Server coordinates these operations. The Scheduler and Kubelet act at different stages, and the exact internal processing includes additional controller and networking steps.

## 3. Step-by-Step Explanation

### Step 1: Developer Creates a YAML File

The developer defines the Pod configuration.

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
      ports:
        - containerPort: 80
```

This file specifies the Pod name, container image, and expected application port.

### Step 2: Developer Submits the Request

Kubernetes command:

```bash
kubectl apply -f pod.yaml
```

OpenShift command:

```bash
oc apply -f pod.yaml
```

The CLI sends the API request to the Kubernetes API Server.

### Step 3: API Server Validates the Request

The API Server checks the request, including authentication, authorization, schema validation, and applicable admission policies.

If the request is rejected, the Pod is not created.

OpenShift can apply additional platform security policies.

### Step 4: Pod Configuration Is Stored

After the request is accepted, the API Server stores the Pod object in etcd, the cluster's backing datastore.

The stored object represents the desired Pod configuration.

At this stage, the container may not yet be running.

### Step 5: Scheduler Selects a Worker Node

The Scheduler identifies an eligible node based on factors such as:

- Available CPU and memory.
- Resource requests.
- Node selectors and affinity rules.
- Taints and tolerations.
- Scheduling constraints.

After selecting a node, the Scheduler records the Pod's node assignment.

If no suitable node is available, the Pod can remain in Pending state.

### Step 6: Kubelet Receives the Pod Assignment

The Kubelet on the selected worker node observes the assigned Pod and works to make the actual state match the desired configuration.

It coordinates container creation and other required setup operations.

### Step 7: Container Runtime Starts the Container

The Kubelet works with the configured container runtime, such as containerd or CRI-O, through the Container Runtime Interface (CRI).

The runtime obtains the image if necessary and creates and starts the container.

### Step 8: Pod Networking Is Configured

The cluster's networking implementation configures the Pod's network connectivity and IP address.

The exact sequence depends on the runtime, CNI implementation, and cluster configuration.

### Step 9: Application Starts

The container entry point starts the application.

For example, the Nginx container starts the Nginx web server, which normally listens on port 80.

### Step 10: Pod Becomes Ready

The Pod can enter Running phase when its containers are running. This does not automatically mean the application is ready.

Readiness probes and other readiness conditions determine whether the Pod should receive normal application traffic through a Service.

## 4. Pod Creation Commands

### Kubernetes

Create the Pod:

```bash
kubectl apply -f pod.yaml
```

Check Pod status:

```bash
kubectl get pods
```

Check node assignment and IP:

```bash
kubectl get pods -o wide
```

Check detailed events:

```bash
kubectl describe pod nginx-pod
```

Check application logs:

```bash
kubectl logs nginx-pod
```

### OpenShift

Create a project:

```bash
oc new-project pod-demo
```

Create the Pod:

```bash
oc apply -f pod.yaml
```

Check Pod status:

```bash
oc get pods
```

Check node assignment and IP:

```bash
oc get pods -o wide
```

Check detailed events:

```bash
oc describe pod nginx-pod
```

Check application logs:

```bash
oc logs nginx-pod
```

## 5. Common Pod Creation Problems

**Pending:**
The Pod has not completed scheduling or setup. Check resource availability, node selectors, affinity rules, taints, and PVC status.

**ErrImagePull or ImagePullBackOff:**
The image cannot be pulled. Check the image name, tag, registry access, and credentials.

**ContainerCreating:**
The container is still being prepared. Check image pulling, volume mounting, networking, and Pod events.

**CrashLoopBackOff:**
The container repeatedly exits. Check application logs, startup commands, configuration, and resource limits.

**CreateContainerConfigError:**
The container configuration cannot be prepared. Check referenced Secrets, ConfigMaps, and other required configuration.

**OpenShift-specific permission issues:**
Check Pod events and service account permissions.

Useful troubleshooting commands:

```bash
kubectl describe pod nginx-pod
kubectl logs nginx-pod --previous
kubectl get events --sort-by=.metadata.creationTimestamp
```

In OpenShift, use the corresponding `oc` commands.

## 6. Kubernetes vs OpenShift Workflow

| Stage | Kubernetes | OpenShift |
|---|---|---|
| CLI | kubectl | oc |
| API Server | Kubernetes API Server | Kubernetes API Server |
| Datastore | etcd | etcd |
| Scheduling | Kubernetes Scheduler | Kubernetes Scheduler |
| Node agent | Kubelet | Kubelet |
| Container runtime | Cluster-configured runtime | Cluster-configured runtime |
| Additional security | Kubernetes security controls | Kubernetes controls plus OpenShift features |

The core Pod creation workflow is the same in both platforms.

## 7. Important Interview Questions

**Q1. What happens after running `kubectl apply -f pod.yaml`?**
The API Server processes the request, validates it, and stores the accepted Pod object. The Scheduler assigns a node, and the Kubelet works with the container runtime to start the container.

**Q2. What is the role of etcd?**
etcd stores Kubernetes cluster state, including Pod objects and their configuration.

**Q3. What is the role of the Scheduler?**
The Scheduler selects a suitable worker node for an unscheduled Pod.

**Q4. What is the role of the Kubelet?**
The Kubelet ensures that Pods assigned to its node are running according to their specifications.

**Q5. Does a Pod become Ready immediately after creation?**
No. The Pod must be scheduled, its containers must start, and its readiness conditions must be satisfied.

**Q6. Is the workflow different in OpenShift?**
The core workflow is the same. OpenShift adds platform-specific security, networking, and management capabilities.

## 8. Quick Revision

1. Developer creates the YAML file.
2. CLI submits the request to the API Server.
3. API Server authenticates, authorizes, validates, and processes the request.
4. The accepted Pod object is stored in etcd.
5. Scheduler selects a worker node.
6. Kubelet on that node coordinates Pod setup.
7. Container runtime starts the container.
8. Cluster networking configures Pod connectivity.
9. Application starts.
10. Readiness conditions determine when the Pod is ready for traffic.

Remember: The API Server accepts and coordinates requests, the Scheduler selects a node, the Kubelet manages the assigned Pod, and the container runtime runs the container.
