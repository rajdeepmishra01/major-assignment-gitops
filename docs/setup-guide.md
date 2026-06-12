# Setup Guide — Todo Platform

> Complete step-by-step instructions to reproduce the full platform from scratch.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Clone the Repositories](#2-clone-the-repositories)
3. [Run Locally with Docker Compose](#3-run-locally-with-docker-compose)
4. [Create the k3d Cluster](#4-create-the-k3d-cluster)
5. [Install Required CLI Tools](#5-install-required-cli-tools)
6. [Configure GHCR Access](#6-configure-ghcr-access)
7. [Bootstrap FluxCD](#7-bootstrap-fluxcd)
8. [Verify Flux Installation](#8-verify-flux-installation)
9. [Install Monitoring Stack](#9-install-monitoring-stack)
10. [Install Logging Stack](#10-install-logging-stack)
11. [Apply Security and Policies](#11-apply-security-and-policies)
12. [Verify Full Deployment](#12-verify-full-deployment)
13. [Access the Application](#13-access-the-application)
14. [Access Grafana](#14-access-grafana)
15. [Trigger a New Deployment](#15-trigger-a-new-deployment)

---

## 1. Prerequisites

Ensure the following are installed on your machine before starting:

| Tool | Version | Install |
|---|---|---|
| Docker Desktop | Latest | https://www.docker.com/products/docker-desktop |
| k3d | v5.x | `curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh \| bash` |
| kubectl | Latest | https://kubernetes.io/docs/tasks/tools/ |
| Helm | v3.x | `curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 \| bash` |
| Flux CLI | Latest | `curl -s https://fluxcd.io/install.sh \| bash` |
| Node.js | v20+ | https://nodejs.org/ |

Verify all tools are available:

```bash
docker --version
k3d version
kubectl version --client
helm version
flux --version
node --version
```

You will also need:
- A GitHub account
- A GitHub Personal Access Token (PAT) with `repo` and `packages:read` scopes
- Access to the `rajdeepmishra01` GHCR namespace (or fork and adapt the repositories)

---

## 2. Clone the Repositories

```bash
mkdir major-assignment-devops
cd major-assignment-devops

# Application source code
git clone https://github.com/rajdeepmishra01/major-assignment-app.git

# GitOps configuration
git clone https://github.com/rajdeepmishra01/major-assignment-gitops.git
```

---

## 3. Run Locally with Docker Compose

This step validates the application before deploying to Kubernetes.

```bash
cd major-assignment-app

# Build and start all services
docker compose up --build

# Services available at:
# http://localhost:3000   → frontend
# http://localhost:3001   → auth-service
# http://localhost:3002   → todo-service
# localhost:5432          → PostgreSQL
```

Test basic functionality:

```bash
# Register a user
curl -X POST http://localhost:3001/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"testuser","password":"testpass123"}'

# Login
curl -X POST http://localhost:3001/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"testuser","password":"testpass123"}'
```

Shut down when done:

```bash
docker compose down
```

---

## 4. Create the k3d Cluster

```bash
# Delete existing cluster if present
k3d cluster delete todo-platform

# Create new cluster
k3d cluster create todo-platform \
  --servers 1 \
  --agents 2 \
  -p "8080:80@loadbalancer" \
  -p "8443:443@loadbalancer"

# Verify cluster is ready
kubectl get nodes -o wide
```

Expected output:
```
NAME                         STATUS   ROLES                  AGE
k3d-todo-platform-server-0   Ready    control-plane,master   ...
k3d-todo-platform-agent-0    Ready    <none>                 ...
k3d-todo-platform-agent-1    Ready    <none>                 ...
```

The cluster includes:
- Traefik ingress controller (pre-installed)
- `local-path` storage provisioner

---

## 5. Install Required CLI Tools

### cert-manager

cert-manager will be managed by Flux via the `security/` kustomization. However, if you need to install it manually:

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
kubectl wait --for=condition=available --timeout=120s deployment/cert-manager -n cert-manager
```

### VPA (Vertical Pod Autoscaler)

VPA CRDs and components are included in `advanced/crds/` and managed by Flux.

---

## 6. Configure GHCR Access

The cluster needs credentials to pull images from GitHub Container Registry.

```bash
# Create namespace first (Flux will also do this, but do it now for the secret)
kubectl create namespace todo-platform

# Create image pull secret
kubectl create secret docker-registry ghcr-credentials \
  --docker-server=ghcr.io \
  --docker-username=rajdeepmishra01 \
  --docker-password=<YOUR_GITHUB_PAT> \
  --docker-email=<your-email> \
  -n todo-platform

# Verify
kubectl get secret ghcr-credentials -n todo-platform
```

> **Note**: The `HelmRelease` and Flux image automation also need GHCR credentials. Flux uses the same PAT provided during bootstrap for image scanning.

---

## 7. Bootstrap FluxCD

Flux is bootstrapped once. After bootstrapping, it manages itself from the GitOps repository.

```bash
export GITHUB_TOKEN=<your-github-pat>
export GITHUB_USER=rajdeepmishra01

# Bootstrap Flux onto the cluster
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=major-assignment-gitops \
  --branch=main \
  --path=./clusters/local \
  --personal \
  --components-extra=image-reflector-controller,image-automation-controller

# This will:
# 1. Install Flux controllers in the flux-system namespace
# 2. Create a GitRepository pointing to major-assignment-gitops
# 3. Create a Kustomization to apply ./clusters/local
# 4. Push gotk-components.yaml and gotk-sync.yaml to the repo
```

After bootstrap, Flux will automatically:
- Apply the `todo-platform` Helm chart
- Set up monitoring (if included in the kustomization)
- Configure image automation

---

## 8. Verify Flux Installation

```bash
# Check all Flux controllers are running
flux check
kubectl get pods -n flux-system

# Check source synchronization
flux get sources git -A

# Check kustomizations
flux get kustomizations -A

# Check HelmRelease status
flux get helmreleases -A

# Check image automation
flux get image repository -A
flux get image policy -A
flux get image update -A
```

Expected HelmRelease status:
```
NAME            REVISION   SUSPENDED   READY   MESSAGE
todo-platform   0.1.10     False       True    Release reconciliation succeeded
```

---

## 9. Install Monitoring Stack

The monitoring stack is managed by Flux via `monitoring/` kustomization. To install manually:

```bash
# Add Prometheus community Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Create monitoring namespace
kubectl create namespace monitoring

# Install kube-prometheus-stack
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set grafana.adminPassword=prom-operator \
  --wait

# Apply custom ServiceMonitors
kubectl apply -f major-assignment-gitops/monitoring/prometheus/servicemonitor.yaml

# Apply Grafana dashboard ConfigMap (if applicable)
# kubectl apply -f major-assignment-gitops/monitoring/grafana/

# Verify
kubectl get pods -n monitoring
```

---

## 10. Install Logging Stack

Loki and Vector are managed by Flux via `monitoring/loki/` configuration. To install manually:

```bash
# Add Grafana Helm repo
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Create logging namespace
kubectl create namespace logging

# Install Loki
helm install loki grafana/loki \
  --namespace logging \
  --values major-assignment-gitops/monitoring/loki/values.yaml \
  --wait

# Install Vector
helm install vector vector/vector \
  --namespace logging \
  --values major-assignment-gitops/monitoring/loki/vector-values.yaml \
  --wait

# Verify
kubectl get pods -n logging
kubectl port-forward -n logging svc/loki 3100:3100 &
curl http://localhost:3100/ready
```

---

## 11. Apply Security and Policies

These are managed by Flux via `policies/` and `security/` kustomizations. To apply manually:

```bash
# cert-manager ClusterIssuer and Certificate
kubectl apply -f major-assignment-gitops/security/cert-manager/

# RBAC
kubectl apply -f major-assignment-gitops/policies/rbac/

# NetworkPolicy
kubectl apply -f major-assignment-gitops/policies/networkpolicy/

# ResourceQuota and LimitRange
kubectl apply -f major-assignment-gitops/policies/resourcequota/
kubectl apply -f major-assignment-gitops/policies/limitrange/

# PodDisruptionBudget
kubectl apply -f major-assignment-gitops/policies/pdb/

# Verify
kubectl get certificate -A
kubectl get networkpolicy -n todo-platform
kubectl get resourcequota -n todo-platform
kubectl get limitrange -n todo-platform
```

---

## 12. Verify Full Deployment

Run this checklist after everything is deployed:

```bash
# Cluster nodes
kubectl get nodes -o wide

# All namespaces
kubectl get ns

# All pods across all namespaces
kubectl get pods -A

# Application namespace
kubectl get pods -n todo-platform
kubectl get svc -n todo-platform
kubectl get ingress -n todo-platform

# TLS certificate
kubectl get certificate -A
kubectl describe certificate -n todo-platform

# Persistent storage
kubectl get pvc -n todo-platform
kubectl get pv

# Autoscaling
kubectl get hpa -n todo-platform
kubectl get vpa -n todo-platform

# Scheduled workloads
kubectl get cronjob -n todo-platform
kubectl get jobs -n todo-platform

# Security
kubectl get networkpolicy -n todo-platform
kubectl get resourcequota -n todo-platform
kubectl get limitrange -n todo-platform

# Flux status
flux get sources git -A
flux get kustomizations -A
flux get helmreleases -A
flux get image repository -A
flux get image policy -A
flux get image update -A

# Monitoring
kubectl get pods -n monitoring
kubectl get pods -n logging
```

All pods should show `Running` or `Completed` status. All Flux resources should show `READY: True`.

---

## 13. Access the Application

```bash
# Open in browser (self-signed cert — accept the browser warning)
open https://localhost:8443

# Or test with curl (skip TLS verification)
curl -k https://localhost:8443/

# Health checks
curl -k https://localhost:8443/api/auth/health
curl -k https://localhost:8443/api/todos/health
```

Register and login:

```bash
# Register
curl -k -X POST https://localhost:8443/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"demo","password":"demo123"}'

# Login — save the token
TOKEN=$(curl -k -s -X POST https://localhost:8443/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"demo","password":"demo123"}' | jq -r '.token')

echo "Token: $TOKEN"

# Create a todo
curl -k -X POST https://localhost:8443/api/todos \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"title":"My first Kubernetes todo"}'

# List todos
curl -k https://localhost:8443/api/todos \
  -H "Authorization: Bearer $TOKEN"
```

---

## 14. Access Grafana

```bash
# Port-forward Grafana
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80

# Open http://localhost:3000
# Username: admin
# Password: prom-operator
```

Add Loki datasource in Grafana (if not already configured):
1. Settings → Data Sources → Add data source → Loki
2. URL: `http://loki.logging:3100`
3. Save & Test

Import the custom dashboard:
1. Dashboards → Import
2. Upload `monitoring/grafana/dashboard.json`

---

## 15. Trigger a New Deployment

To simulate the full GitOps pipeline:

```bash
# Make a small code change in major-assignment-app
# Push to main
git -C major-assignment-app add .
git -C major-assignment-app commit -m "trigger: test deployment"
git -C major-assignment-app push origin main

# GitHub Actions will run → build new image with new tag
# Flux will detect new tag within 1 minute
# Check image automation
flux get image repository -A
flux get image policy -A
flux get image update -A

# Watch reconciliation
flux reconcile source git flux-system
flux get helmreleases -A

# Watch pod rollout
kubectl rollout status deployment/todo-platform-frontend -n todo-platform
kubectl rollout status deployment/todo-platform-auth-service -n todo-platform
kubectl rollout status deployment/todo-platform-todo-service -n todo-platform

# Verify new image tag is running
kubectl get pods -n todo-platform -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].image}{"\n"}{end}'
```

---

*See [gitops-flow.md](gitops-flow.md) for a detailed explanation of how Flux manages the GitOps reconciliation cycle.*
