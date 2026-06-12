# Monitoring and Observability

---

## Overview

The **Todo Platform** implements a complete observability stack for monitoring application performance, infrastructure health, and centralized logging.

**Monitoring Stack:**

| Component          | Purpose            |
| ------------------ | ------------------ |
| Prometheus         | Metrics Collection |
| Grafana            | Visualization      |
| ServiceMonitor     | Service Discovery  |
| Loki               | Log Storage        |
| Vector             | Log Collection     |
| Node Exporter      | Node Metrics       |
| Kube State Metrics | Kubernetes Metrics |

---

## Observability Architecture

```text
Application Metrics           Application Logs
        |                            |
        v                            v
  ServiceMonitor                  Vector
        |                            |
        v                            v
   Prometheus                       Loki
        |                            |
        +------------+---------------+
                     |
                     v
                  Grafana
```


---

## Monitoring Namespace

All monitoring components run inside the dedicated `monitoring` namespace.

Verify:

```bash
kubectl get pods -n monitoring
```


---

## Prometheus

Prometheus collects metrics from:

- Kubernetes Cluster
- Application Services
- Infrastructure Components

Service discovery is handled through ServiceMonitor resources.

Verify:

```bash
kubectl get servicemonitors -A
```


---

## Grafana Dashboards

Grafana provides dashboards for:

- Cluster Health
- CPU Usage
- Memory Usage
- Pod Status
- Application Metrics


---

## Centralized Logging

Logs are collected using Vector and stored in Loki.

```text
Application Pods
      |
      v
    Vector
      |
      v
     Loki
      |
      v
 Grafana Explore
```

**Benefits:**

- Centralized Log Management
- Simplified Troubleshooting
- Full Operational Visibility


---

## Validation Commands

```bash
kubectl get pods -n monitoring
kubectl get servicemonitors -A
kubectl get daemonsets -A
```

---

## Deliverables

| Component          | Status |
| ------------------ | ------ |
| Prometheus         | Yes    |
| Grafana            | Yes    |
| ServiceMonitor     | Yes    |
| Node Exporter      | Yes    |
| Kube State Metrics | Yes    |
| Loki               | Yes    |
| Vector             | Yes    |
| Dashboards         | Yes    |
| Log Aggregation    | Yes    |

---

## Summary

The Todo Platform implements a complete monitoring and logging solution using Prometheus, Grafana, Loki, and Vector.

The observability stack provides visibility into application performance, infrastructure health, resource utilization, and centralized logs, enabling efficient monitoring and troubleshooting of Kubernetes workloads.