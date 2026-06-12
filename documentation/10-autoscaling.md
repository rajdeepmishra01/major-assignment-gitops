# Autoscaling

---

## Overview

The **Todo Platform** implements Kubernetes autoscaling to automatically adapt to changing workloads.

**Implemented Components:**

| Component               | Purpose                             |
| ----------------------- | ----------------------------------- |
| Horizontal Pod Autoscaler (HPA) | Scale pod replicas on CPU load |
| Vertical Pod Autoscaler (VPA)   | Recommend CPU/memory sizing    |
| Metrics Server          | Provides resource usage metrics     |

These components improve availability, scalability, and resource efficiency.

---

## Autoscaling Architecture

```text
Application Metrics
        |
        v
   Metrics Server
        |
   +----+----+
   |         |
   v         v
  HPA       VPA
   |         |
   +----+----+
        |
        v
 Application Workloads
```


---

## Horizontal Pod Autoscaler (HPA)

HPA automatically increases or decreases pod replicas based on CPU utilization.

**Configured resources:**

- `frontend-hpa`
- `auth-service-hpa`
- `todo-service-hpa`

**Example configuration:**

```yaml
minReplicas: 2
maxReplicas: 10
averageUtilization: 50
```

Verify:

```bash
kubectl get hpa -n todo-platform
```


---

## Vertical Pod Autoscaler (VPA)

VPA analyzes workload resource consumption and provides CPU and memory recommendations.

**Configured resource:** `todo-service-vpa`

**Current mode:**

```yaml
updateMode: Off
```

This allows safe evaluation of recommendations without automatically modifying running workloads.

Verify:

```bash
kubectl get vpa -A
```


---

## Metrics Server

Metrics Server provides the resource metrics required by both HPA and VPA.

Verify:

```bash
kubectl top pods -A
kubectl top nodes
```


---

## Validation Commands

```bash
kubectl get hpa -A
kubectl get vpa -A
kubectl top pods -A
```

---

## Deliverables

| Component                | Status |
| ------------------------ | ------ |
| Metrics Server           | Yes    |
| HPA                      | Yes    |
| frontend-hpa             | Yes    |
| auth-service-hpa         | Yes    |
| todo-service-hpa         | Yes    |
| VPA                      | Yes    |
| Resource Recommendations | Yes    |

---

## Summary

The Todo Platform uses Kubernetes HPA and VPA to improve scalability and resource utilization. HPA dynamically adjusts replica counts based on workload demand, while VPA provides resource recommendations based on observed application behavior.