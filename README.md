# Deploying MongoDB with UI on Kubernetes

Kubernetes manifests for running MongoDB alongside [mongo-express](https://github.com/mongo-express/mongo-express) (a web-based MongoDB admin UI), wired together with a Secret, a ConfigMap, and Services.

## Components

| File | Kind(s) | Purpose |
|---|---|---|
| [mongo-secret.yaml](mongo-secret.yaml) | `Secret` | Stores the MongoDB root username/password (base64-encoded) as `mongodb-secret`. |
| [mongo.yaml](mongo.yaml) | `Deployment`, `Service` | Runs the `mongo` image, reading credentials from `mongodb-secret`. Exposes it internally as `mongodb-service` on port `27017`. |
| [mongo-configmap.yaml](mongo-configmap.yaml) | `ConfigMap` | Provides `database_url: mongodb-service:27017` so other pods can locate MongoDB by service name. |
| [mongo-express.yaml](mongo-express.yaml) | `Deployment`, `Service` | Runs `mongo-express`, using the same credentials from `mongodb-secret` and the connection info from `mongodb-configmap`. Exposed via a `LoadBalancer` Service (`mongo-express-service`) on port `8081`. |

## Architecture

```
mongodb-secret (Secret)
      │
      ├──> mongodb-deployment ──> mongodb-service (ClusterIP, :27017)
      │                                  │
      │                          mongodb-configmap (database_url)
      │                                  │
      └──> mongo-express-deployment ──> mongo-express-service (LoadBalancer, :8081)
```

## Prerequisites

- A running Kubernetes cluster (e.g. [Minikube](https://minikube.sigs.k8s.io/), Docker Desktop, kind, or a cloud provider's cluster)
- `kubectl` configured to talk to that cluster

## Deploy

Apply the manifests in order (Secret and ConfigMap first, since the Deployments reference them):

```bash
kubectl apply -f mongo-secret.yaml
kubectl apply -f mongo-configmap.yaml
kubectl apply -f mongo.yaml
kubectl apply -f mongo-express.yaml
```

Check that everything is up:

```bash
kubectl get pods
kubectl get svc
```

## Accessing mongo-express

`mongo-express-service` is of type `LoadBalancer` and requests `nodePort: 30000`.

- **Minikube**: run `minikube service mongo-express-service` to open it in your browser, or `minikube tunnel` to get an external IP.
- **Cloud provider**: `kubectl get svc mongo-express-service` will show an `EXTERNAL-IP` once provisioned; browse to `http://<EXTERNAL-IP>:8081`.
- **Any cluster (NodePort fallback)**: browse to `http://<node-ip>:30000`.

## Credentials

The default credentials in [mongo-secret.yaml](mongo-secret.yaml) decode to `root` / `root` (base64 for `root` is `cm9vdA==`). Replace these with your own base64-encoded values before using this outside of local/testing environments:

```bash
echo -n 'your-username' | base64
echo -n 'your-password' | base64
```

## Cleanup

```bash
kubectl delete -f mongo-express.yaml
kubectl delete -f mongo.yaml
kubectl delete -f mongo-configmap.yaml
kubectl delete -f mongo-secret.yaml
```
