# Docker Setup

---

## Overview

The **Todo Platform** uses Docker to containerize all application components, ensuring consistency across development, CI/CD, and Kubernetes environments.

**Containerized Services:**

| Service       | Technology      |
| ------------- | --------------- |
| Frontend      | React + Vite    |
| Auth Service  | Node.js         |
| Todo Service  | Node.js         |
| PostgreSQL    | PostgreSQL      |

---

## Docker Architecture

```text
                Docker Network
                       |
    +------------------+------------------+
    |          |               |           |
    v          v               v           v
 Frontend  Auth Service  Todo Service  PostgreSQL
```


---

## Project Structure

Each application component contains its own Dockerfile.

```text
apps/
+-- frontend/
+-- auth-service/
+-- todo-service/
+-- database/
```

Docker Compose is used to orchestrate all containers during local development.

---

## Building & Running Containers

Build and start all services:

```bash
docker compose up --build
```

Verify running containers:

```bash
docker ps
```

**Expected containers:**

```text
todo-frontend
todo-auth-service
todo-todo-service
todo-postgres
```


---

## Container Networking

Containers communicate through a shared Docker network, enabling service discovery without exposing internal services externally.

```text
Frontend
    |
    +----> Auth Service
    |
    +----> Todo Service
                |
                v
           PostgreSQL
```

---

## Persistent Storage

PostgreSQL uses Docker volumes to persist data across container restarts.

Verify volumes:

```bash
docker volume ls
```


---

## Image Verification

Verify locally built images:

```bash
docker images
```

**Expected images:**

```text
frontend
auth-service
todo-service
postgres
```


---

## Container Health Validation

Verify service health:

```bash
curl http://localhost:5001/health
curl http://localhost:5002/health
```

**Expected Response:**

```json
{
  "status": "UP"
}
```


---

## Docker & GitOps Workflow

Docker images created during development are published to GitHub Container Registry (GHCR) and deployed through FluxCD.

```text
Application Code
       |
       v
Docker Image
       |
       v
GHCR
       |
       v
FluxCD
       |
       v
Kubernetes Cluster
```

---

## Validation Commands

```bash
docker ps
docker images
docker volume ls
```

---

## Deliverables

| Component              | Status |
| ---------------------- | ------ |
| Dockerfiles            | Yes    |
| Docker Compose         | Yes    |
| Frontend Container     | Yes    |
| Auth Service Container | Yes    |
| Todo Service Container | Yes    |
| PostgreSQL Container   | Yes    |
| Networking             | Yes    |
| Volumes                | Yes    |
| Image Build            | Yes    |

---

## Summary

Docker provides the foundation for application portability and deployment consistency across the Todo Platform. All application components are containerized and orchestrated through Docker Compose, forming the basis for CI/CD automation and Kubernetes-based GitOps deployments.