# Security and Autoscaling — Todo Platform

> RBAC, Secrets, NetworkPolicy, TLS, HPA, VPA, and resource policies.

---

## Table of Contents

1. [Security Overview](#1-security-overview)
2. [JWT Authentication](#2-jwt-authentication)
3. [Kubernetes Secrets](#3-kubernetes-secrets)
4. [TLS with cert-manager](#4-tls-with-cert-manager)
5. [NetworkPolicy](#5-networkpolicy)
6. [RBAC](#6-rbac)
7. [ResourceQuota and LimitRange](#7-resourcequota-and-limitrange)
8. [PodDisruptionBudget](#8-poddisruptionbudget)
9. [CI/CD Security Scanning](#9-cicd-security-scanning)
10. [Horizontal Pod Autoscaler (HPA)](#10-horizontal-pod-autoscaler-hpa)
11. [Vertical Pod Autoscaler (VPA)](#11-vertical-pod-autoscaler-vpa)
12. [Backup CronJob](#12-backup-cronjob)
13. [Autoscaling Verification Commands](#13-autoscaling-verification-commands)
14. [Security Verification Commands](#14-security-verification-commands)

---

## 1. Security Overview

The platform implements security in multiple layers:

```
Layer 1: Application — JWT authentication, password hashing (bcrypt)
Layer 2: Transport  — TLS via cert-manager (Traefik terminates HTTPS)
Layer 3: Network    — NetworkPolicy rules restrict pod-to-pod traffic
Layer 4: Kubernetes — RBAC controls API access; Secrets store credentials
Layer 5: Resources  — ResourceQuota and LimitRange prevent resource abuse
Layer 6: CI/CD      — Trivy scans Docker images; SonarCloud scans code
```

---

## 2. JWT Authentication

Authentication is implemented at the application layer using JSON Web Tokens (JWT).

**Flow:**

```
1. User POSTs credentials to /api/auth/login
2. Auth service verifies password against bcrypt hash in PostgreSQL
3. Auth service issues a signed JWT (HS256, signed with JWT_SECRET)
4. Frontend stores the JWT token
5. Subsequent requests to /api/todos include: Authorization: Bearer <token>
6. Todo service middleware verifies the JWT signature
7. Decoded user ID is used to scope the query
```

**Environment variables involved:**
- `JWT_SECRET` — the signing key (stored in Kubernetes Secret)
- `DATABASE_URL` — PostgreSQL connection string (stored in Kubernetes Secret)

---

## 3. Kubernetes Secrets

Sensitive configuration is stored in Kubernetes Secrets, not in plaintext ConfigMaps.

The Secret is created by the Helm chart template `templates/secret.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: todo-platform-secret
  namespace: todo-platform
type: Opaque
data:
  DATABASE_URL: {{ .Values.postgres.databaseUrl | b64enc | quote }}
  JWT_SECRET: {{ .Values.jwtSecret | b64enc | quote }}
```

Default values from `values.yaml`:
```yaml
postgres:
  database: todoapp
  user: postgres
  password: postgres
jwtSecret: secretkey
```

> **Production note**: In a production environment, these values should be overridden with secure values via Sealed Secrets or an external secrets manager. The `security/sealed-secrets/` directory contains the Sealed Secrets controller configuration for this purpose.

**View secret (base64 encoded):**
```bash
kubectl get secret todo-platform-secret -n todo-platform -o yaml

# Decode a specific key
kubectl get secret todo-platform-secret -n todo-platform \
  -o jsonpath='{.data.JWT_SECRET}' | base64 -d
```

---

## 4. TLS with cert-manager

HTTPS is enabled using cert-manager with a self-signed `ClusterIssuer`.

**Components:**

| Resource | Location | Description |
|---|---|---|
| cert-manager | `cert-manager` namespace | Manages certificate lifecycle |
| `ClusterIssuer` | `security/cert-manager/` | Self-signed issuer (cluster-wide) |
| `Certificate` | `helm-charts/templates/` | TLS certificate for `localhost` |
| Secret (TLS) | `todo-platform` namespace | Stores certificate and key |

**Self-signed ClusterIssuer:**

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-cluster-issuer
spec:
  selfSigned: {}
```

**Certificate:**

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: todo-platform-tls
  namespace: todo-platform
spec:
  secretName: todo-platform-tls-secret
  issuerRef:
    name: selfsigned-cluster-issuer
    kind: ClusterIssuer
  dnsNames:
    - localhost
    - todo-platform.local
```

**Verify TLS:**
```bash
kubectl get certificate -A
kubectl describe certificate todo-platform-tls -n todo-platform

# Test HTTPS (accept self-signed cert)
curl -k https://localhost:8443/
```

The browser will show a security warning for the self-signed certificate — this is expected in a local development environment.

---

## 5. NetworkPolicy

NetworkPolicy manifests in `policies/networkpolicy/` restrict network traffic between pods.

**Default deny-all (ingress):**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: todo-platform
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```

**Allow ingress traffic only from Traefik:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-from-traefik
  namespace: todo-platform
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/part-of: todo-platform
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
```

**Allow postgres access only from app services:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-postgres-from-services
  namespace: todo-platform
spec:
  podSelector:
    matchLabels:
      app: postgres
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app.kubernetes.io/part-of: todo-platform
      ports:
        - port: 5432
```

```bash
kubectl get networkpolicy -n todo-platform
kubectl describe networkpolicy -n todo-platform
```

---

## 6. RBAC

Role-Based Access Control manifests are in `policies/rbac/`.

**Purpose**: Grant least-privilege API access to service accounts.

Example RBAC for the todo-platform namespace:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: todo-platform-reader
  namespace: todo-platform
rules:
  - apiGroups: [""]
    resources: ["pods", "services", "configmaps"]
    verbs: ["get", "list", "watch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: todo-platform-reader-binding
  namespace: todo-platform
subjects:
  - kind: ServiceAccount
    name: default
    namespace: todo-platform
roleRef:
  kind: Role
  name: todo-platform-reader
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl get role -n todo-platform
kubectl get rolebinding -n todo-platform
kubectl auth can-i get pods --namespace todo-platform --as system:serviceaccount:todo-platform:default
```

---

## 7. ResourceQuota and LimitRange

### ResourceQuota

Defined in `policies/resourcequota/`. Limits total resource consumption for the `todo-platform` namespace.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: todo-platform-quota
  namespace: todo-platform
spec:
  hard:
    requests.cpu: "4"
    requests.memory: "4Gi"
    limits.cpu: "8"
    limits.memory: "8Gi"
    pods: "20"
    services: "10"
    persistentvolumeclaims: "5"
```

### LimitRange

Defined in `policies/limitrange/`. Sets default CPU and memory requests/limits for pods that do not specify them.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: todo-platform-limits
  namespace: todo-platform
spec:
  limits:
    - type: Container
      default:
        cpu: "500m"
        memory: "512Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
      max:
        cpu: "2"
        memory: "2Gi"
```

```bash
kubectl get resourcequota -n todo-platform
kubectl describe resourcequota -n todo-platform

kubectl get limitrange -n todo-platform
kubectl describe limitrange -n todo-platform
```

---

## 8. PodDisruptionBudget

PodDisruptionBudgets (PDB) in `policies/pdb/` ensure that a minimum number of pods are available during voluntary disruptions (node maintenance, rolling updates).

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: frontend-pdb
  namespace: todo-platform
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: frontend
```

Three PDBs are defined — one for each of `frontend`, `auth-service`, and `todo-service`.

```bash
kubectl get pdb -n todo-platform
kubectl describe pdb -n todo-platform
```

---

## 9. CI/CD Security Scanning

### Trivy

Trivy performs vulnerability scanning on each Docker image after the build step in GitHub Actions.

```yaml
# In .github/workflows/pipeline.yml
- name: Trivy scan
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: ghcr.io/rajdeepmishra01/frontend:${{ github.run_number }}
    format: table
    exit-code: 0          # Report-only; does not fail the pipeline
    severity: CRITICAL,HIGH
```

`exit-code: 0` means Trivy reports vulnerabilities without blocking the pipeline. This is acceptable for a development environment and satisfies the assignment requirement to demonstrate security scanning.

### SonarCloud

SonarCloud performs static code analysis including:
- Code smells
- Bugs
- Security hotspots
- Duplicate code
- Coverage thresholds

Configuration: `major-assignment-app/sonar-project.properties`

---

## 10. Horizontal Pod Autoscaler (HPA)

HPA automatically scales the number of pod replicas based on observed CPU utilization.

### Configuration

Three HPAs are defined in `helm-charts/todo-platform/templates/hpa.yaml`:

```yaml
# Values from values.yaml
hpa:
  enabled: true
  frontend:
    minReplicas: 2
    maxReplicas: 6
    cpuUtilization: 50
  authService:
    minReplicas: 2
    maxReplicas: 6
    cpuUtilization: 50
  todoService:
    minReplicas: 2
    maxReplicas: 6
    cpuUtilization: 50
```

### Behavior

| Condition | Action |
|---|---|
| CPU > 50% sustained | Scale up (add replicas) |
| CPU < 50% for cooldown period | Scale down (remove replicas) |
| Replicas already at maxReplicas | No further scale-up |
| Replicas already at minReplicas | No further scale-down |

### Verification

```bash
kubectl get hpa -n todo-platform

# Detailed HPA status
kubectl describe hpa frontend-hpa -n todo-platform
kubectl describe hpa auth-service-hpa -n todo-platform
kubectl describe hpa todo-service-hpa -n todo-platform
```

Expected output:
```
NAME               REFERENCE                           TARGETS   MINPODS   MAXPODS   REPLICAS
frontend-hpa       Deployment/todo-platform-frontend   3%/50%    2         6         2
auth-service-hpa   Deployment/todo-platform-auth       1%/50%    2         6         2
todo-service-hpa   Deployment/todo-platform-todo       2%/50%    2         6         2
```

### Load test to trigger HPA

```bash
# Install hey or use kubectl run with stress
kubectl run load-test --image=busybox --rm -it --restart=Never -- \
  sh -c "while true; do wget -qO- https://todo-platform-frontend/; done"

# Watch HPA react
watch kubectl get hpa -n todo-platform
```

---

## 11. Vertical Pod Autoscaler (VPA)

VPA analyzes historical resource usage and provides recommendations for CPU and memory requests and limits. The `todo-service-vpa` is deployed in **Off mode** (recommendation only).

### Why Off / Recommendation Mode?

In `Off` mode, VPA:
- Monitors actual pod resource consumption
- Computes right-sized CPU and memory recommendations
- **Does not** automatically restart pods to apply recommendations

This is appropriate for the assignment because:
- It satisfies the requirement to deploy VPA
- It avoids unexpected pod restarts in the development environment
- It provides useful sizing data without disruption

### VPA Manifest

Located in `advanced/vpa/vpa.yaml`:

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: todo-service-vpa
  namespace: todo-platform
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: todo-platform-todo-service
  updatePolicy:
    updateMode: "Off"
  resourcePolicy:
    containerPolicies:
      - containerName: todo-service
        minAllowed:
          cpu: "50m"
          memory: "64Mi"
        maxAllowed:
          cpu: "1000m"
          memory: "1Gi"
```

### Viewing VPA Recommendations

```bash
kubectl get vpa -n todo-platform

kubectl describe vpa todo-service-vpa -n todo-platform
```

Example recommendation output:

```
Recommendation:
  Container Recommendations:
    Container Name: todo-service
    Lower Bound:
      Cpu:     25m
      Memory:  128Mi
    Target:
      Cpu:     50m
      Memory:  256Mi
    Uncapped Target:
      Cpu:     50m
      Memory:  256Mi
    Upper Bound:
      Cpu:     200m
      Memory:  512Mi
```

The **Target** values are the recommended requests to set for optimal resource allocation.

---

## 12. Backup CronJob

A `CronJob` in the `todo-platform` namespace simulates periodic PostgreSQL backups.

### Purpose

- Demonstrates Kubernetes `CronJob` functionality
- Simulates a real backup workflow using `pg_dump`
- Satisfies the assignment requirement for scheduled workloads

### CronJob Definition

Located in `advanced/cronjobs/postgres-backup-cronjob.yaml`:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
  namespace: todo-platform
spec:
  schedule: "0 */6 * * *"    # Every 6 hours
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: pg-backup
              image: postgres:16-alpine
              command:
                - /bin/sh
                - -c
                - |
                  echo "Starting backup at $(date)"
                  pg_dump $DATABASE_URL > /dev/null && echo "Backup completed successfully"
              env:
                - name: DATABASE_URL
                  valueFrom:
                    secretKeyRef:
                      name: todo-platform-secret
                      key: DATABASE_URL
```

### Verification

```bash
# Check CronJob schedule
kubectl get cronjob -n todo-platform

# Check completed jobs
kubectl get jobs -n todo-platform

# Check job logs (replace <job-name> with actual name)
kubectl get jobs -n todo-platform
kubectl logs job/<job-name> -n todo-platform

# Manually trigger a job from the CronJob
kubectl create job --from=cronjob/postgres-backup manual-backup-test -n todo-platform
kubectl logs job/manual-backup-test -n todo-platform
```

---

## 13. Autoscaling Verification Commands

```bash
# HPA status
kubectl get hpa -n todo-platform
kubectl describe hpa -n todo-platform

# VPA status and recommendations
kubectl get vpa -n todo-platform
kubectl describe vpa todo-service-vpa -n todo-platform

# Current pod resource usage (requires metrics-server)
kubectl top pods -n todo-platform
kubectl top nodes

# Check current replica counts
kubectl get deployments -n todo-platform
```

---

## 14. Security Verification Commands

```bash
# Secrets
kubectl get secret -n todo-platform
kubectl describe secret todo-platform-secret -n todo-platform

# TLS Certificate
kubectl get certificate -A
kubectl describe certificate todo-platform-tls -n todo-platform

# NetworkPolicy
kubectl get networkpolicy -n todo-platform
kubectl describe networkpolicy -n todo-platform

# RBAC
kubectl get role -n todo-platform
kubectl get rolebinding -n todo-platform
kubectl auth can-i list pods -n todo-platform

# ResourceQuota
kubectl get resourcequota -n todo-platform
kubectl describe resourcequota -n todo-platform

# LimitRange
kubectl get limitrange -n todo-platform
kubectl describe limitrange -n todo-platform

# PodDisruptionBudget
kubectl get pdb -n todo-platform
kubectl describe pdb -n todo-platform

# CronJob and Jobs
kubectl get cronjob -n todo-platform
kubectl get jobs -n todo-platform
```

---

*See [monitoring-logging.md](monitoring-logging.md) for observability details.*
*See [troubleshooting.md](troubleshooting.md) for common issues and fixes.*
