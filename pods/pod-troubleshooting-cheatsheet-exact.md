# Kubernetes Pod Troubleshooting – Commands Cheat Sheet

## A. Basic inspection

```bash
kubectl get pods
kubectl get pods -o wide
kubectl get pods -A
kubectl get pods -n <namespace>
kubectl describe pod <pod-name> -n <namespace>
kubectl get events -n <namespace> --sort-by=.metadata.creationTimestamp
```

## B. Logs

```bash
# Current container logs
kubectl logs <pod-name> -n <namespace>

# Logs from the previous terminated container
kubectl logs <pod-name> -n <namespace> --previous

# Logs from all containers
kubectl logs <pod-name> -n <namespace> --all-containers=true

# Logs from a particular container
kubectl logs <pod-name> -c <container-name> -n <namespace>

# Follow logs
kubectl logs -f <pod-name> -n <namespace>
```

## C. Resources and nodes

```bash
kubectl top pods -A
kubectl top nodes
kubectl describe node <node-name>
kubectl get nodes
kubectl get pods -o wide
```

`kubectl top` requires a functioning metrics pipeline.

## D. Workload and storage inspection

```bash
kubectl get deployments,replicasets,statefulsets,daemonsets -n <namespace>
kubectl get pvc,pv -n <namespace>
kubectl get storageclass
kubectl get svc,endpointslices -n <namespace>
kubectl get networkpolicy -n <namespace>
```

## E. Execute commands inside a container

```bash
kubectl exec -it <pod-name> -n <namespace> -- sh
kubectl exec <pod-name> -n <namespace> -- cat /etc/resolv.conf
kubectl exec <pod-name> -n <namespace> -- printenv
```

The container must be running and contain the requested executable. Minimal images may not include `sh`, `curl`, `nslookup`, or other troubleshooting tools.

From <https://chatgpt.com/c/6ad4f14d-c854-83e8-92e2-ac525b2cd6a2>
