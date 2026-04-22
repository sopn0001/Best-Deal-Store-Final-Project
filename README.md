# Best Deal Store — Final Project

A simplified, microservices-based e-commerce web application deployed on **Azure Kubernetes Service (AKS)**.

---

## Architecture Diagram

![Best Deal Store Architecture](architecture.png)

### Data Flow Summary

| Flow | Path |
|------|------|
| Customer browses products | User → Store-Front → Category-and-Product-Service → MongoDB |
| Customer places order | User → Store-Front → Order-Service → RabbitMQ |
| Order fulfilment | RabbitMQ → Makeline-Service → MongoDB |
| Admin manages orders | Admin → Store-Admin → Makeline-Service → MongoDB |
| Admin manages products/categories | Admin → Store-Admin → Category-and-Product-Service → MongoDB |

---

## Application Overview

**Best Deal Store** is a cloud-native web store with two frontends and three backend microservices:

| Service | Tech | Role |
|---------|------|------|
| Store Front | React + Vite + nginx | Customer shopping portal — browse, cart, checkout |
| Store Admin | React + Vite + nginx | Employee portal — manage products, categories, orders |
| Order Service | Node.js + Express | REST API for order creation and status updates; publishes to RabbitMQ |
| Category & Product Service | Python + FastAPI | REST API for product and category CRUD |
| Makeline Service | Python (pika) | Background worker — consumes orders from RabbitMQ and updates status |
| MongoDB | Docker Hub `mongo:7` | Persistent document store (StatefulSet + PVC) |
| RabbitMQ | Docker Hub `rabbitmq:3-management` | Message broker between Order and Makeline services |

---

## Repository & Docker Hub Links

| Service | GitHub Repository | Docker Hub Image |
|---------|-------------------|-----------------|
| Store Front | [Best-Deal-Store-Front](https://github.com/sopn0001/Best-Deal-Store-Front) | [best-deal-store-front](https://hub.docker.com/r/sopn0001/best-deal-store-front) |
| Store Admin | [Best-Deal-Store-Admin](https://github.com/sopn0001/Best-Deal-Store-Admin) | [best-deal-store-admin](https://hub.docker.com/r/sopn0001/best-deal-store-admin) |
| Order Service | [Best-Deal-Order-Service](https://github.com/sopn0001/Best-Deal-Order-Service) | [best-deal-order-service](https://hub.docker.com/r/sopn0001/best-deal-order-service) |
| Category & Product Service | [Best-Deal-Category-and-Product-Service](https://github.com/sopn0001/Best-Deal-Category-and-Product-Service) | [best-deal-category-and-product-service](https://hub.docker.com/r/sopn0001/best-deal-category-and-product-service) |
| Makeline Service | [Best-Deal-Makeline-Service](https://github.com/sopn0001/Best-Deal-Makeline-Service) | [best-deal-makeline-service](https://hub.docker.com/r/sopn0001/best-deal-makeline-service) |
| Deployment Files | [Best-Deal-Store-Final-Project](https://github.com/sopn0001/Best-Deal-Store-Final-Project) | — |

---

## Deployment Instructions

### Prerequisites

- Azure CLI (`az`) and an active Azure subscription
- `kubectl` installed
- Docker Hub account

---

### 1. Create AKS Cluster

```bash
az group create --name best-deal-rg --location canadacentral

az aks create \
  --resource-group best-deal-rg \
  --name best-deal-aks \
  --node-count 2 \
  --enable-managed-identity \
  --generate-ssh-keys

az aks get-credentials --resource-group best-deal-rg --name best-deal-aks
```

---

### 2. Update Image Names

Replace `idrissop` in the manifest with your Docker Hub username if needed:

```bash
cd "Deployment Files"
sed -i '' 's/idrissop/YOUR_DOCKERHUB_USERNAME/g' all-manifests.yaml
# ConfigMap and Secret files do not contain image names; no change needed there for Docker Hub username.
```

---

### 3. Apply Manifests

Apply ConfigMap and Secret first, then the rest of the stack:

```bash
kubectl apply -f "Deployment Files/01-configmap.yaml"
kubectl apply -f "Deployment Files/02-secret.yaml"
kubectl apply -f "Deployment Files/all-manifests.yaml"
```

---

### 4. Verify Deployment

```bash
kubectl get all -n best-deal
```

Wait for all pods to reach `Running` status.

---

### 5. Get External IPs

```bash
kubectl get svc -n best-deal
```

Look for `EXTERNAL-IP` on the `store-front` and `store-admin` LoadBalancer services (may take 2–3 minutes to provision).

- **Store Front**: `http://<store-front-EXTERNAL-IP>`
- **Store Admin**: `http://<store-admin-EXTERNAL-IP>`

---

### CI/CD Setup (GitHub Actions)

For each service repository, add the following secrets in **GitHub → Settings → Secrets and Variables → Actions**:

| Secret | Description |
|--------|-------------|
| `DOCKERHUB_USERNAME` | Your Docker Hub username |
| `DOCKERHUB_TOKEN` | Docker Hub access token |
| `AZURE_CREDENTIALS` | JSON output of `az ad sp create-for-rbac` |
| `AKS_RESOURCE_GROUP` | e.g. `best-deal-rg` |
| `AKS_CLUSTER_NAME` | e.g. `best-deal-aks` |

Generate Azure credentials:

```bash
az ad sp create-for-rbac \
  --name best-deal-cicd \
  --role contributor \
  --scopes /subscriptions/<SUBSCRIPTION_ID>/resourceGroups/best-deal-rg \
  --json-auth
```

Pushing to `main` on any service repository will automatically build the Docker image, push it to Docker Hub, and roll out the new image to AKS.

---

### Cleanup

```bash
kubectl delete namespace best-deal
az group delete --name best-deal-rg --yes
```
