# Monitoring and Logging — Todo Platform

> Observability stack: Prometheus, Grafana, Loki, and Vector.

---

## Table of Contents

1. [Observability Overview](#1-observability-overview)
2. [Prometheus Setup](#2-prometheus-setup)
3. [ServiceMonitor Configuration](#3-servicemonitor-configuration)
4. [Grafana Dashboard](#4-grafana-dashboard)
5. [Dashboard Panel Reference](#5-dashboard-panel-reference)
6. [Useful PromQL Queries](#6-useful-promql-queries)
7. [Prometheus Alerts](#7-prometheus-alerts)
8. [Loki Setup](#8-loki-setup)
9. [Vector Log Collector](#9-vector-log-collector)
10. [Viewing Logs in Grafana](#10-viewing-logs-in-grafana)
11. [Access Commands](#11-access-commands)
12. [Troubleshooting Observability](#12-troubleshooting-observability)

---

## 1. Observability Overview

The observability stack provides:

| Layer | Tool | Namespace | Purpose |
|---|---|---|---|
| Metrics collection | Prometheus | `monitoring` | Scrapes pod and node metrics |
| Metrics visualization | Grafana | `monitoring` | Dashboards and alerting |
| Infrastructure metrics | kube-state-metrics | `monitoring` | Kubernetes object metrics |
| Node metrics | node-exporter | `monitoring` | Host-level CPU/memory/disk |
| Log collection | Vector DaemonSet | `logging` | Collects pod logs from all nodes |
| Log storage | Loki | `logging` | Log aggregation and querying |

All components are installed via Helm and managed by Flux.

---

## 2. Prometheus Setup

Prometheus is installed as part of the `kube-prometheus-stack` Helm chart in the `monitoring` namespace.

```bash
# Verify Prometheus is running
kubectl get pods -n monitoring -l app.kubernetes.io/name=prometheus

# Access Prometheus UI
kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-prometheus 9090:9090
# Open http://localhost:9090
```

**What Prometheus scrapes by default** (from kube-prometheus-stack):
- Kubernetes API server
- kubelet / cAdvisor (pod CPU and memory)
- kube-state-metrics (deployments, pods, HPAs, etc.)
- node-exporter (node CPU, memory, disk, network)

**Custom scrape targets** added via ServiceMonitor:
- `auth-service:3001/metrics`
- `todo-service:3002/metrics`

---

## 3. ServiceMonitor Configuration

The file `monitoring/prometheus/servicemonitor.yaml` configures Prometheus to scrape the application services.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: todo-platform
  namespace: monitoring
  labels:
    release: monitoring   # Must match kube-prometheus-stack release name
spec:
  namespaceSelector:
    matchNames:
      - todo-platform
  selector:
    matchLabels:
      app.kubernetes.io/part-of: todo-platform   # Must match service labels
  endpoints:
    - port: http
      path: /metrics
      interval: 15s
```

> **Critical**: The `selector.matchLabels` in the ServiceMonitor must match the labels on the Kubernetes `Service` resources. A mismatch causes Prometheus to show "0 targets" for those endpoints.

**Verify scrape targets are active:**
```bash
kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-prometheus 9090:9090 &
# Navigate to http://localhost:9090/targets
# Look for todo-platform/todo-platform-0 and todo-platform/todo-platform-1
```

---

## 4. Grafana Dashboard

The custom dashboard is stored at `monitoring/grafana/dashboard.json`.

**Access Grafana:**
```bash
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80
# Open http://localhost:3000
# Username: admin
# Password: prom-operator
```

**Import the dashboard:**
1. Grafana → Dashboards → Import
2. Upload `monitoring/grafana/dashboard.json`
3. Select Prometheus datasource
4. Click Import

**Add Loki datasource (if not present):**
1. Configuration → Data Sources → Add
2. Type: Loki
3. URL: `http://loki.logging:3100`
4. Save & Test

---

## 5. Dashboard Panel Reference

The Grafana dashboard is organized into the following row groups:

### Service Health

| Panel | PromQL | Description |
|---|---|---|
| Frontend Running | `kube_deployment_status_replicas_ready{deployment="todo-platform-frontend"}` | Ready replica count |
| Auth Service Up | `up{job="todo-platform-auth-service"}` | 1 = up, 0 = down |
| Todo Service Up | `up{job="todo-platform-todo-service"}` | 1 = up, 0 = down |
| Postgres Running | `kube_deployment_status_replicas_ready{deployment="todo-platform-postgres"}` | Ready replica count |

### Request Metrics

| Panel | PromQL | Description |
|---|---|---|
| Auth Request Rate | `rate(http_requests_total{job="todo-platform-auth-service"}[5m])` | Requests per second |
| Todo Request Rate | `rate(http_requests_total{job="todo-platform-todo-service"}[5m])` | Requests per second |
| 4xx Error Rate | `rate(http_requests_total{status=~"4.."}[5m])` | Client errors/s |
| 5xx Error Rate | `rate(http_requests_total{status=~"5.."}[5m])` | Server errors/s |
| HTTP Method Breakdown | `sum by (method)(rate(http_requests_total[5m]))` | Rate by HTTP verb |
| HTTP Status Distribution | `sum by (status)(rate(http_requests_total[5m]))` | Rate by status code |

### Node.js Runtime

| Panel | PromQL | Description |
|---|---|---|
| Heap Used | `nodejs_heap_size_used_bytes` | Node.js heap usage in bytes |
| Heap Total | `nodejs_heap_size_total_bytes` | Total allocated heap |
| Event Loop Lag | `nodejs_eventloop_lag_seconds` | Event loop delay |
| Active Handles | `nodejs_active_handles_total` | Active async handles |

### Business Metrics

| Panel | PromQL | Description |
|---|---|---|
| Total Registered Users | `sum(app_registered_users_total)` | Aggregate across replicas |
| Total Todos | `sum(app_todos_total)` | All todos in database |
| Active Todos | `sum(app_active_todos_total)` | Incomplete todos |
| Completed Todos | `sum(app_completed_todos_total)` | Completed todos |
| Completion Rate | `sum(app_completed_todos_total) / sum(app_todos_total) * 100` | Percentage |

> **Note**: Business metrics use `sum()` aggregation to avoid duplicate values from multiple pod replicas all exposing the same database counts.

### Autoscaling

| Panel | PromQL | Description |
|---|---|---|
| HPA Current Replicas | `kube_horizontalpodautoscaler_status_current_replicas` | Per HPA |
| HPA Desired Replicas | `kube_horizontalpodautoscaler_status_desired_replicas` | Per HPA |
| HPA Min Replicas | `kube_horizontalpodautoscaler_spec_min_replicas` | Per HPA |
| HPA Max Replicas | `kube_horizontalpodautoscaler_spec_max_replicas` | Per HPA |

### Pod Restarts

| Panel | PromQL | Description |
|---|---|---|
| Pod Restart Count | `kube_pod_container_status_restarts_total{namespace="todo-platform"}` | Per container |

### Log Panels

Each service has a log panel linked to Loki:

```
{namespace="todo-platform", container="frontend"} | json
{namespace="todo-platform", container="auth-service"} | json
{namespace="todo-platform", container="todo-service"} | json
{namespace="todo-platform", container="postgres"} | json
```

---

## 6. Useful PromQL Queries

```promql
# All pods in todo-platform namespace
kube_pod_info{namespace="todo-platform"}

# CPU usage per pod
rate(container_cpu_usage_seconds_total{namespace="todo-platform"}[5m])

# Memory usage per pod
container_memory_working_set_bytes{namespace="todo-platform"}

# Auth service request total
http_requests_total{job="todo-platform-auth-service"}

# HTTP 5xx error rate for todo-service
rate(http_requests_total{job="todo-platform-todo-service", status=~"5.."}[5m])

# HPA scaling status
kube_horizontalpodautoscaler_status_current_replicas{namespace="todo-platform"}

# Pod ready status
kube_pod_status_ready{namespace="todo-platform", condition="true"}

# Node CPU utilization
1 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m]))

# PVC usage
kubelet_volume_stats_used_bytes{namespace="todo-platform"}
```

---

## 7. Prometheus Alerts

Custom alerting rules are defined in `monitoring/prometheus/alerts.yaml`.

Key alerts:

| Alert | Condition | Severity |
|---|---|---|
| `TodoServiceDown` | `up{job="todo-platform-todo-service"} == 0` for 1m | Critical |
| `AuthServiceDown` | `up{job="todo-platform-auth-service"} == 0` for 1m | Critical |
| `HighErrorRate` | 5xx rate > 0.1 req/s for 2m | Warning |
| `PodRestartingFrequently` | restart count > 5 in 10m | Warning |
| `PVCUsageHigh` | PVC used > 80% | Warning |

```bash
# Check alert rules
kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-prometheus 9090:9090 &
# Navigate to http://localhost:9090/alerts
```

---

## 8. Loki Setup

Loki is deployed using the `grafana/loki` Helm chart in the `logging` namespace.

Configuration is in `monitoring/loki/values.yaml`.

```bash
# Check Loki is running
kubectl get pods -n logging -l app.kubernetes.io/name=loki

# Check Loki is ready
kubectl port-forward -n logging svc/loki 3100:3100 &
curl http://localhost:3100/ready

# Query labels (via LogQL)
curl http://localhost:3100/loki/api/v1/labels
```

Loki stores logs in a single-binary deployment using local storage (suitable for development). Data is indexed by stream labels: `namespace`, `pod`, `container`, `node_name`.

---

## 9. Vector Log Collector

Vector replaces Promtail as the log shipping agent because Promtail encountered inotify file descriptor limits in the k3d environment.

Vector is deployed as a `DaemonSet` — one pod per node — so it can read container log files from the node filesystem.

Configuration is in `monitoring/loki/vector-values.yaml`.

**How Vector works:**

```
Node filesystem: /var/log/containers/*.log
         │
         ▼
Vector DaemonSet (runs on each node)
   ├── Source: kubernetes_logs (reads pod logs)
   ├── Transform: parse JSON, add Kubernetes metadata labels
   └── Sink: loki (HTTP push to http://loki.logging:3100)
```

```bash
# Check Vector DaemonSet
kubectl get daemonset -n logging

# Check Vector pod logs
kubectl logs -n logging -l app.kubernetes.io/name=vector --tail=30

# Verify Vector is shipping to Loki
kubectl logs -n logging -l app.kubernetes.io/name=vector | grep "sent"
```

---

## 10. Viewing Logs in Grafana

### Explore view

1. Grafana → Explore
2. Select datasource: **Loki**
3. Enter a LogQL query:

```logql
# All logs from todo-platform namespace
{namespace="todo-platform"}

# Auth service logs only
{namespace="todo-platform", container="auth-service"}

# Filter for errors
{namespace="todo-platform"} |= "error"

# Filter for specific pod
{pod="todo-platform-todo-service-abc123"}

# Count log lines per service over time
sum by (container) (count_over_time({namespace="todo-platform"}[5m]))
```

### Dashboard log panels

The Grafana dashboard includes log panels for each service showing the last 50 log lines. These panels use:
- Datasource: Loki
- Query: `{namespace="todo-platform", container="<service-name>"}`
- Visualization: Logs panel
- Time range: Last 15 minutes

---

## 11. Access Commands

```bash
# Grafana
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80
# http://localhost:3000 | admin / prom-operator

# Prometheus
kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-prometheus 9090:9090
# http://localhost:9090

# Alertmanager
kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-alertmanager 9093:9093
# http://localhost:9093

# Loki
kubectl port-forward -n logging svc/loki 3100:3100
# curl http://localhost:3100/ready
# curl http://localhost:3100/loki/api/v1/labels
```

---

## 12. Troubleshooting Observability

### Prometheus shows no targets for app services

**Cause**: ServiceMonitor labels do not match service labels or the `release` label does not match the kube-prometheus-stack release name.

**Fix**:
1. Check ServiceMonitor selector: `kubectl get servicemonitor -n monitoring -o yaml`
2. Check service labels: `kubectl get svc -n todo-platform --show-labels`
3. Ensure `matchLabels` values align exactly
4. Ensure ServiceMonitor has `labels.release: monitoring` (must match Helm release name)

### Grafana shows "No data" on business metrics panels

**Cause**: Multiple pod replicas each expose the same counter, causing duplicate values when using direct metrics without aggregation.

**Fix**: Wrap PromQL in `sum()` aggregation:
```promql
# Wrong
app_todos_total{job="todo-platform-todo-service"}

# Correct
sum(app_todos_total{job="todo-platform-todo-service"})
```

### Loki not connected in Grafana

**Cause**: Incorrect Loki URL in the Grafana datasource configuration.

**Fix**: Set the URL to include the full Kubernetes service DNS:
```
http://loki.logging.svc.cluster.local:3100
# or (shorter form within cluster)
http://loki.logging:3100
```

### Vector pods not running

**Cause**: Insufficient permissions for Vector to read node log files.

**Fix**: Check Vector DaemonSet pod logs and ensure the ServiceAccount has the correct RBAC permissions to list pods and read cluster metadata.

```bash
kubectl describe daemonset vector -n logging
kubectl logs -n logging -l app.kubernetes.io/name=vector
```

### Todo service panel shows "Down"

**Cause**: The Prometheus `job` label in the ServiceMonitor does not match the dashboard PromQL query.

**Fix**: Check the job label:
```bash
# See actual job labels
curl -s http://localhost:9090/api/v1/labels | python3 -m json.tool | grep job
curl -s 'http://localhost:9090/api/v1/series?match[]=up' | python3 -m json.tool
```
Update the dashboard PromQL to use the actual `job` label value.

---

*See [troubleshooting.md](troubleshooting.md) for general cluster troubleshooting.*
