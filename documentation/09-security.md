# Security

---

## Overview

The **Todo Platform** implements multiple Kubernetes security controls to protect workloads, network communication, sensitive data, and cluster resources.

**Implemented Controls:**

| Control             | Purpose                        |
| ------------------- | ------------------------------ |
| Kubernetes Secrets  | Sensitive Data Management      |
| RBAC                | Access Control                 |
| Network Policies    | Traffic Isolation              |
| TLS + cert-manager  | Encrypted Communication        |
| ResourceQuota       | Resource Governance            |
| LimitRange          | Container Resource Limits      |
| PriorityClass       | Workload Scheduling Priority   |
| PodDisruptionBudget | High Availability              |
| Trivy               | Container Image Scanning       |

---

## Security Architecture

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


---

## Access Control (RBAC)

Role-Based Access Control (RBAC) restricts access to Kubernetes resources using:

- Role
- RoleBinding
- ServiceAccount

Verify:

```bash
kubectl get roles,rolebindings -A
```


---

## Network Policies

Network Policies restrict communication between workloads and allow only required traffic flows.

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

Verify:

```bash
kubectl get networkpolicy -A
```


---

## Secrets Management

Sensitive configuration is stored using Kubernetes Secrets.

**Examples:**

- Database Credentials
- JWT Secret

Verify:

```bash
kubectl get secrets -n todo-platform
```


---

## TLS & cert-manager

TLS certificates are automatically managed using cert-manager.

**Resources:**

- ClusterIssuer
- Certificate
- TLS Secret

Verify:

```bash
kubectl get clusterissuers
kubectl get certificates -A
```


---

## Resource Governance

To prevent resource abuse, the platform implements:

| Control             | Purpose                      |
| ------------------- | ---------------------------- |
| ResourceQuota       | Namespace-level limits       |
| LimitRange          | Per-container defaults       |
| PriorityClass       | Workload scheduling priority |
| PodDisruptionBudget | Availability guarantees      |

Verify:

```bash
kubectl get resourcequota -A
kubectl get limitrange -A
kubectl get priorityclass
kubectl get pdb -A
```


---

## Container Security Scanning

Container images are scanned during CI/CD using Trivy.

**Benefits:**

- Vulnerability Detection
- Dependency Analysis
- Secure Image Delivery


---

## Deliverables

| Component           | Status |
| ------------------- | ------ |
| Secrets             | Yes    |
| RBAC                | Yes    |
| Network Policies    | Yes    |
| TLS                 | Yes    |
| cert-manager        | Yes    |
| ResourceQuota       | Yes    |
| LimitRange          | Yes    |
| PriorityClass       | Yes    |
| PodDisruptionBudget | Yes    |
| Trivy Scanning      | Yes    |

---

## Summary

The Todo Platform follows a defense-in-depth security model using Kubernetes-native controls for access management, workload isolation, secret protection, TLS encryption, resource governance, and container image security.