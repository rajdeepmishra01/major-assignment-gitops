# Jobs, CronJobs & CRDs

---

## Overview

Beyond Deployments and Services, Kubernetes provides resources for batch processing, scheduled tasks, and platform extensibility.

**Implemented Resources:**

| Resource                   | Purpose                        |
| -------------------------- | ------------------------------ |
| Job                        | One-time batch task execution  |
| CronJob                    | Scheduled recurring tasks      |
| Custom Resource Definition | Kubernetes API extensibility   |

These resources demonstrate operational automation and advanced Kubernetes capabilities.

---

## Architecture

```text
Kubernetes Cluster
        |
  +-----+------+------+
  |             |     |
  v             v     v
 Job        CronJob  CRD
  |             |     |
  v             v     v
Batch        Scheduled  Custom
Task           Task     Resource
```

---

## Jobs

A Kubernetes Job executes a task until completion.

### Implementation

A parallel Job was created to demonstrate batch workload execution.

```yaml
kind: Job
completions: 3
parallelism: 3
```

### Validation

```bash
kubectl get jobs -A
kubectl describe job <job-name>
```


---

## CronJobs

CronJobs execute workloads on a defined schedule.

### Implementation

The project includes a PostgreSQL backup CronJob.

**Purpose:**

- Automated database backup
- Scheduled execution
- Operational resilience

### Validation

```bash
kubectl get cronjobs -A
kubectl get jobs -A
```

Verify backup logs:

```bash
kubectl logs job/<backup-job>
```


---

## Custom Resource Definitions (CRDs)

CRDs extend the Kubernetes API with custom resource types.

### Implementation

A custom Backup CRD was created to demonstrate Kubernetes extensibility.

**Benefits:**

- API Extension
- Custom Resource Management
- Advanced Kubernetes Concepts

### Validation

List all CRDs:

```bash
kubectl get crd
```

List custom resources:

```bash
kubectl get <custom-resource>
```


---

## Deliverables

| Component          | Status |
| ------------------ | ------ |
| Job                | Yes    |
| Parallel Execution | Yes    |
| CronJob            | Yes    |
| PostgreSQL Backup  | Yes    |
| CRD                | Yes    |
| Custom Resource    | Yes    |

---

## Summary

The Todo Platform implements advanced Kubernetes workload management using Jobs, CronJobs, and Custom Resource Definitions. These resources enable batch processing, automated database backups, and Kubernetes API extensibility while demonstrating production-style operational automation.