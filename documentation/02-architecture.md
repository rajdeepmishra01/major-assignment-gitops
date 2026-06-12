# Architecture

---

## Overview

The **Todo Platform** is a cloud-native microservices application deployed on Kubernetes and managed through GitOps using FluxCD.

The platform consists of:

| Layer          | Component            |
| -------------- | -------------------- |
| Frontend       | React Frontend       |
| Authentication | Auth Service         |
| Business Logic | Todo Service         |
| Database       | PostgreSQL           |
| Orchestration  | Kubernetes & Helm    |
| Automation     | FluxCD               |
| Observability  | Prometheus & Grafana |
| Logging        | Loki                 |
| CI/CD          | GitHub Actions       |

---

## High-Level Architecture

```text
                           Developer
                               |
                               v
                    GitHub (Application Repo)
                               |
                               v
                      GitHub Actions CI/CD
             (Lint -> Test -> Sonar -> Trivy -> Build)
                               |
                               v
                    GitHub Container Registry
                               |
                               v
                     Flux Image Automation
                               |
                               v
                    GitHub (GitOps Repository)
                               |
                               v
                             FluxCD
                               |
                               v

+----------------------------------------------------------+
|                Kubernetes Cluster (k3d)                  |
|                                                          |
|  Ingress                                                 |
|     |                                                    |
|     v                                                    |
|  Frontend (React)                                        |
|     |                                                    |
|     +---------> Auth Service ----------+                 |
|     |                                  |                 |
|     +---------> Todo Service ----------+--> PostgreSQL   |
|                                             (PVC)        |
|                                                          |
|  Monitoring Stack                                        |
|  Prometheus --> Grafana                                  |
|  Vector -----> Loki -----> Grafana                       |
|                                                          |
|  Security & Operations                                   |
|  RBAC | NetworkPolicy | TLS | HPA | VPA | CronJobs       |
+----------------------------------------------------------+
```

---

## Application Architecture

The application follows a microservices design.

| Service      | Technology            | Responsibility   |
| ------------ | --------------------- | ---------------- |
| Frontend     | React, Vite           | User Interface   |
| Auth Service | Node.js, Express, JWT | Authentication   |
| Todo Service | Node.js, Express      | Todo Management  |
| PostgreSQL   | PostgreSQL            | Data Persistence |

### Request Flow

```text
User
  |
  v
Ingress
  |
  v
Frontend
  |
  +----> Auth Service ----> PostgreSQL
  |
  +----> Todo Service ----> PostgreSQL
```


---

## Kubernetes Architecture

The platform runs on a multi-node k3d cluster.

```text
k3d Cluster
|
+-- Control Plane
+-- Worker Node 1
+-- Worker Node 2
```

**Namespaces:**

| Namespace               | Purpose                 |
| ----------------------- | ----------------------- |
| `todo-platform`         | Production workloads    |
| `todo-platform-dev`     | Development environment |
| `todo-platform-staging` | Staging environment     |
| `monitoring`            | Observability stack     |
| `flux-system`           | FluxCD controllers      |

**Implemented Resources:**

- Deployment, Service, Ingress
- ConfigMap, Secret, PVC
- HPA, NetworkPolicy


---

## GitOps Deployment Flow

All deployments are managed through FluxCD.

```text
Code Change
     |
     v
GitHub Actions
     |
     v
Docker Image
     |
     v
GHCR
     |
     v
Flux Image Automation
     |
     v
GitOps Repository
     |
     v
FluxCD
     |
     v
Kubernetes Cluster
```

---

## Observability Architecture

Monitoring and logging are implemented using Prometheus, Grafana, Loki, and Vector.

```text
Application Metrics        Application Logs
        |                          |
        v                          v
  ServiceMonitor               Vector
        |                          |
        v                          v
   Prometheus                    Loki
        |                          |
        +------------+-------------+
                     |
                     v
                  Grafana
```


---

## Security Architecture

Security is implemented through multiple Kubernetes-native controls.

| Control             | Purpose                   |
| ------------------- | ------------------------- |
| RBAC                | Access Management         |
| Network Policies    | Traffic Isolation         |
| Kubernetes Secrets  | Sensitive Data Management |
| TLS Certificates    | Encrypted Communication   |
| ResourceQuota       | Resource Governance       |
| LimitRange          | Container Resource Limits |
| PodDisruptionBudget | High Availability         |

```text
User
  |
 HTTPS
  |
 Ingress
  |
 Network Policies
  |
 Application Services
  |
 Kubernetes Secrets
  |
 PostgreSQL
```