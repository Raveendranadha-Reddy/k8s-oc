# Kubernetes Pod Logs, Events and Audit Troubleshooting – Complete Notes

## 1. Introduction

When a Pod fails in Kubernetes, three important sources help identify the root cause:

1. **Pod logs** — show what the application or container process reported.
2. **Kubernetes events** — show important cluster actions and failures, such as scheduling errors, image pull failures, and probe failures.
3. **Audit logs** — record API server requests and actions, helping investigate who performed an operation, what resource was affected, and whether the request succeeded.

These sources answer different troubleshooting questions.

| Source | Main question | Example |
|---|---|---|
| Pod logs | What happened inside the application? | Java application failed to connect to a database |
| Events | What happened during Pod lifecycle or scheduling? | FailedScheduling, BackOff, FailedMount |
| Audit logs | Which API request caused or attempted a change? | A user deleted a Deployment |

Important: Application logs, events, and audit logs are different data sources. One may explain the symptom while another identifies the underlying cause.

## 2. Pod Logs

### What are Pod logs?

Pod logs are the standard output (stdout) and standard error (stderr) streams produced by containers.

Kubernetes provides commands to retrieve these logs through the container runtime and kubelet.

Common use cases:

- Application startup failures.
- Exceptions and stack traces.
- Database connection errors.
- HTTP errors.
- Container crash investigation.
- Previous container instance failures.

### Basic commands

List Pods:

```bash
kubectl get pods -A
```

Read logs:

```bash
kubectl logs <pod-name>
```

Specify a namespace:

```bash
kubectl logs <pod-name> -n <namespace>
```

Follow logs continuously:

```bash
kubectl logs -f <pod-name> -n <namespace>
```

Show recent lines:

```bash
kubectl logs <pod-name> --tail=100 -n <namespace>
```

Show logs from a time range:

```bash
kubectl logs <pod-name> \
  --since=30m \
  -n <namespace>
```

Show timestamps:

```bash
kubectl logs <pod-name> \
  --timestamps \
  -n <namespace>
```

### Troubleshooting crashed containers

When a container restarts, its previous instance's logs are often essential.

```bash
kubectl logs <pod-name> --previous -n <namespace>
```

For a multi-container Pod:

```bash
kubectl logs <pod-name> \
  -c <container-name> \
  -n <namespace>
```

Previous logs for a specific container:

```bash
kubectl logs <pod-name> \
  -c <container-name> \
  --previous \
  -n <namespace>
```

Important: `--previous` retrieves logs from the previous terminated container instance when available. It does not retrieve unlimited historical logs from every restart.

### Example: CrashLoopBackOff

```bash
kubectl get pod coffee-backend -n coffeeshop
kubectl describe pod coffee-backend -n coffeeshop
kubectl logs coffee-backend -n coffeeshop --previous
```

Possible application log:

```
APPLICATION FAILED TO START
Web server failed to start. Port 8080 was already in use.
```

This points to an application startup or port conflict problem, rather than a scheduling problem.

## 3. Multi-Container Pod Logs

A Pod can contain multiple containers, such as:

- Application container.
- Sidecar proxy.
- Log collector.
- Metrics exporter.

List the containers:

```bash
kubectl get pod <pod-name> \
  -o jsonpath='{.spec.containers[*].name}' \
  -n <namespace>
```

Read a specific container's logs:

```bash
kubectl logs <pod-name> \
  -c <container-name> \
  -n <namespace>
```

Read logs from all regular containers:

```bash
kubectl logs <pod-name> \
  --all-containers=true \
  -n <namespace>
```

For init containers:

```bash
kubectl logs <pod-name> \
  -c <init-container-name> \
  -n <namespace>
```

For previous instances of a specific container:

```bash
kubectl logs <pod-name> \
  -c <container-name> \
  --previous \
  -n <namespace>
```

Remember that an init container may have completed successfully and is no longer running. Its logs can still be useful for diagnosing initialization failures.

## 4. Kubernetes Events

### What are Kubernetes events?

Events are records of notable changes or conditions involving Kubernetes resources.

Examples:

- Pod scheduling failures.
- Image pull failures.
- Container restart backoff.
- Failed volume mounts.
- Readiness or liveness probe failures.
- Node pressure and eviction.
- Failed resource creation.

### Basic event commands

List events in the current namespace:

```bash
kubectl get events
```

List events across all namespaces:

```bash
kubectl get events -A
```

Sort events by creation timestamp:

```bash
kubectl get events \
  --sort-by=.metadata.creationTimestamp
```

Filter events for a specific Pod:

```bash
kubectl describe pod <pod-name> -n <namespace>
```

`kubectl describe` includes relevant events for the resource when available.

### Important event fields

| Field | Meaning |
|---|---|
| Type | Usually Normal or Warning |
| Reason | Short identifier for the event |
| Message | Explanation of what happened |
| Object / InvolvedObject | Resource associated with the event |
| Count or Series | Repeated event information, depending on API representation |
| FirstTimestamp / LastTimestamp | Legacy timestamp fields; newer event representations use other time fields |

The exact columns displayed by `kubectl` vary by Kubernetes version and event representation.

### Common event reasons

| Event reason | Likely meaning |
|---|---|
| FailedScheduling | No suitable node currently satisfies scheduling requirements |
| Pulling | Image download has started |
| Failed | An operation failed; inspect the message for details |
| BackOff | Kubernetes is delaying another attempt, commonly after repeated failures |
| FailedMount | A volume could not be mounted |
| Unhealthy | A health probe failed |
| Evicted | Pod was evicted |
| SuccessfulCreate | A controller successfully created a resource |

Event reasons are not guaranteed to be identical in every version or failure scenario.

## 5. Troubleshooting with Events

### Case 1: Pod stuck in Pending

```bash
kubectl get pod <pod-name> -n <namespace>
kubectl describe pod <pod-name> -n <namespace>
kubectl get events -n <namespace> \
  --sort-by=.metadata.creationTimestamp
```

Example event:

```
Warning  FailedScheduling
0/3 nodes are available: insufficient cpu.
```

Interpretation:

- The scheduler could not find a suitable node.
- CPU requests and node allocatable resources must be checked.
- Increasing the replica count alone will not resolve a capacity shortage.

### Case 2: ImagePullBackOff

```bash
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace>
```

Look for events indicating:

- Image not found.
- Invalid image tag.
- Registry authentication failure.
- Network or DNS failure.

For image pull errors, events are often more useful than application logs because the application container may never have started.

### Case 3: FailedMount

```bash
kubectl describe pod <pod-name> -n <namespace>
kubectl get pvc -n <namespace>
kubectl get pv
```

Check:

- PVC status.
- StorageClass and CSI driver.
- Volume permissions.
- Secret or ConfigMap existence.
- Node-specific attachment or mount errors.

### Case 4: Liveness or readiness probe failure

```bash
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --previous
```

Check the probe path, port, initial delay, timeout, application startup time, and dependencies.

A failed readiness probe makes the container unready; it does not by itself restart the container. Repeated liveness or startup probe failures can cause container restarts.

## 6. Kubernetes Audit Logs

### What are audit logs?

Kubernetes audit logs record API server requests and the associated audit events.

They help investigate:

- Who created or deleted a Pod.
- Who updated a Deployment.
- Which identity changed a ConfigMap or Secret.
- Whether an API request was allowed or denied.
- What resource was affected.
- When an API operation occurred.
- Which client or source address made the request, when recorded.

Important: Audit logs record Kubernetes API activity. They are not a replacement for application logs, and they do not automatically record every command or action performed inside a container.

For example, an audit log may record a request to delete a Pod, but the Pod's application logs are needed to understand what happened inside the application before it terminated.

## 7. Audit Policy and Audit Levels

The Kubernetes API server uses an audit policy to determine which requests are logged and how much request information is recorded.

Common audit levels:

| Level | Information recorded |
|---|---|
| None | Do not log matching requests |
| Metadata | Request metadata, such as identity, verb, resource, and result |
| Request | Metadata plus request body, when applicable |
| RequestResponse | Metadata, request body, and response body, when applicable |

Higher detail can increase storage costs and may expose sensitive data. Audit policy should be designed carefully.

### Example audit policy

The following is an illustrative policy fragment, not a complete production policy.

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  - level: Metadata
    resources:
      - group: ""
        resources:
          - pods
          - pods/log
          - events
  - level: Request
    resources:
      - group: apps
        resources:
          - deployments
  - level: None
    nonResourceURLs:
      - /healthz*
      - /readyz*
      - /livez*
```

Audit policy rules are evaluated in order; the first matching rule determines the audit level. A real policy must account for required resource groups, subresources, users, namespaces, and sensitive operations.

Security note: Avoid recording request and response bodies for Secrets or other sensitive resources unless there is a carefully justified need and suitable protection for the audit data.

## 8. Where Are Audit Logs Stored?

Audit log location depends on the Kubernetes distribution and API server configuration.

Common possibilities include:

- A file configured through API server audit flags.
- A managed control-plane audit logging service.
- A centralized logging platform.
- A security information and event management (SIEM) system.

For a self-managed Kubernetes cluster, inspect the API server configuration for options such as:

```
--audit-policy-file
--audit-log-path
--audit-log-maxage
--audit-log-maxbackup
--audit-log-maxsize
```

These are API server configuration flags, not commands to run directly in a normal application container.

For managed clusters, audit logs may be exposed through the cloud provider's logging service rather than as files on worker nodes.

## 9. Audit Troubleshooting Workflow

Suppose someone deleted a production Deployment.

### Step 1: Confirm the current state

```bash
kubectl get deployments -A
kubectl get events -A \
  --sort-by=.metadata.creationTimestamp
```

Events may provide context, but they might not preserve enough information to establish who initiated the deletion.

### Step 2: Locate the audit record

Search the configured audit log destination for:

- The relevant time range.
- `verb: delete`.
- The affected resource and namespace.
- The Deployment name.

### Step 3: Examine the audit event

Important fields commonly include:

| Field | Purpose |
|---|---|
| requestReceivedTimestamp | When the API server received the request |
| stageTimestamp | When the recorded audit stage occurred |
| user.username | Authenticated identity |
| verb | API action, such as create, update, patch, or delete |
| objectRef | Resource and namespace |
| sourceIPs | Recorded source addresses |
| responseStatus.code | API response status code |
| auditID | Identifier for correlating an audit request |

The exact available fields depend on the audit event stage and policy.

### Step 4: Determine the result

An API request appearing in an audit log does not automatically mean the operation succeeded. Check `responseStatus`, the audit stage, and any relevant response information.

### Step 5: Correlate with other evidence

Correlate the audit record with:

- Kubernetes events.
- Deployment and ReplicaSet history.
- Application logs.
- CI/CD pipeline logs.
- GitOps controller logs.
- Identity provider or access logs, when relevant.

This helps distinguish a manual operation from a deployment automation or controller action.

## 10. Application Logs vs Events vs Audit Logs

| Scenario | Best initial source |
|---|---|
| Java application throws an exception | Application logs |
| Pod cannot find a suitable node | Pod description and events |
| Image pull authentication fails | Events |
| Container repeatedly crashes | Current and previous container logs |
| PVC cannot mount | Events and PVC/CSI diagnostics |
| User or automation deleted a Deployment | Audit logs and CI/CD/GitOps records |
| API request was denied | Audit logs and API response details |
| Application returns HTTP 500 | Application logs and application telemetry |
| Node experiences memory pressure | Node conditions, events, kubelet logs and metrics |

Use multiple sources when the failure crosses component boundaries.

## 11. End-to-End Troubleshooting Procedure

When a Pod is failing, follow this sequence.

### Step 1: Check Pod status

```bash
kubectl get pods -n <namespace> -o wide
```

### Step 2: Inspect Pod details

```bash
kubectl describe pod <pod-name> -n <namespace>
```

### Step 3: Read application logs

```bash
kubectl logs <pod-name> -n <namespace>
```

If the container restarted:

```bash
kubectl logs <pod-name> \
  --previous \
  -n <namespace>
```

### Step 4: Inspect events

```bash
kubectl get events -n <namespace> \
  --sort-by=.metadata.creationTimestamp
```

### Step 5: Inspect dependencies

Depending on the error, check:

```bash
kubectl get nodes
kubectl get pvc -n <namespace>
kubectl get svc -n <namespace>
kubectl get endpointslices -n <namespace>
kubectl get configmaps -n <namespace>
kubectl get secrets -n <namespace>
```

### Step 6: Investigate API changes

If the incident involves a resource being changed or deleted, inspect the configured audit log destination and relevant CI/CD or GitOps logs.

### Step 7: Confirm the root cause

Do not conclude that an event is the root cause merely because it occurred near the failure time. Correlate timestamps, resource names, request identities, and application evidence.

## 12. OpenShift Troubleshooting Commands

OpenShift uses Kubernetes logs and events, with additional platform-specific logging and auditing mechanisms.

### Pod logs

```bash
oc logs <pod-name> -n <project-name>
oc logs <pod-name> --previous -n <project-name>
oc logs <pod-name> -c <container-name> -n <project-name>
```

### Events

```bash
oc get events -n <project-name> \
  --sort-by=.metadata.creationTimestamp
oc describe pod <pod-name> -n <project-name>
```

### Node details

```bash
oc get nodes
oc describe node <node-name>
```

### Audit logs

For OpenShift, the location and access method depend on the cluster type and audit configuration. On managed OpenShift services, control-plane audit logs may be exposed through provider logging services. In self-managed clusters, audit configuration and access depend on the control-plane deployment and cluster policies.

Do not assume that API server audit logs are available through `oc logs` on an application Pod.

## 13. Common Mistakes

1. **Using `kubectl logs` for a container that never started.** Use Pod events to diagnose image pulls, scheduling, and volume setup.
2. **Ignoring `--previous`.** The current container may have restarted and lost the most useful immediate context.
3. **Assuming events are permanent.** Events are retained for a limited period and are not a durable audit history.
4. **Assuming audit logs record everything inside a container.** They record API server activity according to the audit configuration.
5. **Assuming every API request succeeded.** Check the response status and audit stage.
6. **Collecting too much audit detail.** Request and response bodies can expose credentials or sensitive data.
7. **Troubleshooting without timestamps.** Correlate logs, events, and audit records using the same time window.
8. **Deleting the Pod before collecting evidence.** Capture logs, events, and relevant resource details first when possible.

## 14. Interview Questions and Answers

**Q1. What is the difference between Pod logs and Kubernetes events?**
Pod logs show application or container output. Events show notable Kubernetes lifecycle and control-plane actions.

**Q2. How do you get logs from a crashed container?**
Use `kubectl logs <pod-name> --previous -n <namespace>`.

**Q3. What is the first command you use to troubleshoot a Pending Pod?**
`kubectl describe pod <pod-name> -n <namespace>`, followed by inspection of the relevant scheduling events.

**Q4. What are Kubernetes audit logs?**
They record API server requests and associated information about the identity, operation, resource, timing, and result.

**Q5. Can audit logs identify who deleted a Deployment?**
They can, provided the relevant request was captured and retained, and the audit event contains sufficient identity and resource information.

**Q6. Do Kubernetes events provide a complete history of all changes?**
No. Events are operational signals with limited retention, not a complete or durable audit trail.

**Q7. What is the difference between Metadata and RequestResponse audit levels?**
Metadata records request metadata. RequestResponse can include metadata, the request body, and the response body.

**Q8. Why might `kubectl logs` return no useful output?**
The container may not have started, the application may not write to stdout or stderr, the logs may have rotated, or the wrong container or namespace may have been selected.

**Q9. How do you investigate a denied API request?**
Inspect the API response and relevant audit event, including the authenticated identity, verb, resource, and response status. Check RBAC and admission policies as appropriate.

**Q10. What is the recommended troubleshooting strategy?**
Start with Pod status and description, inspect current and previous logs, review events, check dependencies, and use audit logs when investigating API actions or access.

## 15. Quick Revision

- Pod logs: Application and container output.
- Events: Kubernetes lifecycle and operational signals.
- Audit logs: API server requests and recorded outcomes.
- Current logs: `kubectl logs <pod>`.
- Previous container logs: `kubectl logs <pod> --previous`.
- Pod events: `kubectl describe pod <pod>`.
- All namespace events: `kubectl get events -A`.
- Audit investigation: Check identity, verb, resource, timestamps, and response status.
- Security: Protect audit data and avoid unnecessary sensitive request/response bodies.
- Best practice: Correlate all three sources before concluding root cause.

Final takeaway: Use logs to understand application behavior, events to understand Kubernetes operations, and audit logs to investigate API activity and resource changes. Combining them provides a much stronger troubleshooting picture than relying on any one source.
