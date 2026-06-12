# CI/CD Pipeline

---

## Overview

The **Todo Platform** uses GitHub Actions to automate code validation, testing, security scanning, container image creation, and GitOps-based deployments.

The pipeline ensures that every change is validated before being released to Kubernetes.

**Technologies:**

| Tool                         | Purpose                  |
| ---------------------------- | ------------------------ |
| GitHub Actions               | Pipeline Orchestration   |
| Vitest                       | Unit Testing             |
| SonarCloud                   | Code Quality Analysis    |
| Trivy                        | Security Scanning        |
| Docker                       | Container Build          |
| GitHub Container Registry    | Image Storage            |
| FluxCD Image Automation      | GitOps Deployment        |

---

## Pipeline Architecture

```text
Developer Push
      |
      v
GitHub Actions
      |
      +----> Lint
      |
      +----> Test + Coverage
      |
      +----> SonarCloud Analysis
      |
      +----> Trivy Security Scan
      |
      +----> Docker Build
      |
      +----> Push Images to GHCR
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

## Workflow Trigger

The pipeline runs automatically on:

- Push to `main`
- Push to `develop`
- Pull Requests

**Workflow location:**

```text
.github/workflows/pipeline.yml
```


---

## Pipeline Stages

### Stage 1 — Linting

Code quality validation using ESLint.

**Purpose:**

- Enforce coding standards
- Detect syntax issues early


---

### Stage 2 — Testing & Coverage

Automated tests run for:

- Auth Service
- Todo Service

Coverage reports are generated during execution.

**Current Coverage:** `90%+`


---

### Stage 3 — SonarCloud Analysis

SonarCloud performs:

- Code Quality Analysis
- Security Analysis
- Coverage Validation

The Quality Gate must pass before the pipeline proceeds.


---

### Stage 4 — Trivy Security Scan

Container images are scanned for vulnerabilities before publishing.

**Benefits:**

- Early vulnerability detection
- Secure image delivery


---

### Stage 5 — Docker Build & Publish

Docker images are built for:

- `frontend`
- `auth-service`
- `todo-service`

Images are pushed to GitHub Container Registry.

```text
ghcr.io/<owner>/frontend:<tag>
ghcr.io/<owner>/auth-service:<tag>
ghcr.io/<owner>/todo-service:<tag>
```


---

## GitOps Integration

The CI pipeline does not deploy directly to Kubernetes. Instead:

```text
GitHub Actions
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

This approach ensures deployments remain fully GitOps-compliant.


---

## Validation

Verify successful workflow execution:

- GitHub Actions workflow completed successfully
- SonarCloud Quality Gate passed
- Trivy scan completed
- Images available in GHCR
- Flux detected and deployed updated image


---

## Deliverables

| Component          | Status |
| ------------------ | ------ |
| GitHub Actions     | Yes    |
| Linting            | Yes    |
| Automated Testing  | Yes    |
| Coverage Reporting | Yes    |
| SonarCloud         | Yes    |
| Trivy Scanning     | Yes    |
| Docker Build       | Yes    |
| GHCR Push          | Yes    |
| FluxCD Integration | Yes    |

---

## Summary

The Todo Platform implements an automated CI/CD workflow using GitHub Actions, SonarCloud, Trivy, Docker, GHCR, and FluxCD.

Every code change is validated, tested, scanned, containerized, and delivered through a GitOps workflow, providing a secure and reliable deployment process.