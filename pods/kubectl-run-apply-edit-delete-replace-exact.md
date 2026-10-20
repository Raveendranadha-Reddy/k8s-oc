# Kubernetes Notes: kubectl run, apply, edit, delete, and replace

## 1. Introduction

kubectl is a command-line tool used to communicate with the Kubernetes API server and manage cluster resources.

These five commands are commonly used to create, update, modify, and delete Kubernetes resources.

| Command | Purpose |
|---|---|
| kubectl run | Create a Pod quickly |
| kubectl apply | Create or update resources using YAML |
| kubectl edit | Modify a live resource |
| kubectl delete | Delete a resource |
| kubectl replace | Replace an existing resource using a file |

## 2. kubectl run

### Definition

kubectl run is used to create a Pod quickly from the command line.

### Example 1: Create a Pod

```bash
kubectl run nginx --image=nginx
```

Explanation:

- run — creates a Pod.
- nginx — Pod name.
- `--image=nginx` — container image.

### Example 2: Create a Pod with a specific namespace

```bash
kubectl run nginx --image=nginx -n dev
```

### Example 3: Create a Pod with a port declaration

```bash
kubectl run nginx --image=nginx --port=80
```

Important: `--port=80` declares the container port; it does not expose the Pod externally.

### Verify the Pod

```bash
kubectl get pods
kubectl describe pod nginx
kubectl logs nginx
```

### OpenShift

```bash
oc run nginx --image=nginx
oc get pods
oc describe pod nginx
```

Remember: kubectl run is useful for quick testing and troubleshooting. For production applications, YAML manifests and controllers such as Deployments are generally preferred.

## 3. kubectl apply

### Definition

kubectl apply creates a resource if it does not exist or updates it according to the configuration file.

### Example: Create a Pod YAML

File: `nginx-pod.yaml`

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
      image: nginx:latest
      ports:
        - containerPort: 80
```

### Apply the YAML

```bash
kubectl apply -f nginx-pod.yaml
```

### Verify

```bash
kubectl get pods
kubectl describe pod nginx-pod
```

### Update the resource

Change the image in the YAML file:

```yaml
image: nginx:1.28
```

Then run:

```bash
kubectl apply -f nginx-pod.yaml
```

Important: An existing standalone Pod's container image can generally be changed only by recreating the Pod. Therefore, changing the image in a Pod YAML and applying it may fail because the Pod's specification is largely immutable. For a Deployment, changing the image in the manifest triggers a rollout.

### Apply all YAML files in a directory

```bash
kubectl apply -f ./manifests/
```

### OpenShift

```bash
oc apply -f nginx-pod.yaml
oc get pods
```

Remember: apply is the standard declarative approach for managing resources from YAML manifests.

## 4. kubectl edit

### Definition

kubectl edit opens a live Kubernetes resource in a text editor so you can modify its configuration directly.

### Example: Edit a Deployment

```bash
kubectl edit deployment nginx
```

You can modify fields such as the container image or replica count, provided the changes are valid.

Example:

```yaml
spec:
  replicas: 3
```

Save and exit the editor. Kubernetes processes the update.

### Verify

```bash
kubectl get deployment nginx
kubectl get pods
kubectl rollout status deployment/nginx
```

### Edit a resource in a specific namespace

```bash
kubectl edit deployment nginx -n dev
```

### OpenShift

```bash
oc edit deployment nginx
```

For an OpenShift DeploymentConfig:

```bash
oc edit deploymentconfig nginx
```

Important points:

- edit modifies the live resource.
- It does not automatically update your original YAML file on disk.
- Some fields are immutable and cannot be changed directly.
- Editing a Deployment's Pod template can trigger a new rollout.

Remember: Use edit for quick changes; use version-controlled YAML and apply for repeatable configuration management.

## 5. kubectl delete

### Definition

kubectl delete removes Kubernetes resources from the cluster.

### Example 1: Delete a Pod

```bash
kubectl delete pod nginx-pod
```

### Example 2: Delete a Deployment

```bash
kubectl delete deployment nginx
```

### Example 3: Delete using a YAML file

```bash
kubectl delete -f nginx-pod.yaml
```

### Example 4: Delete resources in a namespace

```bash
kubectl delete pod nginx-pod -n dev
```

### Example 5: Delete all Pods in the current namespace

```bash
kubectl delete pods --all
```

Warning: This deletes all Pods in the current namespace. Pods managed by Deployments or other controllers may be recreated automatically.

### Verify deletion

```bash
kubectl get pods
kubectl get deployments
```

### OpenShift

```bash
oc delete pod nginx-pod
oc delete deployment nginx
oc delete -f nginx-pod.yaml
```

Remember: Deleting a Pod does not necessarily delete its application permanently. If a Deployment manages it, the Deployment can create a replacement Pod.

## 6. kubectl replace

### Definition

kubectl replace updates an existing resource using the complete configuration supplied in a YAML or JSON file.

Unlike apply, replace expects the resource to exist already, unless you use the `--force` option.

### Example: Replace an existing resource

```bash
kubectl replace -f nginx-pod.yaml
```

The file must describe an existing resource, and the requested changes must be allowed by the Kubernetes API.

### Common issue with Pods

Pod specifications contain many immutable fields. If you change an immutable field, such as the container image of a standalone Pod, a normal replacement may fail.

For a disposable test Pod, you can use:

```bash
kubectl replace --force -f nginx-pod.yaml
```

This deletes and recreates the resource. It may cause downtime and is not a general production update strategy.

### Recommended approach for a Deployment

For example, update the Deployment's image directly:

```bash
kubectl set image deployment/nginx nginx=nginx:1.28
```

Then check the rollout:

```bash
kubectl rollout status deployment/nginx
kubectl rollout history deployment/nginx
```

### OpenShift

```bash
oc replace -f nginx-pod.yaml
```

Remember: replace submits a replacement configuration for an existing resource. Use it carefully, particularly with immutable fields and production workloads.

## 7. Main Differences

| Feature | run | apply | edit | delete | replace |
|---|---|---|---|---|---|
| Main purpose | Quick Pod creation | Declarative create/update | Edit live resource | Remove resource | Replace existing resource |
| Uses YAML file | Optional | Yes, commonly | No; edits live object | Optional | Yes |
| Creates if missing | Yes, for supported run usage | Yes | No | No | No, normally |
| Updates existing resource | Limited | Yes | Yes, if allowed | No | Yes, if allowed |
| Common use | Testing | Standard deployments | Quick troubleshooting change | Cleanup | Explicit resource replacement |

## 8. Real-Time Scenario

Suppose you need to deploy NGINX in a development namespace.

**Step 1: Create a Pod quickly**

```bash
kubectl run nginx --image=nginx -n dev
```

**Step 2: Inspect the Pod**

```bash
kubectl get pods -n dev
kubectl describe pod nginx -n dev
```

**Step 3: Manage a Deployment using YAML**

```bash
kubectl apply -f nginx-deployment.yaml
```

**Step 4: Change the replica count interactively**

```bash
kubectl edit deployment nginx -n dev
```

**Step 5: Check the rollout**

```bash
kubectl rollout status deployment/nginx -n dev
```

**Step 6: Delete a test Pod**

```bash
kubectl delete pod nginx -n dev
```

If a Deployment manages that Pod, Kubernetes will normally create a replacement.

## 9. Interview Questions and Answers

**Q1. What is the difference between kubectl run and kubectl apply?**
kubectl run is commonly used to create a Pod quickly from the command line. kubectl apply manages resources declaratively using YAML manifests.

**Q2. What is the difference between apply and replace?**
apply creates missing resources or updates existing resources declaratively. replace updates an existing resource using the supplied configuration and can fail when changes violate immutability rules.

**Q3. What is the difference between edit and apply?**
edit changes the live resource directly. apply uses a configuration file, which can be stored in version control.

**Q4. If you delete a Pod managed by a Deployment, what happens?**
The Deployment's ReplicaSet normally creates a replacement Pod to maintain the desired replica count.

**Q5. What does kubectl replace --force do?**
It deletes the existing resource and creates it again from the supplied file. This can cause downtime and should be used cautiously.

**Q6. Which command is preferred for production configuration management?**
`kubectl apply -f` with reviewed, version-controlled manifests is a common approach. Larger environments may use GitOps tools such as Argo CD or Flux.

## 10. Quick Revision

- `kubectl run` — create a Pod quickly.
- `kubectl apply -f file.yaml` — create or update resources declaratively.
- `kubectl edit resource name` — modify a live resource.
- `kubectl delete resource name` — delete a resource.
- `kubectl replace -f file.yaml` — replace the configuration of an existing resource.
- `kubectl replace --force -f file.yaml` — delete and recreate the resource.
- `oc` provides corresponding commands for OpenShift.

Production tip: Prefer Deployments over manually managed Pods for stateless applications, and use declarative manifests with version control to make changes repeatable and auditable.
