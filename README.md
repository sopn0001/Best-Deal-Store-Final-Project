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

## Local run (Docker images from Docker Hub)

Use **`docker-compose.hub.yml`** to pull **`idrissop/*`** images (override with `DOCKERHUB_USER`) and run MongoDB, RabbitMQ, all APIs, and both UIs on one machine.

```bash
cd "/path/to/Best-Deal-Store-Final-Project"
export DOCKERHUB_USER=idrissop
docker compose -f docker-compose.hub.yml pull
docker compose -f docker-compose.hub.yml up
```

- **Store Front:** http://localhost:8080  
- **Store Admin:** http://localhost:8081  
- **RabbitMQ management UI:** http://localhost:15672 (login `bestdeal` / `bestdeal`)

Optional sample data: Compose publishes Mongo on **`localhost:27018`** (so it does not fight another instance on 27017):

```bash
export MONGO_URL=mongodb://localhost:27018
cd "../Best-Deal-Category-and-Product-Service"
python seed_data.py --reset
```

Stop the stack: `Ctrl+C` then `docker compose -f docker-compose.hub.yml down` (add `-v` to remove the Mongo volume).

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

Create the **namespace first** (ConfigMap and Secret require `namespace: best-deal` to exist), then the rest:

```bash
kubectl apply -f "Deployment Files/00-namespace.yaml"
kubectl apply -f "Deployment Files/configmap.yaml"
kubectl apply -f "Deployment Files/secret.yaml"
kubectl apply -f "Deployment Files/all-manifests.yaml"
```

---

### 4. Verify Deployment

```bash
kubectl get all -n best-deal
```

Wait for all pods to reach `Running` status.

---

### 5. Open the websites

```bash
kubectl get svc -n best-deal
```

For `store-front` and `store-admin`, wait until **EXTERNAL-IP** is a public address (not `<pending>`). On AKS this can take **several minutes** the first time.

- **Store Front**: `http://<store-front-EXTERNAL-IP>`
- **Store Admin**: `http://<store-admin-EXTERNAL-IP>`

If **EXTERNAL-IP stays `<pending>`** for a long time, check quotas / permissions for public load balancers, or use a tunnel instead of the public IP:

```bash
kubectl port-forward -n best-deal svc/store-front 8080:80
```

Then open **http://localhost:8080** in your browser (same idea for admin: `svc/store-admin 8081:80`).

If the page loads but **products or orders fail**, check backend pods and logs:

```bash
kubectl get pods -n best-deal
kubectl logs -n best-deal deploy/category-product-service --tail=50
kubectl logs -n best-deal deploy/order-service --tail=50
kubectl describe pod -n best-deal -l app=store-front
```

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
