# Kubernetes Pod Storage – Complete Notes

## 1. What is Pod Storage?

Pod Storage refers to the storage mechanisms used by containers running inside a Kubernetes Pod.

By default, data written to a container's writable filesystem is ephemeral. It can be lost when that container is recreated.

Kubernetes provides Volumes to allow containers to access shared, temporary, or persistent storage.

Why do Pods need storage?

- Store application-generated files
- Share files between containers in the same Pod
- Store configuration files and secrets
- Store temporary processing data
- Persist database data
- Access external storage systems

Important: A Pod volume is not automatically persistent. Persistence depends on the volume type and its lifecycle.

## 2. Container Filesystem vs Pod Volume

| Container Filesystem | Pod Volume |
|---|---|
| Writable layer belongs to a container | Defined at Pod level |
| Data can be lost when container is recreated | Lifecycle depends on volume type |
| Not automatically shared | Can be mounted into multiple containers |
| Suitable for disposable container data | Suitable for shared or persistent data |

Example: A container writes a file to `/tmp` in its own writable layer. That file may disappear after the container is recreated.

If it writes to an `emptyDir` volume, the file survives an ordinary container restart within the same Pod, but is lost when the Pod is removed.

## 3. Kubernetes Pod Storage Architecture

```
                  Kubernetes Pod
              ┌──────────────────────┐
              │                      │
              │  Container A         │
              │  /app/data           │
              │       │              │
              │       ▼              │
              │    Volume            │
              │       ▲              │
              │       │              │
              │  Container B         │
              │  /shared/data        │
              │                      │
              └──────────┬───────────┘
                         │
                 Volume source
                         │
              ┌──────────┴───────────┐
              │                      │
           emptyDir             PVC / PV
          Temporary            Persistent
```

The `volumes` field declares storage sources at the Pod level.

The `volumeMounts` field tells each container where to mount a volume.

## 4. Types of Pod Storage

Kubernetes supports several volume types.

| Volume Type | Main Purpose |
|---|---|
| emptyDir | Temporary storage shared within a Pod |
| hostPath | Mount a node filesystem path |
| persistentVolumeClaim | Use storage backed by a PersistentVolume |
| configMap | Mount non-sensitive configuration |
| secret | Mount sensitive configuration |
| projected | Combine supported sources into one directory |
| downwardAPI | Expose Pod metadata as files |
| csi | Use a Container Storage Interface driver |
| nfs | Mount an NFS share |

Production recommendation: For persistent application data, use a PVC with an appropriate StorageClass and CSI driver.

## 5. emptyDir Volume

An `emptyDir` volume is created when a Pod is assigned to a node.

It initially contains no files.

It can be shared by multiple containers in the same Pod.

Data survives an ordinary container crash or restart, but is deleted when the Pod is removed from the node.

### YAML Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-demo
spec:
  containers:
    - name: writer
      image: busybox
      command:
        - sh
        - -c
        - |
          while true; do
            date >> /data/output.log
            sleep 5
          done
      volumeMounts:
        - name: shared-storage
          mountPath: /data
    - name: reader
      image: busybox
      command:
        - sh
        - -c
        - |
          while true; do
            cat /shared/output.log 2>/dev/null
            sleep 5
          done
      volumeMounts:
        - name: shared-storage
          mountPath: /shared
  volumes:
    - name: shared-storage
      emptyDir: {}
```

### Commands

```bash
kubectl apply -f emptydir-demo.yaml
kubectl get pods
kubectl logs emptydir-demo -c reader
kubectl exec emptydir-demo -c writer -- ls /data
kubectl describe pod emptydir-demo
```

### Important points

- Both containers access the same underlying volume.
- Mount paths can be different.
- Data is temporary.
- `emptyDir.medium: Memory` uses a memory-backed filesystem.
- Memory-backed `emptyDir` usage counts toward memory accounting, so configure limits carefully.

### Real-time use cases

- Temporary cache
- File processing
- Sharing generated files between containers
- Sidecar log processing

## 6. hostPath Volume

A `hostPath` volume mounts a file or directory from the Kubernetes worker node into a Pod.

### YAML Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-demo
spec:
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - name: host-storage
          mountPath: /host-data
  volumes:
    - name: host-storage
      hostPath:
        path: /var/log
        type: Directory
```

### Important points

- Storage is tied to a particular node.
- Moving the Pod to another node may expose different data.
- Mounting host directories introduces security risks.
- Pod permissions and admission policies may prevent its use.

Production caution: Avoid general-purpose `hostPath` storage for application data. It is mainly useful for certain privileged system components and controlled node-level operations.

## 7. PersistentVolume (PV)

A PersistentVolume is a Kubernetes cluster storage resource.

It represents storage provisioned manually or dynamically through a storage system.

Examples include:

- AWS EBS
- AWS EFS
- Azure Disk
- Azure Files
- Google Persistent Disk
- NFS
- Storage provided through CSI drivers

A PV has its own lifecycle independent of an individual Pod.

## 8. PersistentVolumeClaim (PVC)

A PersistentVolumeClaim is a request for storage made by a workload.

A PVC specifies requirements such as:

- Storage capacity
- Access mode
- StorageClass
- Optional volume mode

The Kubernetes storage system binds the claim to a suitable PV.

### Storage Architecture

```
          Application Pod
                |
                v
               PVC
                |
                v
                PV
                |
                v
           CSI Driver
                |
                v
      External Storage System
         (EBS / EFS / etc.)
```

### Example PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ebs-sc
  resources:
    requests:
      storage: 5Gi
```

### Example Pod Using PVC

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pvc-demo
spec:
  containers:
    - name: app
      image: nginx:stable
      volumeMounts:
        - name: app-storage
          mountPath: /usr/share/nginx/html
  volumes:
    - name: app-storage
      persistentVolumeClaim:
        claimName: app-pvc
```

Important: The PVC must exist in the same namespace as the Pod. The `ebs-sc` StorageClass and its storage driver must also be configured in the cluster.

### Commands

```bash
kubectl apply -f pvc.yaml
kubectl get pvc
kubectl get pv
kubectl describe pvc app-pvc
kubectl apply -f pvc-pod.yaml
kubectl describe pod pvc-demo
```

## 9. StorageClass

A StorageClass defines how Kubernetes dynamically provisions storage.

It can specify:

- CSI provisioner
- Storage parameters
- Reclaim policy
- Volume binding mode
- Volume expansion support

### AWS EBS StorageClass Example

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

### Explanation

| Field | Meaning |
|---|---|
| provisioner | CSI driver responsible for provisioning |
| type: gp3 | AWS EBS volume type |
| reclaimPolicy: Delete | Dynamically created PV's backing storage is normally deleted after claim release |
| WaitForFirstConsumer | Delays binding/provisioning until a consuming Pod is scheduled |
| allowVolumeExpansion | Permits supported PVC size increases |

For Amazon EKS, the EBS CSI driver must be installed and configured with the necessary IAM permissions. The provisioner shown is for the standard EBS CSI driver; EKS Auto Mode uses a different provisioner.

## 10. Volume Access Modes

Access modes describe how a volume can be mounted.

| Access Mode | Meaning |
|---|---|
| ReadWriteOnce (RWO) | Read-write mounting by one node |
| ReadOnlyMany (ROX) | Read-only mounting by multiple nodes |
| ReadWriteMany (RWX) | Read-write mounting by multiple nodes |
| ReadWriteOncePod (RWOP) | Read-write mounting by only one Pod cluster-wide, for supported CSI volumes |

Important: ReadWriteOnce does not mean exactly one Pod. Multiple Pods on the same node may be able to use an RWO volume, depending on the storage driver and workload configuration.

### EBS vs EFS

- AWS EBS: Typically RWO block storage, subject to its supported attachment model.
- AWS EFS: Shared filesystem storage, commonly used for RWX workloads.

Access mode support depends on the storage backend and CSI driver.

## 11. Volume Reclaim Policies

A PersistentVolume can have a reclaim policy.

### Retain

The PV and underlying storage are retained for manual recovery or cleanup after the claim is deleted.

### Delete

The backing storage is normally deleted when the PV is released through the provisioner.

Production caution: Always check the PV reclaim policy before deleting a PVC containing important data.

```bash
kubectl get pv
kubectl describe pv <pv-name>
```

## 12. ConfigMap Volume

A ConfigMap can be mounted into a Pod as configuration files.

### Example

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  application.properties: |
    server.port=8080
    app.environment=development
```

### Pod Volume Configuration

```yaml
volumes:
  - name: config-volume
    configMap:
      name: app-config
```

### Container Mount

```yaml
volumeMounts:
  - name: config-volume
    mountPath: /etc/app-config
    readOnly: true
```

The file becomes available as:

```
/etc/app-config/application.properties
```

ConfigMap volumes are intended for configuration, not application data persistence.

## 13. Secret Volume

Secrets can be mounted as files.

### Example

```yaml
volumes:
  - name: secret-volume
    secret:
      secretName: app-secret
```

### Container Mount

```yaml
volumeMounts:
  - name: secret-volume
    mountPath: /etc/secrets
    readOnly: true
```

Secret volume files are mounted read-only.

For production security, enable appropriate RBAC, encryption at rest, and external secret-management integrations where needed.

## 14. projected Volume

A projected volume combines multiple supported sources into one directory.

Supported sources include ConfigMaps, Secrets, Downward API information, and service account tokens.

### Example

```yaml
volumes:
  - name: combined-config
    projected:
      sources:
        - configMap:
            name: app-config
        - secret:
            name: app-secret
```

Useful when an application expects configuration from multiple sources in one location.

## 15. downwardAPI Volume

The Downward API exposes selected Pod information to containers.

For example, a container can read its namespace or labels from files.

### Example

```yaml
volumes:
  - name: pod-info
    downwardAPI:
      items:
        - path: namespace
          fieldRef:
            fieldPath: metadata.namespace
```

The container can mount this volume and read the namespace value from the `namespace` file.

## 16. subPath and subPathExpr

`subPath` mounts a specific subdirectory or file from a volume.

### Example

```yaml
volumeMounts:
  - name: app-storage
    mountPath: /app/logs
    subPath: logs
```

This mounts the `logs` directory from the volume at `/app/logs`.

`subPathExpr` supports environment-variable expansion in the selected subpath.

Important: ConfigMap and Secret volumes mounted through `subPath` do not receive normal automatic updates.

## 17. Ephemeral Storage and Resource Limits

Kubernetes can track local ephemeral storage used by:

- Container writable layers
- Container logs
- Disk-backed `emptyDir` volumes

### Example

```yaml
resources:
  requests:
    ephemeral-storage: "1Gi"
  limits:
    ephemeral-storage: "2Gi"
```

If the Pod or container exceeds applicable ephemeral-storage limits, Kubernetes may evict the Pod.

Monitoring and enforcement depend on node filesystem configuration and kubelet support.

### Troubleshooting

```bash
kubectl describe pod <pod-name>
kubectl describe node <node-name>
kubectl get events --sort-by=.metadata.creationTimestamp
```

Look for messages related to DiskPressure, eviction, or ephemeral-storage usage.

## 18. Volume Mount Lifecycle

Typical sequence:

1. A Pod is created.
2. The scheduler selects a node.
3. Kubernetes coordinates volume binding and attachment as required.
4. The CSI driver or kubelet prepares the volume.
5. Kubelet mounts the volume for the Pod.
6. Containers start and access their mount paths.
7. When the Pod terminates, volumes are unmounted and detached as appropriate.

For persistent volumes, data retention depends on the PV, PVC, storage backend, and reclaim policy.

## 19. Kubernetes vs OpenShift Pod Storage

Both Kubernetes and OpenShift use the same fundamental storage resources:

- PersistentVolumes
- PersistentVolumeClaims
- StorageClasses
- CSI drivers
- Pod volumes

### Commands

| Kubernetes | OpenShift |
|---|---|
| `kubectl get pvc` | `oc get pvc` |
| `kubectl get pv` | `oc get pv` |
| `kubectl get sc` | `oc get sc` |
| `kubectl describe pvc app-pvc` | `oc describe pvc app-pvc` |
| `kubectl get pods` | `oc get pods` |

Real-time issue: A volume may mount successfully, but the application still receives Permission denied because the mounted directory's ownership or permissions are incompatible with the container's runtime UID.

## 20. Pod Storage Troubleshooting

### Scenario 1: PVC is Pending

Possible causes:

- StorageClass not found
- CSI driver unavailable
- Insufficient storage capacity
- Topology or availability-zone constraints
- Waiting for a consuming Pod due to `WaitForFirstConsumer`

Commands:

```bash
kubectl get pvc
kubectl describe pvc <pvc-name>
kubectl get storageclass
kubectl get events --sort-by=.metadata.creationTimestamp
```

### Scenario 2: Pod Stuck in ContainerCreating

Possible causes:

- Volume attachment failure
- Volume mount failure
- CSI driver issues
- Storage backend unavailable

Commands:

```bash
kubectl describe pod <pod-name>
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl get csidrivers
kubectl get csinodes
```

### Scenario 3: Multi-Attach Error

This can happen when a storage volume that does not support the requested attachment pattern is already attached to another node.

Solutions:

- Check the storage backend's attachment limitations.
- Review Pod placement.
- Confirm that old attachments are released.
- Use suitable RWX storage when shared multi-node access is required.

### Scenario 4: Permission Denied

Possible causes:

- Incorrect directory ownership
- Container runs as a non-root UID
- `fsGroup` or CSI driver ownership behavior
- Cluster security policy restrictions on the Pod's security context

Commands:

```bash
kubectl exec <pod-name> -- id
kubectl exec <pod-name> -- ls -ld /data
kubectl describe pod <pod-name>
```

### Scenario 5: Data Lost After Pod Recreation

Possible causes:

- Data stored in container writable layer
- `emptyDir` used instead of PVC
- PVC deleted
- PV reclaim policy caused backend deletion

Solution: Use persistent storage with appropriate retention and backup policies.

## 21. Hands-on Lab: Shared Storage Between Containers

### Step 1: Create the Pod

Save as `pod-storage-lab.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: storage-lab
spec:
  containers:
    - name: writer
      image: busybox
      command:
        - sh
        - -c
        - |
          echo "Hello from writer" > /data/message.txt
          sleep 3600
      volumeMounts:
        - name: shared
          mountPath: /data
    - name: reader
      image: busybox
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - name: shared
          mountPath: /shared
  volumes:
    - name: shared
      emptyDir: {}
```

### Step 2: Deploy

```bash
kubectl apply -f pod-storage-lab.yaml
```

### Step 3: Verify

```bash
kubectl get pod storage-lab
kubectl exec storage-lab -c writer -- cat /data/message.txt
kubectl exec storage-lab -c reader -- cat /shared/message.txt
```

Expected output:

```
Hello from writer
```

Both containers read the same file because they share the same `emptyDir` volume.

### Step 4: Cleanup

```bash
kubectl delete pod storage-lab
```

After deletion, the `emptyDir` data is no longer available.

## 22. Real-Time Interview Questions

**Q1. What is Pod Storage?**
Pod Storage is the storage used by containers in a Kubernetes Pod, including temporary, configuration, and persistent volumes.

**Q2. What is the difference between emptyDir and PVC?**
`emptyDir` is temporary Pod-scoped storage. A PVC requests storage from a PersistentVolume, which can preserve data beyond the lifetime of a Pod.

**Q3. Does data survive a container restart?**
Data in a mounted volume such as `emptyDir` generally survives an ordinary container restart within the same Pod. Data in the container's writable layer is not guaranteed to survive container recreation.

**Q4. What is the difference between PV and PVC?**
A PV represents cluster storage. A PVC is a request for storage that can bind to a suitable PV.

**Q5. What is a StorageClass?**
A StorageClass defines storage provisioning behavior, including the provisioner, parameters, and binding policy.

**Q6. What is ReadWriteOnce?**
It allows a volume to be mounted read-write by one node. It does not necessarily restrict usage to one Pod.

**Q7. Why is a PVC Pending?**
Possible reasons include a missing StorageClass, unavailable CSI driver, capacity problems, topology constraints, or delayed binding.

**Q8. What is a CSI driver?**
CSI stands for Container Storage Interface. It enables Kubernetes to integrate with storage systems through standardized interfaces.

**Q9. What is a Multi-Attach error?**
It indicates that the requested attachment conflicts with the storage backend's supported attachment configuration.

**Q10. Can multiple containers mount the same volume?**
Yes. Containers within a Pod can mount the same volume, subject to the volume's access and filesystem semantics.

**Q11. Can multiple Pods use one PVC?**
Sometimes. It depends on the access mode, storage driver, attachment model, and node placement.

**Q12. What happens if a PVC is deleted?**
The associated PV is released. The subsequent handling of backing storage depends on the PV reclaim policy and provisioner.

**Q13. What is the purpose of fsGroup?**
`fsGroup` can help control group ownership and access to supported mounted volumes, subject to driver and platform behavior.

**Q14. What is the difference between EBS and EFS?**
EBS provides block storage, commonly for single-node-attached workloads. EFS provides a shared network filesystem that supports access from multiple nodes.

**Q15. What storage issues are common in OpenShift?**
PVC provisioning problems, CSI driver issues, storage topology conflicts, and filesystem permission errors related to non-root containers and cluster security policy restrictions.

## 23. Quick Revision

| Topic | Remember |
|---|---|
| Container writable layer | Ephemeral |
| emptyDir | Temporary Pod storage |
| hostPath | Node filesystem access |
| PV | Cluster storage resource |
| PVC | Storage request |
| StorageClass | Provisioning configuration |
| CSI | Storage integration interface |
| RWO | Read-write from one node |
| RWX | Read-write from multiple nodes |
| RWOP | Single-Pod read-write access |
| ConfigMap volume | Non-sensitive configuration |
| Secret volume | Sensitive configuration |
| subPath | Mount a specific path within a volume |
| Ephemeral-storage limit | Controls supported local temporary storage usage |

Key takeaway: Use `emptyDir` for temporary Pod data, ConfigMap/Secret volumes for configuration, and PVC-backed storage for application data that must survive Pod recreation.
