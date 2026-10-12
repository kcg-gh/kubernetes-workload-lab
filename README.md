# Kubernetes Workload Lab

A local kind lab for practicing application deployment,
configuration, and troubleshooting in Kubernetes.

## Components

- Namespace: `workload-lab`
- nginx Deployment with two replicas
- ClusterIP Service
- HTTP readiness and liveness probes
- CPU and memory requests and limits
- ConfigMap mounted as a volume to serve `index.html`
- Dummy Secret consumed as an environment variable

## Prerequisites

- Docker running
- kind
- kubectl compatible with the cluster version

## Deployment

Create the cluster if it does not already exist:

```bash
kind create cluster --name platform-lab --wait 120s
```

Check the active context:

```bash
kubectl config current-context
```

It should be `kind-platform-lab`.

Apply the resources in order:

```bash
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/configmap.yaml -f k8s/secret.example.yaml
kubectl apply -f k8s/deployment.yaml -f k8s/service.yaml
kubectl rollout status deployment/frontend -n workload-lab --timeout=120s
```

Check the deployed resources:

```bash
kubectl get deployments,replicasets,pods,services -n workload-lab
```

## Local Access

Start port forwarding:

```bash
kubectl port-forward -n workload-lab service/frontend 8081:80
```

Open http://localhost:8081.

Keep the terminal running while accessing the application.
Press `Ctrl+C` to stop forwarding; the application remains running.

## Experiments Completed

### Pod Replacement

Deleting a Pod causes its ReplicaSet to create a replacement
to maintain the desired replica count.

### Readiness Probe Failure

Setting the readiness probe to a nonexistent path causes HTTP 404
responses. The new Pod remains `Running` but is not `Ready`.

Readiness failures do not restart the container. The rollout stalls
while the previous healthy replicas remain available.

### Restoring the Previous Pod Template

Restoring the previous readiness path returns the Deployment to
its earlier Pod template. Kubernetes reuses the matching ReplicaSet
and removes the unhealthy Pod.

### ConfigMap Updates

Updating the ConfigMap eventually updates the mounted HTML file
without recreating the Pods.

This works because the ConfigMap is mounted as a directory,
without `subPath`, and nginx reads the file when serving requests.

### Secret Updates

Updating the Secret does not change environment variables in
existing containers.

Replacing the Pods loads the updated value:

```bash
kubectl rollout restart deployment/frontend -n workload-lab
kubectl rollout status deployment/frontend -n workload-lab --timeout=120s
```

## Resource Configuration

Each nginx container uses:

| Resource | Request | Limit |
|---|---|---|
| CPU | `100m` | `500m` |
| Memory | `64Mi` | `128Mi` |

The scheduler uses requests when deciding where to place Pods.
Limits constrain container consumption.

CPU limits can cause throttling. Exceeding the memory limit can
result in an OOM termination.

These settings give the Pods the `Burstable` QoS class.

## Security

`secret.example.yaml` contains only a dummy credential for learning.

Do not commit real credentials, private keys, or kubeconfig files.
Base64 encoding in Kubernetes Secrets is not encryption.

## Cleanup

Delete the application and all resources in its namespace:

```bash
kubectl delete namespace workload-lab
```

Delete the entire local cluster:

```bash
kind delete cluster --name platform-lab
```
