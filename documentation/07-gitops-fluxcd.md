# GitOps with FluxCD

---

## Overview

The **Todo Platform** follows GitOps principles for Kubernetes deployment and lifecycle management. Git serves as the single source of truth, while FluxCD continuously synchronizes the Kubernetes cluster with the desired state stored in the GitOps repository.

**Implemented Components:**

| Component            | Purpose                          |
| -------------------- | -------------------------------- |
| FluxCD               | GitOps Controller                |
| GitRepository        | Source Monitoring                |
| Kustomization        | Resource Orchestration           |
| HelmRelease          | Helm Chart Deployment            |
| ImageRepository      | Container Image Tracking         |
| ImagePolicy          | Image Version Policy             |
| ImageUpdateAutomation| Automated Image Tag Updates      |

**Benefits:**

- Automated Deployments
- Environment Consistency
- Continuous Reconciliation
- Version-Controlled Infrastructure
- Deployment Auditability

---

## GitOps Architecture

```text
Developer Commit
        |
        v
GitHub Repository
        |
        v
      FluxCD
        |
   +----+----+
   |         |
   v         v
Kustomize  HelmRelease
   |         |
   +----+----+
        |
        v
 Kubernetes Cluster
```


---

## Flux Installation

FluxCD is installed inside the cluster and runs within the `flux-system` namespace.

**Installed Controllers:**

- Source Controller
- Kustomize Controller
- Helm Controller
- Notification Controller

Verify:

```bash
kubectl get pods -n flux-system
```


---

## GitOps Repository Structure

The GitOps repository stores all Kubernetes deployment and infrastructure definitions.

```text
major-assignment-gitops/
+-- clusters/
+-- helm-charts/
+-- monitoring/
+-- security/
+-- policies/
+-- advanced/
```

This repository acts as the source of truth for the cluster.


---

## HelmRelease & Kustomization

Application deployments are managed using FluxCD HelmRelease and Kustomization resources.

### HelmRelease

Responsible for:

- Helm Chart Deployment
- Upgrades & Rollbacks
- Continuous Reconciliation

### Kustomization

Responsible for:

- Environment Management
- Resource Orchestration
- Continuous Synchronization

Verify:

```bash
flux get helmreleases -A
flux get kustomizations -A
```

**Configured environments:**

- `todo-platform-dev`
- `todo-platform-staging`


---

## Continuous Reconciliation

Flux continuously compares the desired state in Git against the actual cluster state.

```text
Git Desired State
       vs
Cluster Actual State
```

If configuration drift is detected, Flux automatically restores the cluster to match the Git-defined state.

Verify:

```bash
flux get all
```

---

## Image Automation

Flux Image Automation automatically updates deployment manifests when new container images are pushed to GitHub Container Registry (GHCR).

**Workflow:**

```text
GHCR
  |
  v
ImageRepository
  |
  v
ImagePolicy
  |
  v
ImageUpdateAutomation
  |
  v
Git Commit
  |
  v
Flux Reconciliation
```

Verify:

```bash
kubectl get imagerepositories -A
kubectl get imagepolicies -A
kubectl get imageupdateautomations -A
```


---

## End-to-End Deployment Flow

```text
Developer Push
      |
      v
GitHub Actions
      |
      v
Docker Build
      |
      v
GHCR
      |
      v
Flux Image Automation
      |
      v
GitOps Repository Update
      |
      v
Flux Reconciliation
      |
      v
Kubernetes Deployment
```

---

## Validation Commands

```bash
flux get all
flux get helmreleases -A
flux get kustomizations -A
kubectl get imagerepositories -A
kubectl get imagepolicies -A
kubectl get imageupdateautomations -A
```

---

## Deliverables

| Component               | Status |
| ----------------------- | ------ |
| FluxCD                  | Yes    |
| GitRepository           | Yes    |
| Kustomization           | Yes    |
| HelmRelease             | Yes    |
| Dev Environment         | Yes    |
| Staging Environment     | Yes    |
| ImageRepository         | Yes    |
| ImagePolicy             | Yes    |
| ImageUpdateAutomation   | Yes    |
| Continuous Reconciliation | Yes  |

---

## Summary

The Todo Platform implements GitOps using FluxCD for Kubernetes application delivery. Deployments are managed through HelmRelease and Kustomization resources, while Flux Image Automation automatically updates image versions from GHCR. This approach provides a fully automated, version-controlled, and auditable deployment workflow.