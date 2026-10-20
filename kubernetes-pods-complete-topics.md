# ☸️ Kubernetes Pods
## Complete Topic Checklist | Kubernetes & OpenShift Learning
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
### 🟦 01. Pod Fundamentals
**━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━**

- **a)** What is a Pod?
- **b)** Why does Kubernetes use Pods?
- **c)** Pod vs Container
- **d)** Pod vs Deployment
- **e)** Single-container Pod
- **f)** Multi-container Pod
- **g)** Pod architecture and components
- **h)** Pod creation workflow
- **i)** Kubernetes API Server, Scheduler, Kubelet roles
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
### 🟩 02. Pod YAML and Configuration
**━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━**

- **a)** Pod YAML structure
- **b)** apiVersion, kind, metadata, spec
- **c)** labels and selectors
- **d)** container name and image
- **e)** command and args
- **f)** ports and environment variables
- **g)** workingDir
- **h)** imagePullPolicy
- **i)** restartPolicy
- **j)** `kubectl run, apply, edit, delete`
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
### 🟪 03. Pod Lifecycle
**━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━**

- **a)** Pending
- **b)** Running
- **c)** Succeeded
- **d)** Failed
- **e)** Unknown
- **f)** Container states: Waiting, Running, Terminated
- **g)** Pod phases vs container states
- **h)** RestartPolicy: Always, OnFailure, Never
- **i)** Pod termination and graceful shutdown
- **j)** terminationGracePeriodSeconds
- **k)** preStop and postStart lifecycle hooks
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
### 🟧 04. Pod Containers
**━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━**

- **a)** Multiple containers in one Pod
- **b)** Sidecar containers
- **c)** Init containers
- **d)** Native sidecar containers
- **e)** Ephemeral containers
- **f)** Shared network namespace
- **g)** Container-to-container communication
- **h)** Shared volumes between containers

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
### 🟥 05. Pod Health Checks
**━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━**

- **a)** Liveness probe
- **b)** Readiness probe
- **c)** Startup probe
- **d)** HTTP, TCP and exec probes
- **e)** initialDelaySeconds
- **f)** periodSeconds
- **g)** timeoutSeconds
- **h)** failureThreshold and successThreshold
- **i)** Probe troubleshooting
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
### 🟦 06. Pod Resources and Performance
**━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━**

- **a)** CPU requests and limits
- **b)** Memory requests and limits
- **c)** QoS classes: Guaranteed, Burstable, BestEffort
- **d)** OOMKilled
- **e)** CPU throttling
- **f)** ResourceQuota
- **g)** LimitRange
- **h)** Pod resource monitoring
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
### 🟩 07. Pod Networking
**━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━**

- **a)** Pod IP address
- **b)** Pod-to-Pod communication
- **c)** Pod-to-Service communication
- **d)** Container ports
- **e)** hostNetwork
- **f)** DNS resolution inside Pods
- **g)** CNI plugins
- **h)** NetworkPolicy
- **i)** Pod connectivity troubleshooting
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
### 🟪 08. Pod Storage
**━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━**

- **a)** emptyDir
- **b)** hostPath
- **c)** PersistentVolume and PersistentVolumeClaim
- **d)** StorageClass
- **e)** Volume mounts
- **f)** Mount propagation basics
- **g)** Read-only volumes
- **h)** Shared storage between containers
- **i)** Volume mount failures
- **j)** Data persistence after Pod deletion
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
### 🟧 09. Pod Scheduling
**━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━**
- **a)** NodeName
- **b)** NodeSelector
- **c)** Node affinity and anti-affinity
- **d)** Taints and tolerations
- **e)** Pod affinity and anti-affinity
- **f)** Topology spread constraints
- **g)** PriorityClass and preemption
- **h)** Pod scheduling failures
- **i)** Scheduler events and node resources
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
### 🟥 10. Pod Security
**━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━**

- **a)** SecurityContext
- **b)** runAsUser and runAsGroup
- **c)** runAsNonRoot
- **d)** allowPrivilegeEscalation
- **e)** privileged containers
- **f)** capabilities
- **g)** readOnlyRootFilesystem
- **h)** seccomp profiles
- **i)** ServiceAccount
- **j)** Pod Security Standards
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
### 🟦 11. Pod Configuration
**━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━**

- **a)** ConfigMap as environment variables
- **b)** ConfigMap as mounted files
- **c)** Secret as environment variables
- **d)** Secret as mounted files
- **e)** imagePullSecrets
- **f)** ServiceAccount token mounting
- **g)** Environment variable precedence
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
### 🟩 12. Pods and Controllers
**━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━**

- **a)** Standalone Pods
- **b)** ReplicaSet
- **c)** Deployment-managed Pods
- **d)** StatefulSet Pods
- **e)** DaemonSet Pods
- **f)** Job Pods
- **g)** CronJob Pods
- **h)** Static Pods
- **i)** PodDisruptionBudget
- **j)** Horizontal Pod Autoscaler
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
### 🟪 13. Pod Troubleshooting – Must Learn
**━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━**

- **a)** Pending
- **b)** ErrImagePull
- **c)** ImagePullBackOff
- **d)** CrashLoopBackOff
- **e)** CreateContainerConfigError
- **f)** CreateContainerError
- **g)** OOMKilled
- **h)** ContainerCreating for a long time
- **i)** Init container failure
- **j)** Readiness probe failure
- **k)** Liveness probe failure
- **l)** Pod stuck in Terminating
- **m)** FailedScheduling
- **n)** DNS failure
- **o)** Network connectivity failure
- **p)** PVC mount failure
- **q)** Permission denied
- **r)** Node NotReady and eviction issues
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
### 🟧 14. Advanced Pod Topics
**━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━**

- **a)** PodDisruptionBudget and voluntary disruptions
- **b)** Graceful application shutdown
- **c)** Pod eviction and node-pressure eviction
- **d)** Preemption
- **e)** Pod overhead
- **f)** RuntimeClass
- **g)** Downward API
- **h)** Projected volumes
- **i)** Pod readiness gates
- **j)** Scheduling gates
- **k)** Pod OS and architecture constraints
- **l)** Pod logs, events and audit troubleshooting
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
### 🟥 15. Practical Commands
**━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━**

- **a)** `kubectl get pods -o wide`
- **b)** `kubectl describe pod`
- **c)** `kubectl logs`
- **d)** `kubectl logs -c`
- **e)** `kubectl logs --previous`
- **f)** `kubectl exec -it`
- **g)** `kubectl debug`
- **h)** `kubectl get events`
- **i)** `kubectl get pod -o yaml`
- **j)** `kubectl top pod`
- **k)** `kubectl explain pod.spec`
- **l)** `kubectl delete pod`
- **m)** `kubectl get pods -w`

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**KUBERNETES PODS · COMPLETE TOPIC LIST**
