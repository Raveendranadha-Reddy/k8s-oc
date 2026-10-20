# Kubernetes Notes: Ephemeral Containers

## 1. What Is an Ephemeral Container?

An ephemeral container is a temporary container added to an existing Pod, primarily for troubleshooting and debugging.

It is useful when the application container is failing or when the application image does not contain debugging tools such as `sh`, `curl`, `ps`, or `tcpdump`.

Unlike regular containers, ephemeral containers are not designed to be restarted or managed as long-running application containers.

Real-time example: Your CoffeeShop backend Pod is running, but you cannot troubleshoot network connectivity because its minimal application image does not contain diagnostic tools. You can add an ephemeral debug container to investigate the Pod's environment.

## 2. Why Do We Use Ephemeral Containers?

Common use cases include:

- Troubleshooting application failures.
- Debugging network connectivity.
- Inspecting DNS resolution.
- Investigating processes in a running container, when process namespace sharing is available.
- Using diagnostic tools without rebuilding the application image.
- Debugging containers that lack a shell.

## 3. Ephemeral Container Architecture

- **Existing Kubernetes Pod:** Existing Pod IP and network namespace.
- **Application Container:** Runs the workload.
- **Ephemeral Container:** Runs debugging tools.

Both containers share the Pod network namespace. Filesystem and process visibility depend on the configured mounts and namespaces.

## 4. How to Create an Ephemeral Container

### Step 1: Find the target Pod

```bash
kubectl get pods -n coffeeshop
```

Example output:

```
NAME                               READY   STATUS
coffee-backend-7c8d9f6b5d-abcde    1/1     Running
```

### Step 2: Add a debug container

```bash
kubectl debug -it coffee-backend-7c8d9f6b5d-abcde \
  -n coffeeshop \
  --image=busybox:1.36 \
  --target=backend \
  -- sh
```

Explanation:

- `kubectl debug` — starts a debugging session.
- `-it` — requests an interactive terminal.
- `--image=busybox:1.36` — specifies the debugging image.
- `--target=backend` — requests targeting the application's process namespace, if supported by the runtime.
- `-- sh` — starts a shell in the debug container.

Replace the example Pod name and `backend` with the actual Pod and container names.

Important: `--target` depends on container runtime support. Process visibility is not guaranteed in every environment.

### Step 3: Run diagnostic commands

Inside the debug container:

```bash
hostname
cat /etc/resolv.conf
nslookup kubernetes.default
```

If the image contains the required tools, you can also test application connectivity:

```bash
wget -O- http://coffee-backend:8080
```

Replace `coffee-backend` with the actual Kubernetes Service name.

### Step 4: Exit the session

```bash
exit
```

Exiting the shell ends the interactive session. Depending on the debug container's command, the ephemeral container may terminate, but its record remains associated with the Pod until the Pod is replaced or deleted.

## 5. Debugging a Container That Has No Shell

Suppose the application uses a minimal or distroless image.

Running this command may fail:

```bash
kubectl exec -it coffee-backend-7c8d9f6b5d-abcde \
  -n coffeeshop -- sh
```

Reason: the application image may not include `sh`.

Instead, add an ephemeral container with diagnostic tools:

```bash
kubectl debug -it coffee-backend-7c8d9f6b5d-abcde \
  -n coffeeshop \
  --image=busybox:1.36 \
  -- sh
```

This gives you a shell in the debug container, even though the application container itself has no shell.

Limitation: The debug container does not automatically see the application container's filesystem or processes. Additional namespace targeting, permissions, and supported debugging options may be required.

## 6. Ephemeral Containers vs Regular Containers vs Init Containers

| Feature | Regular Container | Init Container | Ephemeral Container |
|---|---|---|---|
| Main purpose | Run the application | Initialize the Pod | Troubleshoot a Pod |
| When it starts | During application startup | Before regular application containers | Added when needed |
| Lifecycle | Runs according to its configuration | Traditional init container normally completes first | Temporary; not intended for normal application operation |
| Automatically restarted | According to Pod restart policy | According to Pod restart policy | No |
| Declared in | `spec.containers` | `spec.initContainers` | `spec.ephemeralContainers` |
| Typical example | NGINX | Dependency check | Debug shell |

## 7. How to Inspect Ephemeral Containers

List the Pod:

```bash
kubectl get pod coffee-backend-7c8d9f6b5d-abcde -n coffeeshop
```

Inspect its ephemeral container details:

```bash
kubectl get pod coffee-backend-7c8d9f6b5d-abcde \
  -n coffeeshop \
  -o jsonpath='{.spec.ephemeralContainers[*].name}'
```

Describe the Pod:

```bash
kubectl describe pod coffee-backend-7c8d9f6b5d-abcde -n coffeeshop
```

The Ephemeral Containers section can show the debug container's configuration and status.

## 8. Important Limitations

- You cannot add an ephemeral container by editing the ordinary `spec.containers` list.
- Ephemeral containers are added through the Kubernetes ephemeral-container API, commonly using `kubectl debug`.
- They are not automatically restarted if they exit.
- Their resource configuration has limitations compared with regular containers.
- They do not automatically share the target container's filesystem.
- Access may be restricted by RBAC, Pod Security controls, admission policies, or runtime capabilities.
- Ephemeral containers cannot simply be removed individually from a Pod specification after creation. The record remains until the Pod is replaced or deleted.

## 9. OpenShift Commands

The same core functionality is available in OpenShift, subject to cluster permissions and supported versions.

```bash
oc get pods -n coffeeshop
oc debug pod/coffee-backend-7c8d9f6b5d-abcde -n coffeeshop
```

For a Kubernetes-style ephemeral debug container, where supported:

```bash
oc debug -it pod/coffee-backend-7c8d9f6b5d-abcde \
  -n coffeeshop \
  --image=busybox:1.36 \
  --target=backend \
  -- sh
```

OpenShift's `oc debug` has additional debugging workflows. Check the command's help and your cluster version if a particular option is unsupported.

## 10. Interview Questions and Answers

**Q1. What is an ephemeral container?**
A temporary container added to an existing Pod primarily for troubleshooting and debugging.

**Q2. When do we use ephemeral containers?**
When we need diagnostic tools or a shell to investigate a Pod, especially when the application image lacks those tools.

**Q3. Can an ephemeral container be restarted automatically?**
No. Ephemeral containers are not automatically restarted under the Pod's restart policy.

**Q4. Do ephemeral containers have a separate Pod IP?**
No. They share the Pod's network namespace and IP address.

**Q5. Can an ephemeral container access another container's processes?**
Potentially, if process namespace targeting is supported and correctly configured. It is not guaranteed in every runtime.

**Q6. What is the difference between `kubectl exec` and `kubectl debug`?**
`kubectl exec` runs a command inside an existing container. `kubectl debug` can add an ephemeral container or create a debugging copy of a Pod or node, depending on the options used.

**Q7. Can we delete only an ephemeral container from a Pod?**
There is no supported operation to individually remove an ephemeral container from the Pod specification. Usually, the Pod must be replaced or deleted to remove its ephemeral-container record.

## 11. Quick Revision

- Ephemeral containers are mainly for debugging.
- They can provide tools missing from a minimal application image.
- Use `kubectl debug` to add an ephemeral container.
- They share the Pod network namespace.
- Filesystem and process visibility are not automatic.
- Ephemeral containers are not automatically restarted.
- Their records remain until the Pod is replaced or deleted.
- Kubernetes and OpenShift support related debugging workflows, subject to version and permission constraints.
