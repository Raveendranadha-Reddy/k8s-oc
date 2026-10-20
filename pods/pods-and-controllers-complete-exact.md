# Kubernetes Pods and Controllers – Complete Notes

## 1. What Is a Pod?

A Pod is the smallest deployable unit in Kubernetes. It can contain one or more containers that share a network namespace and can share mounted volumes.

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
    - name: nginx
      image: nginx:stable
      ports:
        - containerPort: 80
```

### Create and verify

```bash
kubectl apply -f pod.yaml
kubectl get pods
kubectl describe pod nginx-pod
kubectl get pods -o wide
```

### Important Pod characteristics

- A Pod has its own IP address in the cluster network.
- Containers within a Pod share the same network namespace.
- Containers can communicate using localhost.
- A Pod can mount shared volumes into multiple containers.
- A Pod is generally treated as disposable.
- A Pod is not normally the preferred resource for managing a long-running application directly.

If a standalone Pod is deleted, Kubernetes does not automatically recreate it merely because it was deleted.

This is where controllers become important.

## 2. What Is a Controller?

A Kubernetes controller continuously observes the actual cluster state and compares it with the desired state.

If a difference exists, it attempts to reconcile the cluster toward the desired state.

For example, if a Deployment requires three replicas but only two are running, its controllers work to restore the desired replica count.

### Controller workflow

1. User defines desired state.
2. API Server stores the desired state.
3. Controller observes cluster state.
4. Controller compares desired state with actual state.
5. Reconciliation takes place.
6. Resources are created, updated, or deleted as needed.

Key concept: Kubernetes uses a declarative model. You declare the desired state, and controllers continuously reconcile toward it.

## 3. Pod vs Controller

| Feature | Pod | Controller |
|---|---|---|
| Purpose | Runs application containers | Manages application resources |
| Desired replica count | Not inherently managed | Supported by appropriate controllers |
| Recreates a deleted Pod | No, not by itself | Usually, if a managed replica is still desired |
| Scaling | Not built in | Depends on controller type |
| Rolling updates | Not built in | Supported by Deployment |
| Typical use | Testing or debugging | Production workload management |

Interview point: A Pod runs the workload. A controller manages the lifecycle and desired state of Pods or other resources.

## 4. Main Kubernetes Controllers

Kubernetes provides several built-in workload controllers.

| Controller | Main Purpose |
|---|---|
| ReplicaSet | Maintains a specified number of matching Pod replicas |
| Deployment | Manages stateless applications, ReplicaSets, rolling updates, and rollbacks |
| StatefulSet | Manages workloads needing stable identities and ordered operations |
| DaemonSet | Runs a Pod on each eligible node, or a selected subset |
| Job | Runs a task to completion |
| CronJob | Creates Jobs on a schedule |
| ReplicationController | Legacy controller for maintaining Pod replicas |

Other controllers manage resources such as Nodes, endpoints, storage, and cluster infrastructure.

## 5. ReplicaSet

A ReplicaSet ensures that a specified number of matching Pods exist.

### Example YAML

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:stable
          ports:
            - containerPort: 80
```

### Deploy and verify

```bash
kubectl apply -f replicaset.yaml
kubectl get rs
kubectl get pods -l app=nginx
kubectl describe rs nginx-rs
```

Expected result: The ReplicaSet attempts to maintain three matching Pods.

### Test self-healing

```bash
kubectl get pods -l app=nginx
kubectl delete pod <one-pod-name>
kubectl get pods -l app=nginx -w
```

A replacement Pod should be created because the ReplicaSet still desires three replicas.

Important: In production, you normally create a Deployment rather than manage a ReplicaSet directly.

## 6. Deployment

A Deployment is the standard controller for many stateless applications.

It manages ReplicaSets, which in turn manage Pods.

### Architecture

```
Deployment
    |
    v
ReplicaSet
    |
    +---- Pod 1
    |
    +---- Pod 2
    |
    +---- Pod 3
```

### Deployment YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:stable
          ports:
            - containerPort: 80
```

### Commands

```bash
kubectl apply -f deployment.yaml
kubectl get deployments
kubectl get replicasets
kubectl get pods -l app=nginx
kubectl scale deployment nginx-deployment --replicas=5
kubectl rollout status deployment/nginx-deployment
kubectl rollout history deployment/nginx-deployment
```

### Update the image

```bash
kubectl set image deployment/nginx-deployment \
  nginx=nginx:1.28
```

Use a valid, approved image version for your environment.

### Roll back

```bash
kubectl rollout undo deployment/nginx-deployment
```

Important: A Deployment manages updates through ReplicaSets. It does not normally create Pods directly.

## 7. StatefulSet

A StatefulSet manages applications that require stable Pod identities and often stable storage associations.

Common use cases include databases, distributed systems, and clustered applications.

### Key characteristics

- Stable Pod names, such as `database-0` and `database-1`
- Stable network identity when used with a suitable headless Service
- Stable storage associations through `volumeClaimTemplates`
- Ordered creation, scaling, and termination according to the configured policy

### Example YAML

```yaml
apiVersion: v1
kind: Service
metadata:
  name: database
spec:
  clusterIP: None
  selector:
    app: database
  ports:
    - port: 5432
      name: database
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database
spec:
  serviceName: database
  replicas: 2
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
    spec:
      containers:
        - name: database
          image: postgres:16
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: database-secret
                  key: POSTGRES_PASSWORD
          volumeMounts:
            - name: database-data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: database-data
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 5Gi
```

Before deploying, create the required Secret:

```bash
kubectl create secret generic database-secret \
  --from-literal=POSTGRES_PASSWORD='replace-with-a-strong-password'
```

A suitable StorageClass must be available for dynamic provisioning, or the claims must be satisfied by appropriate PVs.

### Commands

```bash
kubectl apply -f statefulset.yaml
kubectl get statefulsets
kubectl get pods
kubectl get pvc
```

Important: StatefulSet storage is not automatically deleted just because a Pod is recreated. PVC and PV retention behavior depends on the resource lifecycle and configured policies. Also, StatefulSet does not itself provide database replication or backup.

## 8. DaemonSet

A DaemonSet ensures that a Pod runs on each eligible node, or each node selected by its scheduling rules.

Common use cases:

- Log collection agents
- Node monitoring agents
- Security agents
- Certain networking components

### Example YAML

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-agent
spec:
  selector:
    matchLabels:
      app: node-agent
  template:
    metadata:
      labels:
        app: node-agent
    spec:
      containers:
        - name: agent
          image: busybox:1.36
          command: ["sh", "-c", "sleep 3600"]
```

### Commands

```bash
kubectl apply -f daemonset.yaml
kubectl get daemonsets
kubectl get pods -o wide
kubectl describe daemonset node-agent
```

The exact number of Pods depends on eligible nodes, scheduling constraints, and tolerations.

## 9. Job

A Job runs one or more Pods to completion.

Typical uses include:

- Data migration
- Batch processing
- Report generation
- One-time maintenance tasks

### Example YAML

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-job
spec:
  backoffLimit: 2
  template:
    metadata:
      labels:
        app: batch-job
    spec:
      restartPolicy: Never
      containers:
        - name: worker
          image: busybox:1.36
          command: ["sh", "-c", "echo Processing data; sleep 5; echo Done"]
```

### Commands

```bash
kubectl apply -f job.yaml
kubectl get jobs
kubectl get pods
kubectl logs job/batch-job
kubectl describe job batch-job
```

The Job tracks completion and can create replacement Pods after failures according to its configuration.

Important: A Job's Pod template must use `restartPolicy: Never` or `restartPolicy: OnFailure`.

## 10. CronJob

A CronJob creates Jobs according to a schedule.

### Example YAML

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-cronjob
spec:
  schedule: "0 2 * * *"
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: busybox:1.36
              command:
                - sh
                - -c
                - echo "Run backup task"
```

The schedule `0 2 * * *` means 2:00 AM every day, interpreted using the CronJob's configured time zone; without an explicit time zone, Kubernetes uses the controller manager's local time zone, which is commonly UTC in many clusters.

This is a demonstration only: a real backup job needs actual backup commands, durable storage, and failure handling.

### Commands

```bash
kubectl apply -f cronjob.yaml
kubectl get cronjobs
kubectl get jobs
kubectl describe cronjob backup-cronjob
```

## 11. ReplicationController

ReplicationController is an older Kubernetes controller that maintains a desired number of matching Pods.

It is largely superseded by ReplicaSet and Deployment.

For new applications, prefer Deployment for stateless workloads.

## 12. How Self-Healing Works

Self-healing is one of Kubernetes' key features.

Suppose a Deployment declares three replicas.

| Situation | Expected behavior |
|---|---|
| One container crashes | Kubelet attempts a restart according to restart policy |
| A managed Pod is deleted | Its controller creates a replacement if needed |
| A worker node fails | Replacement Pods may be scheduled on healthy eligible nodes, depending on failure detection and constraints |
| Desired replicas increase | Controller creates additional Pods |
| Desired replicas decrease | Controller removes excess Pods |

Important distinction: The kubelet restarts containers within a Pod. A workload controller creates replacement Pods when its desired state requires them.

## 13. Scaling Controllers

### Scale a Deployment

```bash
kubectl scale deployment nginx-deployment --replicas=5
```

### Scale back down

```bash
kubectl scale deployment nginx-deployment --replicas=2
```

### Verify

```bash
kubectl get deployment nginx-deployment
kubectl get pods -l app=nginx
```

Scaling changes the desired replica count. It does not guarantee that all requested replicas will become Ready if resources, image pulls, storage, or other constraints prevent them from starting.

For automatic scaling, Kubernetes can use a HorizontalPodAutoscaler when suitable metrics and resource configuration are available.

## 14. Labels and Selectors

Controllers use label selectors to identify which Pods they manage.

Example:

```yaml
selector:
  matchLabels:
    app: web
```

The Pod template must include matching labels:

```yaml
template:
  metadata:
    labels:
      app: web
```

For Deployments and ReplicaSets, the selector must match the Pod template labels.

Warning: Incorrect selectors can prevent a controller from managing the intended Pods. Overlapping selectors among controllers can also cause unexpected behavior.

## 15. Pods and Controllers in OpenShift

OpenShift supports Kubernetes workload resources, including:

- Pods
- ReplicaSets
- Deployments
- StatefulSets
- DaemonSets
- Jobs
- CronJobs

Common commands:

```bash
oc get pods
oc get deployments
oc get replicasets
oc get statefulsets
oc get daemonsets
oc get jobs
oc get cronjobs
```

To inspect a workload:

```bash
oc describe deployment <deployment-name>
oc get pods -o wide
oc logs <pod-name>
```

OpenShift also supports platform-specific features and APIs, but the core workload-controller behavior is Kubernetes-based.

## 16. Practical Lab: Deployment Self-Healing

### Step 1: Create a Deployment

Save as `web-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
        - name: nginx
          image: nginx:stable
          ports:
            - containerPort: 80
```

### Step 2: Apply

```bash
kubectl apply -f web-deployment.yaml
```

### Step 3: Verify replicas

```bash
kubectl get deployment web-app
kubectl get replicasets
kubectl get pods -l app=web-app
```

### Step 4: Delete one Pod

```bash
kubectl get pods -l app=web-app
kubectl delete pod <one-pod-name>
```

### Step 5: Observe self-healing

```bash
kubectl get pods -l app=web-app -w
```

A replacement Pod should appear because the Deployment still requires three replicas.

### Step 6: Scale

```bash
kubectl scale deployment web-app --replicas=5
kubectl get pods -l app=web-app
```

### Step 7: Cleanup

```bash
kubectl delete deployment web-app
```

## 17. Troubleshooting

### Scenario 1: Deployment has fewer Ready replicas than desired

```bash
kubectl get deployment
kubectl describe deployment <deployment-name>
kubectl get pods
```

Check image errors, scheduling events, readiness probes, resource constraints, and application logs.

### Scenario 2: ReplicaSet does not create the expected Pods

```bash
kubectl describe rs <replicaset-name>
kubectl get events --sort-by=.metadata.creationTimestamp
```

Check the selector, Pod template, namespace quota, and admission errors.

### Scenario 3: StatefulSet Pods are stuck Pending

```bash
kubectl get pods
kubectl get pvc
kubectl describe pod <pod-name>
kubectl describe pvc <pvc-name>
```

Check storage provisioning, node constraints, and resource requests.

### Scenario 4: DaemonSet does not run on every node

```bash
kubectl get daemonset
kubectl describe daemonset <daemonset-name>
kubectl get nodes
```

Check node selectors, taints, tolerations, affinity, and whether nodes are eligible.

### Scenario 5: Job does not complete

```bash
kubectl get jobs
kubectl get pods
kubectl logs job/<job-name>
kubectl describe job <job-name>
```

Check the command's exit code, retry behavior, permissions, and dependencies.

## 18. Real-Time Interview Questions

**Q1. What is the difference between a Pod and a controller?**
A Pod runs containers. A controller manages desired state and lifecycle behavior for Pods or other resources.

**Q2. What is a ReplicaSet?**
A controller that maintains a specified number of matching Pod replicas.

**Q3. What is a Deployment?**
A controller that manages ReplicaSets and supports declarative updates, scaling, and rollbacks for stateless workloads.

**Q4. What is a StatefulSet?**
A controller for workloads that require stable Pod identities and often persistent storage associations.

**Q5. What is a DaemonSet?**
A controller that runs a Pod on each eligible node or selected subset of nodes.

**Q6. What is a Job?**
A controller that runs Pods to complete a task successfully.

**Q7. What is a CronJob?**
A controller that creates Jobs according to a schedule.

**Q8. What happens when you delete a Pod managed by a Deployment?**
The Deployment's ReplicaSet creates a replacement Pod if the desired replica count still requires it.

**Q9. Does deleting a standalone Pod cause it to be recreated?**
No. A standalone Pod has no controller responsible for recreating it.

**Q10. What is reconciliation?**
The continuous process of comparing desired state with actual state and making changes to reduce the difference.

**Q11. Why are labels and selectors important?**
They allow controllers to identify the Pods they manage.

**Q12. Which controller is preferred for stateless applications?**
Deployment.

**Q13. Which controller is suitable for node-level monitoring agents?**
DaemonSet.

**Q14. Which controller is suitable for scheduled backups?**
CronJob, usually creating Jobs that perform the backup.

**Q15. Does a StatefulSet automatically replicate database data?**
No. It provides stable identities and storage associations, but database replication and data protection require application-level configuration.

## 19. Quick Revision

| Resource | Main Purpose |
|---|---|
| Pod | Runs containers |
| ReplicaSet | Maintains replica count |
| Deployment | Manages ReplicaSets and rolling updates |
| StatefulSet | Stable identity and storage associations |
| DaemonSet | Pod on each eligible node |
| Job | Run a task to completion |
| CronJob | Schedule Jobs |
| ReplicationController | Legacy replica management |

Key takeaway: For most stateless applications, create a Deployment rather than a standalone Pod. Kubernetes controllers provide self-healing, scaling, and lifecycle management by continuously reconciling actual state with the desired state.
