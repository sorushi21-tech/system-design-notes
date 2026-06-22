# Kubernetes and Orchestration

Kubernetes is a container orchestration platform. Docker can run containers, but Kubernetes runs containers across many machines and keeps the system close to the desired state.

## 1. Why Kubernetes Exists

A single container is easy to run. A production system has harder problems:

- Run many containers across many machines.
- Restart failed containers automatically.
- Replace unhealthy instances.
- Scale up and down based on traffic.
- Route traffic to healthy instances.
- Deploy new versions without downtime.
- Manage configuration and secrets.
- Attach storage to stateful workloads.
- Control access and network communication.

Kubernetes solves these orchestration problems.

AWS note: managed Kubernetes on AWS is EKS. Use EKS when your team wants Kubernetes APIs and ecosystem tools on AWS. If you want simpler AWS-native container hosting without Kubernetes operations, compare EKS with ECS Fargate.

---

## 2. Kubernetes Mental Model

Kubernetes is declarative.

You declare desired state:

```text
Run 5 replicas of this application image.
Expose them through this service.
Restart unhealthy pods.
Roll out updates gradually.
```

Kubernetes continuously compares:

```text
desired state vs actual state
```

Then it acts to make actual state match desired state.

---

## 3. Cluster Architecture

A Kubernetes cluster has two main parts:

```text
Control plane
  -> Decides what should happen

Worker nodes
  -> Run application workloads
```

### Control Plane

Important components:

- API server: entry point for Kubernetes commands and controllers.
- etcd: stores cluster state.
- scheduler: chooses which node should run a pod.
- controller manager: runs controllers that reconcile state.

### Worker Node

Important components:

- kubelet: node agent that runs pods.
- container runtime: runs containers, usually containerd.
- kube-proxy or CNI components: handle networking.

On EKS, worker capacity usually comes from:

- Managed node groups
- Self-managed node groups
- Fargate profiles for selected pods

Most Java backend services on EKS run on managed node groups unless the team intentionally chooses Fargate for pod-level isolation and reduced node management.

---

## 4. Pod

A Pod is the smallest deployable unit in Kubernetes.

Usually:

```text
1 pod = 1 application container
```

Sometimes a pod has sidecars:

- Logging sidecar
- Service mesh proxy
- Configuration reloader
- Metrics exporter

Containers inside the same pod share:

- Network namespace
- `localhost`
- Volumes attached to the pod

Pods are disposable. Do not depend on a pod IP or pod filesystem lasting forever.

---

## 5. Deployment and ReplicaSet

### Deployment

A Deployment manages stateless application replicas.

It handles:

- Replica count
- Rolling updates
- Rollbacks
- Replacing failed pods
- Managing ReplicaSets

Use Deployments for:

- REST APIs
- Web applications
- Stateless workers
- Frontend services

### ReplicaSet

A ReplicaSet ensures the requested number of pod replicas exists.

Usually you do not create ReplicaSets directly. You create a Deployment, and the Deployment creates ReplicaSets.

---

## 6. Service

Pods come and go, so their IP addresses change. A Service provides a stable network endpoint for a group of pods.

Service selects pods using labels.

```text
Service -> matching pods
```

Types:

| Type | Purpose |
| --- | --- |
| ClusterIP | Internal cluster access only |
| NodePort | Exposes service on each node port |
| LoadBalancer | Creates cloud load balancer |
| ExternalName | Maps to external DNS name |

Most internal services use ClusterIP. Public services often use LoadBalancer or Ingress.

---

## 7. Ingress

Ingress manages HTTP/HTTPS traffic entering the cluster.

It can route by:

- Hostname
- Path
- TLS certificate

Example:

```text
api.example.com/users -> user-service
api.example.com/orders -> order-service
```

Ingress requires an Ingress Controller, such as:

- NGINX Ingress
- Traefik
- HAProxy
- AWS Load Balancer Controller

On EKS, the AWS Load Balancer Controller is commonly used to create ALBs or NLBs from Kubernetes Ingress and Service resources.

---

## 8. ConfigMap and Secret

### ConfigMap

Stores non-sensitive configuration.

Examples:

- Log level
- Feature toggle
- External service URL
- Environment name

### Secret

Stores sensitive configuration.

Examples:

- Password
- API key
- Token
- TLS certificate

Important: Kubernetes Secrets are base64 encoded by default. Use encryption at rest, RBAC, and preferably external secret managers in production.

On AWS, common production patterns are:

- AWS Secrets Manager with External Secrets Operator
- SSM Parameter Store with External Secrets Operator
- KMS encryption for Kubernetes secrets

---

## 9. Resource Requests and Limits

### Requests

A request is the amount of CPU/memory a container needs for scheduling.

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "512Mi"
```

Kubernetes uses requests to decide where to place pods.

### Limits

A limit is the maximum CPU/memory a container can use.

```yaml
resources:
  limits:
    cpu: "1"
    memory: "1Gi"
```

If memory exceeds the limit, the container can be killed. If CPU exceeds the limit, it is throttled.

Best practice:

- Set requests for all production workloads.
- Set memory limits carefully.
- Be cautious with CPU limits for latency-sensitive services.
- Observe real usage before final sizing.

---

## 10. Health Probes

### Liveness Probe

Answers: should Kubernetes restart this container?

Use when the process is alive but stuck.

### Readiness Probe

Answers: should this pod receive traffic?

Use when the app is starting, overloaded, or temporarily unable to serve.

### Startup Probe

Answers: has slow startup completed?

Use for applications that take a long time to initialize.

Rule of thumb:

- Liveness restarts.
- Readiness removes from traffic.
- Startup protects slow startup.

---

## 11. Workload Types

### Deployment

For stateless services.

Examples:

- APIs
- Web apps
- Stateless workers

### StatefulSet

For stateful systems needing stable identity and storage.

Examples:

- Kafka
- ZooKeeper
- Elasticsearch
- Self-managed databases

Cloud note: prefer managed databases unless you have a strong reason to self-host stateful systems in Kubernetes.

### DaemonSet

Runs one pod on every node.

Examples:

- Log agent
- Metrics agent
- Security agent
- Node networking component

### Job

Runs a task to completion.

Examples:

- One-time migration
- Batch import
- Data repair

### CronJob

Runs Jobs on a schedule.

Examples:

- Cleanup
- Report generation
- Periodic sync

---

## 12. Scaling

### Horizontal Pod Autoscaler

HPA changes the number of pod replicas.

Can scale on:

- CPU
- Memory
- Custom metrics
- External metrics such as queue depth

Good scaling metrics:

- Request rate per pod
- Queue depth for workers
- CPU for compute-heavy workloads
- Latency or concurrency for request services

### Vertical Pod Autoscaler

VPA recommends or changes CPU/memory requests.

Useful for right-sizing, but applying changes may restart pods.

### Cluster Autoscaler

Adds or removes worker nodes when pods cannot be scheduled or nodes are underused.

Relationship:

```text
HPA adds pods -> no room on nodes -> Cluster Autoscaler adds nodes
```

---

## 13. Deployments and Rollbacks

### Rolling Update

Kubernetes gradually replaces old pods with new pods.

Important settings:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0
    maxSurge: 1
```

- `maxUnavailable: 0` keeps existing capacity available.
- `maxSurge: 1` allows one extra pod during rollout.

### Rollback

```bash
kubectl rollout undo deployment/my-service
```

Rollback is safe only if database migrations and API contracts are backward compatible.

### PodDisruptionBudget

PDB protects availability during voluntary disruptions.

Example:

```yaml
minAvailable: 2
```

This means at least two pods must stay available during maintenance.

---

## 14. Security

### RBAC

RBAC controls who can do what.

Objects:

- Role
- ClusterRole
- RoleBinding
- ClusterRoleBinding

Principle: give users, CI systems, and service accounts only the permissions they need.

On EKS, use IRSA (IAM Roles for Service Accounts) so pods can access AWS services without static credentials.

Example:

```text
order-service pod
  -> Kubernetes service account
  -> IAM role through IRSA
  -> permission to read one SQS queue and one secret
```

### Network Policy

Network Policies restrict pod-to-pod traffic.

Example goal:

```text
frontend -> backend -> database
```

Everything else denied.

### Namespaces

Namespaces organize cluster resources.

Common patterns:

- By environment: dev, staging, prod
- By team
- By application

Namespaces are not a complete security boundary alone. Combine them with RBAC, Network Policies, quotas, and admission controls.

---

## 15. Helm

Helm packages Kubernetes YAML into reusable charts.

Chart structure:

```text
chart/
  Chart.yaml
  values.yaml
  templates/
```

Useful commands:

```bash
helm template my-app ./chart
helm install my-app ./chart
helm upgrade my-app ./chart
helm rollback my-app 1
```

Best practices:

- Keep templates readable.
- Put environment differences in values files.
- Use `helm template` before applying.
- Avoid hiding complex logic inside templates.

---

## 16. Common Kubernetes Failures

### CrashLoopBackOff

Container starts, crashes, and restarts repeatedly.

Check:

```bash
kubectl logs <pod>
kubectl describe pod <pod>
```

### ImagePullBackOff

Kubernetes cannot pull the image.

Common causes:

- Wrong image tag
- Registry authentication issue
- Image does not exist

### Pending Pod

Pod cannot be scheduled.

Common causes:

- Not enough CPU/memory
- Node selector mismatch
- Taints/tolerations issue
- Persistent volume unavailable

### Service Not Routing

Common causes:

- Label selector mismatch
- Readiness probe failing
- Wrong target port
- Network Policy blocking traffic

---

## 17. Production Checklist

- Requests and limits are configured.
- Readiness, liveness, and startup probes are configured.
- Graceful shutdown is implemented.
- Deployments use safe rolling update settings.
- PodDisruptionBudget exists for critical services.
- Secrets are managed securely.
- RBAC follows least privilege.
- Network Policies restrict traffic.
- Logs go to stdout/stderr.
- Metrics and traces are collected.
- Autoscaling uses meaningful metrics.
- Rollback procedure is tested.
- EKS workloads use IRSA instead of static AWS keys.
- Images are stored in ECR with immutable tags.
- ALB/NLB behavior is understood when using AWS Load Balancer Controller.
- Cluster, node, pod, and application logs are available in CloudWatch or the team's observability platform.
