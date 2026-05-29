# SEIP Assignment 1, 2026
### Kubernetes Echo API Deployment
## Prerequisites

| Tool | Version tested | Install |
|------|---------------|---------|
| Docker Desktop | 4.75+ | https://www.docker.com/products/docker-desktop |
| kubectl | 1.29+ | bundled with Docker Desktop |
| Git | any | https://git-scm.com |

> **Note:** This guide uses the Kubernetes cluster bundled with Docker Desktop. Minikube instructions are included as an alternative below.

---

## 1: Clone the Repository

```bash
git clone https://github.com/Chatzimanikas-Evaggelos/seip_assignment_1_2026.git
cd seip_assignment_1_2026
```

---

## 2a: Start the Cluster (Docker Desktop)

1. Open **Docker Desktop -> Settings -> Kubernetes**
2. Toggle **Enable Kubernetes** -> click **Apply & Restart**
3. Wait for the green Kubernetes indicator in the bottom left corner
4. Point `kubectl` at the new cluster:

```bash
kubectl config use-context docker-desktop
kubectl cluster-info   # should print a live https:// URL
```

---

## 2b: Start the Cluster (Minikube — alternative)

```bash
# Install Minikube (Windows, via winget)
winget install Kubernetes.minikube

# Start a single-node cluster
minikube start --driver=docker

# Verify
kubectl cluster-info
```

> All subsequent `kubectl` commands work identically regardless of which option one may have chosen.

---

## 3: Apply the Manifests

Apply in dependency order (ConfigMap and Secret must exist before the Deployment tries to reference them):

```bash
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/secret.yaml
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
```

Or apply the whole directory at once (kubectl resolves ordering automatically):

```bash
kubectl apply -f k8s/
```

Expected output:

```
configmap/echo-api-config created
secret/echo-api-secret created
deployment.apps/echo-api created
service/echo-api-service created
```

---

## 4: Verify the Deployment

```bash
# Watch pods reach Running state (takes ~30 seconds)
kubectl get pods -w

# Confirm all 3 replicas are ready
kubectl get deployment echo-api

# Inspect all created resources at once
kubectl get all
```

All three pods should show `STATUS: Running` and `READY: 1/1`.

---

## 5: Port Forwarding (Access the API Locally)

The Service is of type `ClusterIP`, meaning it is only reachable inside the cluster. Use `kubectl port-forward` to map it to your local machine:

```bash
kubectl port-forward service/echo-api-service 8080:80
```

This maps `localhost:8080` on your machine to port `80` on the `echo-api-service`, which in turn forwards to port `3000` on the pods.

Leave this command running in a dedicated terminal window.

---

## 6: Interact with the Endpoints

With port forwarding active, open a second terminal or your browser:

```bash
# Health check
curl http://localhost:8080/health

# Echo endpoint: returns the request body
curl -X POST http://localhost:8080/echo \
     -H "Content-Type: application/json" \
     -d '{"message": "hello"}'

# Root: shows the WELCOME_MESSAGE from the ConfigMap
curl http://localhost:8080/
```

Expected responses:

| Endpoint | Method | Expected Response |
|----------|--------|-------------------|
| `/health` | GET | `{ "status": "ok" }` |
| `/` | GET | `{ "message": "Welcome to the Software Engineering in Practice Assignment Cluster!" }` |
| `/echo` | POST | Mirrors the JSON body that was sent |

---

## 7: Inspect Configuration & Secrets

```bash
# View the ConfigMap values
kubectl describe configmap echo-api-config

# View the Secret (keys only, values are base64-encoded)
kubectl describe secret echo-api-secret

# Decode a secret value
kubectl get secret echo-api-secret -o jsonpath='{.data.API_SECRET_KEY}' | base64 --decode
```

---

## 8: Tear Down

```bash
kubectl delete -f k8s/
```

---

## Kubernetes Manifest Overview

### `configmap.yaml`
Stores non-sensitive runtime configuration as key-value pairs injected into the container as environment variables (`WELCOME_MESSAGE`, `NODE_ENV`).

### `secret.yaml`
Stores the `API_SECRET_KEY` as a Base64-encoded Opaque Secret. Base64 is an encoding, not encryption. In production this would be replaced with a secrets manager (e.g. HashiCorp Vault, AWS Secrets Manager).

### `deployment.yaml`
Declares a `Deployment` with:
- **3 replicas** for basic availability
- **Resource requests & limits** (`100m`/`128Mi` request, `250m`/`256Mi` limit) to prevent noisy-neighbour issues
- **Liveness probe** on `GET /health`. This restarts the container if it stops responding
- **Readiness probe** on `GET /health`. This removes the pod from the Service load balancer until it is ready to accept traffic

### `service.yaml`
Exposes the Deployment internally as a `ClusterIP` Service on port `80`, forwarding to the container's port `3000`. External access is achieved via `kubectl port-forward` (development) or an Ingress Controller (production).

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| `No connection could be made to localhost:8080` | kubectl not connected to a cluster | Run `kubectl config use-context docker-desktop` |
| Pods stuck in `ImagePullBackOff` | Image not publicly accessible on GHCR | Set package visibility to **Public** in GitHub -> Packages |
| Pods stuck in `CrashLoopBackOff` | App is crashing on start | Run `kubectl logs <pod-name>` for details |
| `error: no context exists with the name: docker-desktop` | Kubernetes not enabled in Docker Desktop | Docker Desktop -> Settings -> Kubernetes -> Enable |