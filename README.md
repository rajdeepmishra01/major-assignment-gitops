# PERN Kubernetes DevOps Platform

A real working microservice-based Todo application built with:

- React frontend
- Node.js/Express Auth Service
- Node.js/Express Todo Service
- PostgreSQL database
- Docker Compose
- Kubernetes manifests
- Helm chart
- Flux GitOps skeleton
- Monitoring documentation

## Local Docker Run

```bash
docker compose up --build
```

Open:

```text
http://localhost:5173
```

Health checks:

```bash
curl http://localhost:5001/health
curl http://localhost:5002/health
```

## App Flow

1. Register user
2. Login user
3. JWT token is saved in browser localStorage
4. Create/update/delete todos
5. Todo Service validates JWT and stores todos per user

## Services

| Service | Port | Description |
|---|---:|---|
| frontend | 5173 | React UI |
| auth-service | 5001 | Register/login/JWT |
| todo-service | 5002 | Protected todo CRUD |
| postgres | 5432 | Database |

## Kubernetes Quick Start

Build images:

```bash
docker build -t auth-service:local ./apps/auth-service
docker build -t todo-service:local ./apps/todo-service
docker build -t frontend:local ./apps/frontend
```

For kind:

```bash
kind load docker-image auth-service:local
kind load docker-image todo-service:local
kind load docker-image frontend:local
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/secret.yaml
kubectl apply -f k8s/postgres/
kubectl apply -f k8s/auth-service/
kubectl apply -f k8s/todo-service/
kubectl apply -f k8s/frontend/
```

Port forward frontend:

```bash
kubectl -n todo-platform port-forward svc/frontend 5173:80
```

Open `http://localhost:5173`.
