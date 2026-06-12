# Kubernetes Core

---

## Overview

The **Todo Platform** is deployed on a multi-node Kubernetes cluster created using k3d.

Kubernetes provides workload orchestration, service discovery, self-healing, networking, storage management, and resource governance for the platform.

**Implemented Resources:**

| Resource                   | Purpose                     |
| -------------------------- | --------------------------- |
| Namespace                  | Environment Isolation       |
| Deployment                 | Application Workloads       |
| Service                    | Internal Communication      |
| ConfigMap                  | Configuration Management    |
| Secret                     | Sensitive Data              |
| PersistentVolumeClaim (PVC)| Persistent Storage          |
| Ingress                    | External Access             |
| Resource Requests & Limits | Resource Governance         |

---

## Cluster Architecture

```text
k3d Cluster
|
+-- Control Plane
+-- Worker Node 1
+-- Worker Node 2
```

Verify:

```bash
kubectl get nodes -o wide
```


---

## Namespace Organization

Namespaces are used to isolate environments and platform components.

| Namespace               | Purpose                 |
| ----------------------- | ----------------------- |
| `todo-platform`         | Production workloads    |
| `todo-platform-dev`     | Development environment |
| `todo-platform-staging` | Staging environment     |
| `monitoring`            | Observability stack     |
| `flux-system`           | FluxCD controllers      |

Verify:

```bash
kubectl get namespaces
```


---

## Application Workloads

The platform consists of four Kubernetes deployments.

| Deployment     | Purpose          |
| -------------- | ---------------- |
| `frontend`     | User Interface   |
| `auth-service` | Authentication   |
| `todo-service` | Todo Management  |
| `postgres`     | Database         |

Verify:

```bash
kubectl get deployments -n todo-platform
kubectl get pods -n todo-platform
```

Kubernetes automatically recreates failed pods, providing self-healing capabilities.


---

## Services & Networking

ClusterIP services provide internal communication between workloads.

**Implemented Services:**

- `frontend`
- `auth-service`
- `todo-service`
- `postgres`

Verify services:

```bash
kubectl get svc -n todo-platform
```

Verify external access via Ingress:

```bash
kubectl get ingress -n todo-platform
```


---

## Configuration Management

Application configuration is managed using ConfigMaps and Secrets.

**Examples:**

- Environment Variables
- Database Credentials
- JWT Secrets

Verify:

```bash
kubectl get configmaps -n todo-platform
kubectl get secrets -n todo-platform
```


---

## Persistent Storage

PostgreSQL uses PersistentVolumeClaims (PVCs) to retain data across pod restarts and upgrades.

Verify:

```bash
kubectl get pvc -n todo-platform
```


---

## Resource Management

Resource requests and limits are configured for all workloads.

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

**Benefits:**

- Predictable Scheduling
- Resource Isolation
- Autoscaling Support


---

## Validation Commands

```bash
kubectl get deployments -n todo-platform
kubectl get pods -n todo-platform
kubectl get svc -n todo-platform
kubectl get ingress -n todo-platform
kubectl get pvc -n todo-platform
```

---

## Deliverables

| Component       | Status |
| --------------- | ------ |
| Cluster         | Yes    |
| Namespaces      | Yes    |
| Deployments     | Yes    |
| Pods            | Yes    |
| Services        | Yes    |
| ConfigMaps      | Yes    |
| Secrets         | Yes    |
| PVC             | Yes    |
| Ingress         | Yes    |
| Resource Limits | Yes    |
| Self-Healing    | Yes    |

---

## Summary

Kubernetes provides the foundation for the Todo Platform by managing application workloads, networking, configuration, secrets, storage, and resource allocation.

These core Kubernetes capabilities enable the platform to operate as a scalable, resilient, and production-style cloud-native application.