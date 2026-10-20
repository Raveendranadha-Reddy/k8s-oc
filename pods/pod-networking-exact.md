# Kubernetes Notes: Pod Networking

## 1. What Is Pod Networking?

Pod networking defines how Pods communicate with other Pods, Services, nodes, and external systems in a Kubernetes cluster.

Kubernetes relies on a network implementation, commonly provided by a Container Network Interface (CNI) plugin, to configure Pod networking.

Every Pod normally receives:

- A unique Pod IP address within the cluster's network.
- Its own network namespace.
- Network connectivity according to the cluster's CNI configuration and network policies.

Example: Your CoffeeShop frontend Pod sends an HTTP request to the backend Service, which forwards the request to a backend Pod.

## 2. Kubernetes Networking Architecture

```
                 Kubernetes Cluster
        +--------------------------------+
        |                                |
        |   Worker Node 1                |
        |   +------------------------+   |
        |   | Frontend Pod           |   |
        |   | IP: 10.244.1.10        |   |
        |   +-----------+------------+   |
        |               |                |
        |          Cluster Network       |
        |               |                |
        |   Worker Node 2                |
        |   +-----------+------------+   |
        |   | Backend Pod            |   |
        |   | IP: 10.244.2.20        |   |
        |   +------------------------+   |
        |                                |
        |   Backend Service              |
        |   ClusterIP: 10.96.10.20       |
        +--------------------------------+
```

Illustrative IP addresses only.

The frontend can reach the backend using the Service DNS name rather than relying on a backend Pod IP that may change.

## 3. Four Fundamental Kubernetes Networking Requirements

Kubernetes networking is commonly described using four fundamental requirements.

1. Every Pod receives its own cluster IP address.
2. Pods can communicate with other Pods across nodes without requiring application-level address translation, under the standard Kubernetes networking model.
3. Nodes can communicate with Pods, subject to routing and network configuration.
4. Containers within the same Pod share the Pod's network namespace and can communicate through localhost.

The actual implementation depends on the CNI plugin, cluster architecture, and network policies.

## 4. Pod IP Address

### Definition

A Pod IP is the IP address assigned to a Pod by the cluster networking implementation.

Example:

```bash
kubectl get pods -o wide
```

Output:

```
NAME             READY   STATUS    IP
frontend-pod     1/1     Running   10.244.1.10
backend-pod      1/1     Running   10.244.2.20
```

Inspect a specific Pod:

```bash
kubectl get pod frontend-pod \
  -o jsonpath='{.status.podIP}'
```

### Important points

- Pod IPs can change when a Pod is recreated.
- A Pod IP is not normally a permanent application endpoint.
- Use a Kubernetes Service for stable access to an application.
- Pod IP ranges are determined by the cluster network configuration.

## 5. Communication Between Containers in the Same Pod

Containers in the same Pod share the same network namespace and Pod IP.

They can communicate through `localhost` if the destination process is listening on the requested port.

Example:

```bash
curl http://localhost:8080
```

If one container runs an application on port 8080, another container in the same Pod can access it through that address.

Important: Containers sharing a network namespace must avoid port conflicts on the same IP address and protocol.

## 6. Communication Between Different Pods

Pods can communicate through their Pod IP addresses when cluster networking permits it.

Example:

```bash
curl http://10.244.2.20:8080
```

However, directly using a Pod IP is generally not recommended for application configuration because Pod IPs can change.

Use a Service instead:

```bash
curl http://coffee-backend:8080
```

Here, `coffee-backend` is the backend Service name, assuming it is resolvable from the calling Pod.

## 7. Kubernetes Services and Pod Networking

A Kubernetes Service provides a stable endpoint for accessing a set of Pods.

### Common Service types

| Service type | Purpose |
|---|---|
| ClusterIP | Internal cluster access; the default type |
| NodePort | Exposes a Service on a port of eligible nodes |
| LoadBalancer | Requests an external load balancer through the configured provider or implementation |
| ExternalName | Maps a Service name to an external DNS name |

### Example: Backend Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: coffee-backend
  namespace: coffeeshop
spec:
  type: ClusterIP
  selector:
    app: coffee-backend
  ports:
    - port: 8080
      targetPort: 8080
      protocol: TCP
```

Explanation:

- `port: 8080` is the Service port.
- `targetPort: 8080` is the destination port on selected Pods.
- `selector` identifies Pods that should receive traffic.
- `ClusterIP` provides an internal Service endpoint.

Apply and verify:

```bash
kubectl apply -f backend-service.yaml
kubectl get svc -n coffeeshop
kubectl get endpointslices -n coffeeshop
```

If the Service has no ready endpoints, check whether its selector matches the Pod labels and whether the Pods are ready.

## 8. Pod Networking and DNS

Kubernetes DNS allows applications to access Services by name instead of hardcoding IP addresses.

For example, from a Pod in the `coffeeshop` namespace:

```bash
curl http://coffee-backend:8080
```

A Service can also be addressed using its fully qualified DNS name:

```
coffee-backend.coffeeshop.svc.cluster.local
```

The usual Service DNS format is:

```
SERVICE_NAME.NAMESPACE.svc.cluster.local
```

The cluster domain may differ from `cluster.local` if the cluster was configured with another domain.

### DNS troubleshooting

```bash
kubectl exec -it <pod-name> -n coffeeshop -- \
  nslookup coffee-backend
```

If `nslookup` is not installed in the container, use a temporary debugging container or an appropriate diagnostic image.

## 9. Container Network Interface (CNI)

### Definition

CNI is a standard for integrating container runtimes with network implementations. Kubernetes uses its configured networking implementation to establish Pod connectivity.

A CNI plugin commonly handles tasks such as:

- Assigning Pod IP addresses.
- Configuring network interfaces.
- Establishing connectivity between Pods.
- Integrating with network policy enforcement, where supported.

### Examples of networking implementations

| Implementation | Typical context |
|---|---|
| Amazon VPC CNI | Amazon EKS |
| Calico | Kubernetes networking and network policy |
| Cilium | eBPF-based networking, observability, and policy |
| Flannel | Pod networking and overlay networks |

These implementations are not interchangeable in every environment. Capabilities depend on the plugin, version, and cluster configuration.

### Inspect networking information

```bash
kubectl get pods -o wide
kubectl get nodes -o wide
kubectl get daemonsets -A
```

For an EKS cluster, the Amazon VPC CNI commonly runs as the `aws-node` DaemonSet in the `kube-system` namespace.

```bash
kubectl get daemonset aws-node -n kube-system
kubectl get pods -n kube-system -o wide
```

## 10. Network Policies

A NetworkPolicy defines rules controlling permitted network traffic to or from selected Pods, provided the cluster's network implementation supports enforcement.

Example: Allow ingress traffic to backend Pods from frontend Pods in the same namespace.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: coffeeshop
spec:
  podSelector:
    matchLabels:
      app: coffee-backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: coffee-frontend
      ports:
        - protocol: TCP
          port: 8080
```

This policy selects backend Pods labelled `app: coffee-backend` and permits matching frontend Pods to connect on TCP port 8080.

Important: Once a Pod is selected by an ingress-isolating policy, traffic not allowed by the applicable policies is denied. Ensure required traffic, such as health checks and monitoring, is also permitted when needed.

Apply and inspect:

```bash
kubectl apply -f network-policy.yaml
kubectl get networkpolicy -n coffeeshop
kubectl describe networkpolicy allow-frontend-to-backend -n coffeeshop
```

## 11. Pod Networking in Amazon EKS

Amazon EKS commonly uses the Amazon VPC CNI plugin.

In the standard configuration, Pod IP addresses are allocated from the VPC network address space, subject to the configured networking mode and available IP resources.

Important considerations:

- VPC subnet IP availability.
- ENI and IP address limits.
- Security groups and routing.
- NetworkPolicy implementation and configuration.
- Pod density and node instance type.
- Optional prefix delegation or custom networking configuration.

Useful commands:

```bash
kubectl get pods -o wide
kubectl get nodes -o wide
kubectl get daemonset aws-node -n kube-system
kubectl logs -n kube-system -l k8s-app=aws-node
```

The last command retrieves logs from matching CNI Pods; inspect individual Pod logs if the selector returns multiple instances.

## 12. Pod Networking in OpenShift

OpenShift provides Pod connectivity, Service networking, and NetworkPolicy support through its configured networking stack. The networking configuration depends on the OpenShift version and installation.

Useful commands:

```bash
oc get pods -o wide
oc get nodes -o wide
oc get network.config.openshift.io cluster -o yaml
oc get networkpolicy -A
```

OpenShift clusters may also use additional networking capabilities, depending on the deployment and configuration.

## 13. Pod Networking Troubleshooting

### Scenario 1: Frontend cannot connect to backend

Check the backend Service:

```bash
kubectl get svc -n coffeeshop
kubectl describe svc coffee-backend -n coffeeshop
kubectl get endpointslices -n coffeeshop
```

Check:

- Service name and namespace.
- Service port and target port.
- Service selector and backend Pod labels.
- Backend Pod readiness.
- NetworkPolicy rules.

### Scenario 2: Pod IP connectivity fails

```bash
kubectl get pods -o wide
kubectl describe pod <pod-name>
kubectl get nodes -o wide
```

Investigate:

- CNI health.
- Node routing and firewall rules.
- NetworkPolicy enforcement.
- Cloud security groups or network ACLs, where applicable.
- Node and Pod network configuration.

### Scenario 3: DNS resolution fails

```bash
kubectl exec -it <pod-name> -n coffeeshop -- \
  nslookup coffee-backend
kubectl get svc -n coffeeshop
kubectl get pods -n kube-system
```

Check the Service name, namespace, Pod DNS configuration, and cluster DNS health.

### Scenario 4: Service exists but traffic does not reach the application

```bash
kubectl get svc coffee-backend -n coffeeshop -o yaml
kubectl get endpointslices -n coffeeshop
kubectl get pods -n coffeeshop --show-labels
```

A common cause is a mismatch between the Service selector and Pod labels. Another possibility is that the target application is not listening on the expected port.

## 14. Interview Questions and Answers

**Q1. What is Pod networking?**
Pod networking enables communication between Pods, nodes, Services, and external systems through the cluster's network implementation.

**Q2. Do containers in the same Pod have separate IP addresses?**
Normally, no. They share the Pod's network namespace and IP address.

**Q3. How do Pods communicate across worker nodes?**
Through the cluster networking implementation, which provides routing or overlay connectivity according to the configured network architecture.

**Q4. Why do we use a Service instead of a Pod IP?**
A Service provides a stable endpoint and can distribute traffic to matching backend Pods as they change.

**Q5. What is CNI?**
Container Network Interface is a standard for integrating container runtimes with network implementations.

**Q6. What is a NetworkPolicy?**
A NetworkPolicy defines permitted ingress and egress traffic for selected Pods when the network implementation supports policy enforcement.

**Q7. What is the difference between a Pod IP and a ClusterIP?**
A Pod IP identifies an individual Pod on the cluster network. A ClusterIP is a stable virtual IP assigned to a Service.

**Q8. How do you troubleshoot Service connectivity?**
Check DNS, Service configuration, endpoints, Pod labels, readiness, application ports, NetworkPolicies, and the CNI or cloud network configuration.

## 15. Quick Revision

- Every normal Pod gets its own IP address.
- Containers in the same Pod share a network namespace.
- Different Pods communicate through the cluster network.
- Kubernetes Services provide stable application endpoints.
- DNS enables Service discovery by name.
- CNI provides the underlying Pod networking integration.
- NetworkPolicies control traffic when supported by the networking implementation.
- EKS commonly uses Amazon VPC CNI.
- Troubleshooting should cover DNS, Services, endpoints, policies, routing, and CNI health.
