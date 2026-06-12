# Helm

---

## Overview

The **Todo Platform** uses Helm to package, configure, and deploy Kubernetes resources.

Instead of managing individual YAML files, Helm templates generate all required resources using centralized configuration stored in `values.yaml`.

**Managed Resources:**

- Frontend, Auth Service, Todo Service, PostgreSQL
- Services, ConfigMaps, Secrets
- Ingress, PersistentVolumeClaims
- Horizontal Pod Autoscalers

---

## Helm Chart Structure

The application is packaged as a reusable Helm chart.

```text
helm-charts/
+-- todo-platform/
    +-- Chart.yaml
    +-- values.yaml
    +-- templates/
```


---

## Chart Configuration

### Chart Metadata

Chart information is defined in `Chart.yaml`:

```yaml
apiVersion: v2
name: todo-platform
version: 1.0.0
```

### Environment Configuration

Deployment settings are managed through `values.yaml`:

- Replica Counts
- Resource Limits
- Environment Variables
- Autoscaling Configuration


---

## Resources Managed by Helm

| Resource   | Purpose                 |
| ---------- | ----------------------- |
| Deployment | Application Workloads   |
| Service    | Internal Communication  |
| ConfigMap  | Configuration           |
| Secret     | Sensitive Data          |
| PVC        | Persistent Storage      |
| Ingress    | External Access         |
| HPA        | Autoscaling             |

Verify:

```bash
kubectl get deployments -n todo-platform
kubectl get svc -n todo-platform
kubectl get ingress -n todo-platform
kubectl get pvc -n todo-platform
```


---

## Helm Deployment

Install the application:

```bash
helm install todo-platform \
  ./helm-charts/todo-platform \
  -n todo-platform \
  --create-namespace
```

Verify:

```bash
helm list -A
```


---

## Upgrade & Rollback

**Upgrade:**

```bash
helm upgrade todo-platform \
  ./helm-charts/todo-platform \
  -n todo-platform
```

**Rollback:**

```bash
helm rollback todo-platform <revision>
```

**Verify history:**

```bash
helm history todo-platform -n todo-platform
```

---

## GitOps Integration

FluxCD uses HelmRelease resources to deploy the Helm chart automatically.

```text
Git Repository
      |
      v
    FluxCD
      |
      v
 HelmRelease
      |
      v
 Helm Chart
      |
      v
Kubernetes Resources
```

This enables automated and version-controlled deployments.


---

## Validation Commands

```bash
helm list -A
helm status todo-platform -n todo-platform
helm history todo-platform -n todo-platform
```

---

## Deliverables

| Component   | Status |
| ----------- | ------ |
| Chart.yaml  | Yes    |
| values.yaml | Yes    |
| Templates   | Yes    |
| Deployments | Yes    |
| Services    | Yes    |
| ConfigMaps  | Yes    |
| Secrets     | Yes    |
| PVC         | Yes    |
| Ingress     | Yes    |
| HPA         | Yes    |
| Install     | Yes    |
| Upgrade     | Yes    |
| Rollback    | Yes    |

---

## Summary

Helm provides a reusable and maintainable deployment mechanism for the Todo Platform. The Helm chart packages all Kubernetes resources into a single deployable unit and serves as the deployment foundation used by FluxCD for GitOps-based application delivery.