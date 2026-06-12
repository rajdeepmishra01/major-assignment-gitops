# Troubleshooting Guide

---

## Overview

This document summarizes the major issues encountered during the implementation of the Todo Platform and the solutions used to resolve them.

---

## Docker Issues

### PostgreSQL Container Name Conflict

**Issue:**

Docker Compose failed to start PostgreSQL.

```text
Conflict. The container name "/todo-postgres" is already in use.
```

**Resolution:**

Remove the existing container and restart the stack.

```bash
docker rm -f todo-postgres
docker compose up --build
```


---

### PostgreSQL Image Pull Failure

**Issue:**

Docker failed to pull the PostgreSQL image.

```text
TLS handshake timeout
```

**Resolution:**

Pull the image manually and retry deployment.

```bash
docker pull postgres:15
```


---

## Kubernetes Issues

### k3d Cluster Creation Failure

**Issue:**

k3d cluster creation failed while downloading the proxy image.

```text
ghcr.io/k3d-io/k3d-proxy
TLS handshake timeout
```

**Resolution:**

```bash
docker logout ghcr.io
docker pull ghcr.io/k3d-io/k3d-proxy:5.8.3
k3d cluster create todo-platform
```


---

### Persistent Volume Claim Pending

**Issue:**

PostgreSQL pod could not start because the PVC remained in a `Pending` state.

**Resolution:**

Verify PVC and StorageClass configuration.

```bash
kubectl get pvc
kubectl get storageclass
```


---

## GitOps Issues

### Flux Reconciliation Delay

**Issue:**

Changes pushed to GitHub were not immediately reflected in the cluster.

**Resolution:**

Force reconciliation manually.

```bash
flux reconcile source git flux-system
flux reconcile kustomization flux-system
```

Verify status:

```bash
flux get all
```


---

### Image Automation Not Updating Tags

**Issue:**

Flux Image Automation did not update deployment manifests after a new image was pushed.

**Resolution:**

Verify ImageRepository, ImagePolicy, and ImageUpdateAutomation resources.

```bash
kubectl get imagerepositories -A
kubectl get imagepolicies -A
kubectl get imageupdateautomations -A
```


---

## Monitoring Issues

### Prometheus Targets Not Detected

**Issue:**

Application metrics were not visible in Prometheus.

**Root Cause:** ServiceMonitor configuration mismatch.

**Resolution:**

```bash
kubectl get servicemonitors -A
```

Verify all targets are marked **UP** in Prometheus.


---

### Grafana Dashboard Showing No Data

**Issue:**

Grafana dashboards displayed empty panels.

**Resolution:**

1. Verify Prometheus targets are active.
2. Validate Grafana datasource configuration.
3. Refresh dashboards after metrics collection.


---

## Autoscaling Issues

### HPA Metrics Unavailable

**Issue:**

HPA displayed `unknown` metrics.

**Resolution:**

Verify Metrics Server is running:

```bash
kubectl top pods -A
kubectl get pods -n kube-system
```


---

### VPA Recommendations Missing

**Issue:**

VPA showed no recommendations.

**Resolution:**

Generate application traffic and allow metrics collection time to run.

```bash
kubectl describe vpa
```


---

## CI/CD Issues

### SonarCloud Coverage Failure

**Issue:**

Coverage was not being detected correctly by SonarCloud.

**Resolution:**

Validate coverage report generation:

```bash
cat coverage/lcov.info
```

Ensure application files appear in the report.


---

## Useful Debugging Commands

### Cluster Health

```bash
kubectl get nodes
kubectl get pods -A
kubectl get events -A
```

### FluxCD Status

```bash
flux get all
```

### Monitoring Stack

```bash
kubectl get pods -n monitoring
```