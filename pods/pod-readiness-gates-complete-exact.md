# Kubernetes Pod Readiness Gates – Complete Notes

## 1. What Are Pod Readiness Gates?

Pod Readiness Gates allow Kubernetes to consider additional conditions, beyond container readiness probes, when determining whether a Pod is Ready.

A Pod can have all its containers running and their readiness probes passing, yet still need an external condition to be satisfied before it should receive traffic.

Readiness gates let you define those additional conditions using custom Pod conditions.

For example:

- A cloud load balancer must finish registering a Pod as a backend.
- An external system must complete application initialization.
- A deployment controller must confirm that a Pod is ready to serve traffic.
- A custom operator must verify a prerequisite before the Pod is considered Ready.

Important: Readiness gates do not perform the external check themselves. Another component must set the corresponding Pod condition to `True`.

## 2. Why Are Readiness Gates Needed?

Consider an application running in Kubernetes behind an external load balancer.

The container starts successfully, and its readiness probe passes. However, the load balancer has not finished registering the Pod as a healthy backend.

Without an additional readiness condition, Kubernetes might consider the Pod Ready before the external traffic path is ready.

A readiness gate can make the Pod's overall Ready status depend on both:

1. Container readiness.
2. The external readiness condition.

## 3. How Readiness Gates Work

1. **Pod starts.**
2. **Two requirements are evaluated:**
   - **Container readiness:** The readiness probe passes.
   - **External condition:** A load balancer or controller confirms readiness.
3. **Pod is Ready only when all required conditions are satisfied.**

The readiness gate names a condition type. Kubernetes expects that condition to be present in the Pod's status and have the value `True`.

If the condition is missing or is `False` or `Unknown`, the readiness gate is not satisfied.

## 4. Readiness Gate YAML Example

File: `readiness-gate.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: readiness-gate-demo
  labels:
    app: nginx
spec:
  readinessGates:
    - conditionType: example.com/external-ready
  containers:
    - name: nginx
      image: nginx:1.28
      ports:
        - containerPort: 80
      readinessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 5
```

### Explanation

- `readinessGates`: Lists additional conditions required for the Pod to be Ready.
- `conditionType`: Names the additional condition Kubernetes must check.
- `readinessProbe`: Checks whether the container is ready to serve traffic.
- `containerPort`: Documents the container port; it does not expose the application externally.

In this example, the readiness gate is `example.com/external-ready`.

Important: This is a demonstration condition name. Kubernetes does not automatically create or set it to `True`.

## 5. What Happens If the Readiness Condition Is Missing?

Apply the example:

```bash
kubectl apply -f readiness-gate.yaml
```

Check the Pod:

```bash
kubectl get pod readiness-gate-demo
kubectl describe pod readiness-gate-demo
```

Even if the NGINX readiness probe passes, the Pod can remain `0/1` Ready because the custom readiness condition has not been satisfied.

Inspect the condition:

```bash
kubectl get pod readiness-gate-demo \
  -o jsonpath='{.status.conditions}'
```

The Pod's status will typically show the custom condition as missing or not satisfied, depending on how it is represented by the Kubernetes version and controllers.

## 6. Who Sets the Readiness Gate Condition?

A readiness gate is a condition requirement, not a probe or an automatic controller.

An external controller or authorized component must update the Pod's status with a condition such as:

```yaml
status:
  conditions:
    - type: example.com/external-ready
      status: "True"
      reason: ExternalSystemReady
      message: "External dependency is ready"
```

This illustrates the desired condition. It is not a complete Pod manifest to apply with `kubectl apply`.

Normally, controllers update Pod status using the Kubernetes API, often through the Pod status subresource. The component needs suitable permissions.

If nobody sets the custom condition to `True`, the readiness gate remains unsatisfied.

## 7. Readiness Gates and Kubernetes Services

A Kubernetes Service normally sends traffic to eligible ready endpoints.

When a Pod has a readiness gate that is not satisfied:

- The Pod's overall Ready condition remains false.
- The Pod is generally excluded from normal ready Service endpoints.
- Traffic routing can be delayed until the additional condition becomes true.

Check the Pod:

```bash
kubectl get pods -o wide
```

Check Service endpoints:

```bash
kubectl get endpoints <service-name>
kubectl get endpointslices
```

A Service's `publishNotReadyAddresses` setting and other traffic-routing configurations can change the usual endpoint behavior, so verify the actual Service configuration when troubleshooting.

## 8. Readiness Probe vs Readiness Gate

These two features are related but different.

| Feature | Readiness probe | Readiness gate |
|---|---|---|
| Purpose | Checks whether a container is ready | Adds an external condition to Pod readiness |
| Defined in | Container specification | Pod specification |
| Example | HTTP request to `/health` | Load balancer registration condition |
| Who evaluates it? | Kubelet | Kubernetes Pod readiness logic evaluates the condition; another component must set it |
| Automatically checks an external system? | Only if the configured probe checks it | No |
| Effect on Pod readiness | Container readiness contributes to Pod readiness | Adds another required condition |

Interview point: A readiness probe tests application readiness. A readiness gate allows an external controller or system to contribute to Pod readiness.

## 9. Readiness Gates vs Liveness and Startup Probes

| Feature | Readiness gate | Liveness probe | Startup probe |
|---|---|---|---|
| Main purpose | Add an external readiness condition | Detect a container that should be restarted | Detect when application startup has completed |
| Failure effect | Pod may remain NotReady | Repeated failures can restart the container | Repeated failures can restart the container |
| External controller required? | Usually yes, to set the condition | No, not for ordinary probe operation | No, not for ordinary probe operation |
| Controls Service traffic? | Indirectly through overall Pod readiness | Not directly; restart can affect availability | Not directly; startup controls when other probes begin |

Readiness gates do not replace health probes. An application may need both container health checks and an external readiness condition.

## 10. Real-World Example: Load Balancer Registration

Imagine a deployment where a Pod must be registered and marked healthy in an external load balancer before Kubernetes considers it Ready.

The sequence is:

1. The Pod starts.
2. Its readiness probe passes.
3. A load-balancer integration or controller registers the Pod as a backend.
4. The integration determines that the backend is healthy.
5. The integration sets the Pod's custom readiness condition to `True`.
6. Kubernetes can now mark the Pod Ready, provided its other readiness requirements are satisfied.

Readiness gates are useful when this external state is represented through Pod conditions by a compatible controller.

Do not assume that every cloud load-balancer integration uses a custom readiness gate. The exact behavior depends on the integration and its configuration.

## 11. Troubleshooting Readiness Gates

### Step 1: Check Pod readiness

```bash
kubectl get pods -o wide
```

Look for Pods that are running but not Ready.

### Step 2: Inspect conditions and events

```bash
kubectl describe pod <pod-name>
kubectl get pod <pod-name> \
  -o jsonpath='{.status.conditions}'
```

Check whether:

- Container readiness is `True`.
- The required custom condition is missing.
- The custom condition is `False` or `Unknown`.

### Step 3: Inspect the readiness gate configuration

```bash
kubectl get pod <pod-name> -o yaml
```

Check:

```yaml
spec:
  readinessGates:
    - conditionType: example.com/external-ready
```

Confirm that the condition type exactly matches the one set by the external controller.

### Step 4: Check the responsible controller

Identify which controller or integration should set the condition.

Verify:

- The controller is running.
- It can observe the Pod.
- It can access the external system.
- It has permission to update Pod status.
- It reports success or failure through the expected condition.

### Step 5: Check Service endpoints

```bash
kubectl get svc
kubectl get endpointslices
```

Confirm whether the Pod appears as a ready endpoint and whether the Service has special settings affecting unready endpoints.

### Common causes

1. The condition type is misspelled or does not match.
2. The external controller is not installed or running.
3. The controller cannot update Pod status.
4. The external system has not completed its check.
5. The readiness probe is failing independently.
6. The Pod has another readiness requirement that is not satisfied.

## 12. OpenShift Notes

OpenShift supports Kubernetes readiness gates, subject to the platform version and the availability of a compatible controller.

Useful commands:

```bash
oc get pods -n <project-name>
oc describe pod <pod-name> -n <project-name>
oc get pod <pod-name> -n <project-name> -o yaml
oc get events -n <project-name> \
  --sort-by=.metadata.creationTimestamp
```

When a Pod remains Running but NotReady, inspect its readiness probes and custom Pod conditions. Then identify the controller responsible for setting the readiness gate.

A readiness gate does not bypass OpenShift security policies, resource quotas, or scheduling constraints.

## 13. Interview Questions and Answers

**Q1. What is a Pod readiness gate?**
A readiness gate adds a custom condition that must be true before Kubernetes considers a Pod Ready.

**Q2. Why do we need readiness gates?**
They allow external systems or controllers to contribute to Pod readiness, such as confirming load balancer registration.

**Q3. Does Kubernetes automatically set a custom readiness gate to True?**
No. A compatible controller or authorized component must set the condition.

**Q4. What happens when a readiness gate condition is missing?**
The gate is unsatisfied, so the Pod normally remains NotReady even if its containers are ready.

**Q5. What is the difference between a readiness probe and a readiness gate?**
A readiness probe checks container readiness. A readiness gate adds an additional Pod-level condition, usually managed externally.

**Q6. Can readiness gates affect Service traffic?**
Yes. By keeping the Pod NotReady, a readiness gate generally prevents it from being used as a ready Service endpoint.

**Q7. Do readiness gates restart containers?**
No. An unsatisfied readiness gate affects readiness; it does not directly restart the container.

**Q8. How do you troubleshoot a stuck readiness gate?**
Inspect the Pod's conditions, verify the condition type, and check the external controller responsible for updating Pod status.

**Q9. Can a Pod have multiple readiness gates?**
Yes. All specified readiness gates must be satisfied for the readiness-gate requirement to pass.

**Q10. Are readiness gates a replacement for liveness and readiness probes?**
No. They complement health probes by incorporating additional conditions into Pod readiness.

## 14. Quick Revision

- Readiness gate: Adds a custom Pod readiness condition.
- YAML field: `spec.readinessGates`.
- Condition field: `conditionType`.
- Condition value required: `True`.
- Who sets it? An external controller or authorized component.
- Missing condition: Gate is unsatisfied; Pod generally remains NotReady.
- Traffic impact: Normally excludes the Pod from ready Service endpoints.
- Restart behavior: Does not directly restart the container.
- Troubleshooting: Inspect Pod conditions, events, controller health, permissions, and Service endpoints.

Final takeaway: Use Pod readiness gates when container health alone is insufficient to determine whether a Pod is ready to receive traffic. The external readiness condition must be managed by a compatible controller.
