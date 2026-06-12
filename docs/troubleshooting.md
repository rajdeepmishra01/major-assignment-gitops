# Troubleshooting — Todo Platform

> Common issues encountered during setup and operation, with root causes and fixes.

---

## Table of Contents

1. [Cluster and Node Issues](#1-cluster-and-node-issues)
2. [Image Pull Failures](#2-image-pull-failures)
3. [Pod Not Starting / CrashLoopBackOff](#3-pod-not-starting--crashloopbackoff)
4. [Flux Reconciliation Failures](#4-flux-reconciliation-failures)
5. [Flux Image Automation Not Updating](#5-flux-image-automation-not-updating)
6. [HelmRelease Failures](#6-helmrelease-failures)
7. [Ingress / TLS Issues](#7-ingress--tls-issues)
8. [Prometheus No Scrape Targets](#8-prometheus-no-scrape-targets)
9. [Grafana Issues](#9-grafana-issues)
10. [Loki / Vector Issues](#10-loki--vector-issues)
11. [HPA and VPA Issues](#11-hpa-and-vpa-issues)
12. [Database / PostgreSQL Issues](#12-database--postgresql-issues)
13. [CI/CD Pipeline Failures](#13-cicd-pipeline-failures)
14. [Application API Errors](#14-application-api-errors)
15. [General Diagnostic Commands](#15-general-diagnostic-commands)

---

## 1. Cluster and Node Issues

### k3d cluster fails to start

**Symptom:** `k3d cluster create` hangs or fails.

**Cause:** Ports 8080 or 8443 are already in use on the host.

**Fix:**
```bash
# Check what is using the ports
netstat -ano | findstr :8080
netstat -ano | findstr :8443

# Kill conflicting process or use different ports
k3d cluster create todo-platform \
  --servers 1 --agents 2 \
  -p "9080:80@loadbalancer" \
  -p "9443:443@loadbalancer"
```

### Nodes stuck in NotReady

**Symptom:** `kubectl get nodes` shows nodes as `NotReady`.

**Fix:**
```bash
# Check node conditions
kubectl describe node <node-name>

# Check kubelet logs
k3d node exec k3d-todo-platform-server-0 -- journalctl -u kubelet -n 50

# Restart the cluster
k3d cluster stop todo-platform
k3d cluster start todo-platform
```

### kubectl cannot connect to cluster

**Symptom:** `The connection to the server localhost:6443 was refused`.

**Fix:**
```bash
# Check cluster is running
k3d cluster list

# Switch kubeconfig context
k3d kubeconfig merge todo-platform --kubeconfig-switch-context
kubectl config use-context k3d-todo-platform
kubectl get nodes
```

---

## 2. Image Pull Failures

### ErrImagePull / ImagePullBackOff

**Symptom:** Pods show `ErrImagePull` or `ImagePullBackOff` in `kubectl get pods`.

**Cause:** The cluster cannot authenticate to GHCR, or the image tag does not exist.

**Diagnosis:**
```bash
kubectl describe pod <pod-name> -n todo-platform | grep -A 10 "Events"
```

**Fix 1 — Missing imagePullSecret:**
```bash
kubectl create secret docker-registry ghcr-credentials \
  --docker-server=ghcr.io \
  --docker-username=rajdeepmishra01 \
  --docker-password=<GITHUB_PAT> \
  --docker-email=<your-email> \
  -n todo-platform

# Reference the secret in the pod spec or patch the default service account
kubectl patch serviceaccount default \
  -p '{"imagePullSecrets": [{"name": "ghcr-credentials"}]}' \
  -n todo-platform
```

**Fix 2 — Wrong image tag:**
```bash
# Verify image exists in GHCR
crane ls ghcr.io/rajdeepmishra01/frontend

# Check what tag Flux set in helmrelease.yaml
kubectl get helmrelease todo-platform -n flux-system -o yaml | grep image
```

**Fix 3 — Image visibility:**
Ensure GHCR packages are set to **public** or that the pull secret has the correct PAT scope (`read:packages`).

---

## 3. Pod Not Starting / CrashLoopBackOff

### CrashLoopBackOff

**Symptom:** Pod enters `CrashLoopBackOff`.

**Diagnosis:**
```bash
# Check current logs
kubectl logs <pod-name> -n todo-platform

# Check previous container logs (before last crash)
kubectl logs <pod-name> -n todo-platform --previous

# Describe pod for events
kubectl describe pod <pod-name> -n todo-platform
```

**Common causes for auth-service / todo-service:**

| Cause | Log message | Fix |
|---|---|---|
| PostgreSQL not ready | `ECONNREFUSED` or `getaddrinfo failed` | Wait for postgres pod to be Ready; check `DATABASE_URL` secret |
| Wrong JWT_SECRET | `JsonWebTokenError` | Verify `JWT_SECRET` is correctly set in the Kubernetes Secret |
| Port conflict | `EADDRINUSE` | Ensure only one container is listening on the expected port |
| Missing env variable | `Cannot read properties of undefined` | Check ConfigMap and Secret are mounted correctly |

**Verify environment variables:**
```bash
kubectl exec -it <pod-name> -n todo-platform -- env | grep -E "DATABASE_URL|JWT_SECRET|PORT"
```

### OOMKilled (Out of Memory)

**Symptom:** Pod is killed with reason `OOMKilled`.

**Fix:** Increase memory limit in `values.yaml` or use VPA recommendations to right-size.

```bash
kubectl describe pod <pod-name> -n todo-platform | grep -A 5 "Last State"
# Look for: "Reason: OOMKilled"
```

---

## 4. Flux Reconciliation Failures

### Kustomization shows READY=False

**Diagnosis:**
```bash
flux get kustomizations -A
kubectl describe kustomization flux-system -n flux-system
```

**Common causes:**

| Message | Cause | Fix |
|---|---|---|
| `failed to get source` | GitRepository not syncing | Check `flux get sources git -A` and SSH/HTTPS credentials |
| `HelmRelease not found` | helmrelease.yaml has wrong path | Verify file path and kustomization resources list |
| `namespace not found` | Namespace not created yet | Add `createNamespace: true` to HelmRelease install config |

### GitRepository not syncing

```bash
flux get sources git -A
kubectl describe gitrepository flux-system -n flux-system
```

**Cause:** SSH key expired, or GitHub PAT has insufficient scope.

**Fix:**
```bash
# Re-bootstrap with fresh credentials
export GITHUB_TOKEN=<new-pat>
flux bootstrap github \
  --owner=rajdeepmishra01 \
  --repository=major-assignment-gitops \
  --branch=main \
  --path=./clusters/local \
  --personal \
  --components-extra=image-reflector-controller,image-automation-controller
```

---

## 5. Flux Image Automation Not Updating

### ImageRepository shows no tags

**Diagnosis:**
```bash
flux get image repository -A
kubectl describe imagerepository frontend -n flux-system
```

**Cause 1:** GHCR credentials not configured for Flux.

**Fix:**
```bash
# Create secret in flux-system namespace
kubectl create secret docker-registry ghcr-credentials \
  --docker-server=ghcr.io \
  --docker-username=rajdeepmishra01 \
  --docker-password=<GITHUB_PAT> \
  -n flux-system

# Reference in ImageRepository
# spec.secretRef.name: ghcr-credentials
```

### ImagePolicy selects no image

**Diagnosis:**
```bash
kubectl describe imagepolicy frontend -n flux-system
```

**Cause:** No tags match the policy. If using `numerical` order and CI has not run yet, there are no numeric tags.

**Fix:** Push a change to trigger the CI pipeline. After the first build, numeric tags will exist.

### Image automation commits but pods don't update

**Cause:** The HelmRelease reconciliation interval has not triggered yet, or the source-controller has not picked up the new commit.

**Fix:**
```bash
# Force immediate reconciliation
flux reconcile source git flux-system
flux reconcile helmrelease todo-platform -n flux-system
```

### `$imagepolicy` marker not found

**Cause:** The comment marker format is incorrect or the file path is wrong in `ImageUpdateAutomation.spec.update.path`.

**Fix:** Ensure the marker is an exact JSON comment on the same line as the image tag:
```yaml
image: ghcr.io/rajdeepmishra01/frontend:33 # {"$imagepolicy": "flux-system:frontend"}
```

---

## 6. HelmRelease Failures

### HelmRelease shows READY=False

```bash
flux get helmreleases -A
kubectl describe helmrelease todo-platform -n flux-system
```

**Common failure messages:**

| Message | Cause | Fix |
|---|---|---|
| `chart not found` | Chart path is incorrect | Verify `chart.spec.chart` path matches the actual directory |
| `values validation failed` | Required value missing in values.yaml | Add missing key to the `values:` block in HelmRelease |
| `upgrade retries exhausted` | Pod keeps crashing during rollout | Fix the underlying pod crash first, then run `flux reconcile helmrelease` |

**Reset a failed HelmRelease:**
```bash
# Suspend, delete the Helm release, resume
flux suspend helmrelease todo-platform -n flux-system
helm uninstall todo-platform -n todo-platform
flux resume helmrelease todo-platform -n flux-system
```

---

## 7. Ingress / TLS Issues

### 404 Not Found on all routes

**Diagnosis:**
```bash
kubectl get ingress -n todo-platform
kubectl describe ingress -n todo-platform
kubectl get pods -n kube-system -l app.kubernetes.io/name=traefik
```

**Cause:** Traefik is not running, or the ingress annotation is incorrect.

**Fix:** Ensure the Traefik pod is running. Check ingress annotations match Traefik's ingress class:
```yaml
annotations:
  kubernetes.io/ingress.class: traefik
```

### Certificate not issued

**Diagnosis:**
```bash
kubectl get certificate -A
kubectl describe certificate todo-platform-tls -n todo-platform
kubectl get certificaterequest -n todo-platform
kubectl get challenges -n todo-platform
```

**Cause:** cert-manager pods not running, or ClusterIssuer not created.

**Fix:**
```bash
# Check cert-manager
kubectl get pods -n cert-manager

# Check ClusterIssuer
kubectl get clusterissuer
kubectl describe clusterissuer selfsigned-cluster-issuer

# Re-apply security manifests
kubectl apply -f security/cert-manager/
```

### SSL handshake error

**Cause:** Browser refuses to accept the self-signed certificate.

**Fix:** Accept the browser warning manually (click Advanced → Proceed to localhost). Use `-k` flag with `curl` to skip TLS verification in tests.

---

## 8. Prometheus No Scrape Targets

### Auth-service or todo-service not appearing in Prometheus targets

**Diagnosis:**
```bash
# Check ServiceMonitor exists
kubectl get servicemonitor -n monitoring

# Check service labels
kubectl get svc -n todo-platform --show-labels

# Check ServiceMonitor selector
kubectl get servicemonitor todo-platform -n monitoring -o yaml | grep -A 5 selector
```

**Cause:** `selector.matchLabels` in ServiceMonitor does not match the labels on the Kubernetes `Service` resource.

**Fix:** Align the labels. For example, if the service has:
```yaml
labels:
  app.kubernetes.io/part-of: todo-platform
```

The ServiceMonitor must have:
```yaml
selector:
  matchLabels:
    app.kubernetes.io/part-of: todo-platform
```

Also ensure the ServiceMonitor has:
```yaml
labels:
  release: monitoring   # Must match kube-prometheus-stack Helm release name
```

---

## 9. Grafana Issues

### No data on dashboard panels

**Cause 1:** Prometheus datasource not configured.
**Fix:** Configuration → Data Sources → Prometheus → URL: `http://monitoring-kube-prometheus-prometheus.monitoring:9090`

**Cause 2:** Wrong `job` label in PromQL query.
**Fix:** Check actual job labels:
```bash
curl -s 'http://localhost:9090/api/v1/series?match[]=up' | python3 -m json.tool | grep job
```
Update the PromQL to use the exact `job` label value.

**Cause 3:** Loki datasource not configured.
**Fix:** Configuration → Data Sources → Loki → URL: `http://loki.logging:3100`

### Business metrics showing doubled values

**Cause:** Multiple pod replicas each expose the same database counter.

**Fix:** Use `sum()` aggregation in PromQL:
```promql
# Wrong: shows value from each replica separately
app_todos_total

# Correct: aggregates across all replicas
sum(app_todos_total{job="todo-platform-todo-service"})
```

### Grafana dashboard import fails

**Cause:** Dashboard JSON references a datasource UID that does not exist.

**Fix:** After importing, edit each panel → Change datasource to the correct configured datasource.

---

## 10. Loki / Vector Issues

### Loki not receiving logs

**Diagnosis:**
```bash
kubectl get pods -n logging
kubectl logs -n logging -l app.kubernetes.io/name=vector --tail=50
```

**Cause 1:** Vector sink URL is wrong.
**Fix:** Ensure Vector values configure the Loki sink endpoint as `http://loki.logging.svc.cluster.local:3100/loki/api/v1/push`.

**Cause 2:** Vector pod lacks permission to read node logs.
**Fix:**
```bash
kubectl get clusterrolebinding | grep vector
kubectl describe clusterrolebinding vector
```
Ensure Vector's ServiceAccount has `list pods` and `get pods` permissions at the cluster level.

### Loki ready endpoint returns 503

**Cause:** Loki is still starting up.

**Fix:** Wait 60 seconds after deployment, then retry:
```bash
kubectl rollout status deployment/loki -n logging
curl http://localhost:3100/ready
```

### Promtail crashing (if used instead of Vector)

**Cause:** Promtail inotify watcher exceeds the OS file descriptor limit.

**Fix:** Switch to Vector (already implemented in this project). Or increase system limits:
```bash
# On Linux host
sudo sysctl -w fs.inotify.max_user_instances=8192
sudo sysctl -w fs.inotify.max_user_watches=524288
```

---

## 11. HPA and VPA Issues

### HPA shows `<unknown>/50%` for CPU target

**Cause:** The metrics server is not installed or not running.

**Fix:**
```bash
kubectl get pods -n kube-system | grep metrics-server

# k3d includes metrics-server by default — if missing:
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Also check that resource `requests` are set on the deployment containers — HPA cannot calculate utilization without them.

### VPA shows no recommendations

**Cause:** VPA needs time to observe resource usage before generating recommendations (typically 15–30 minutes).

**Fix:** Wait and retry. Verify VPA recommender is running:
```bash
kubectl get pods -n kube-system | grep vpa
kubectl logs -n kube-system deployment/vpa-recommender --tail=30
```

---

## 12. Database / PostgreSQL Issues

### Services cannot connect to PostgreSQL

**Symptom:** Auth-service or todo-service shows `ECONNREFUSED` or `getaddrinfo failed`.

**Diagnosis:**
```bash
# Check postgres pod is running
kubectl get pods -n todo-platform -l app=postgres

# Check postgres service
kubectl get svc todo-platform-postgres -n todo-platform

# Verify DATABASE_URL in secret
kubectl get secret todo-platform-secret -n todo-platform -o jsonpath='{.data.DATABASE_URL}' | base64 -d
```

**Cause 1:** PostgreSQL pod is not yet Ready.
**Fix:** Wait for the postgres pod to show `Running/1/1 Ready`, then let the app pods restart.

**Cause 2:** Incorrect `DATABASE_URL` format.
**Fix:** Correct format: `postgresql://postgres:postgres@todo-platform-postgres:5432/todoapp`

### PostgreSQL PVC stuck in Pending

**Diagnosis:**
```bash
kubectl describe pvc postgres-pvc -n todo-platform
kubectl get storageclass
```

**Cause:** No `local-path` StorageClass in the cluster.

**Fix:** The `local-path` provisioner is included with k3d by default. If missing, reinstall it:
```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
```

---

## 13. CI/CD Pipeline Failures

### SonarCloud analysis fails with coverage error

**Cause:** LCOV file path prefix does not match what SonarCloud expects.

**Fix:** In `sonar-project.properties`:
```properties
sonar.javascript.lcov.reportPaths=apps/auth-service/coverage/lcov.info,apps/todo-service/coverage/lcov.info
```

Also ensure the paths are relative to the repository root, not the service directory.

### Trivy scan fails the pipeline

**Cause:** Trivy found HIGH or CRITICAL vulnerabilities and `exit-code` was set to `1`.

**Fix:** Set `exit-code: 0` to make Trivy a reporting tool without blocking the pipeline:
```yaml
- name: Trivy scan
  uses: aquasecurity/trivy-action@master
  with:
    exit-code: 0
```

### Docker build fails — COPY file not found

**Cause:** Build context does not include the required file.

**Fix:** Ensure the Dockerfile `COPY` paths are relative to the build context (the service directory, not the repo root).

---

## 14. Application API Errors

### 401 Unauthorized on /api/todos

**Cause:** JWT token is missing, expired, or the `Authorization` header format is wrong.

**Fix:** Ensure the header is `Authorization: Bearer <token>` (with exactly one space between `Bearer` and the token).

### 403 Forbidden

**Cause:** Token is valid but belongs to a different user than the resource being accessed.

### 500 Internal Server Error

**Cause:** Usually a database connection issue or an unhandled exception in the service.

**Fix:**
```bash
kubectl logs deployment/todo-platform-todo-service -n todo-platform --tail=30
kubectl logs deployment/todo-platform-auth-service -n todo-platform --tail=30
```

---

## 15. General Diagnostic Commands

```bash
# Full pod status
kubectl get pods -A -o wide

# Describe a failing pod
kubectl describe pod <pod-name> -n <namespace>

# Get logs (current)
kubectl logs <pod-name> -n <namespace>

# Get logs (previous container — if pod restarted)
kubectl logs <pod-name> -n <namespace> --previous

# Get events (sorted by time)
kubectl get events -n <namespace> --sort-by='.lastTimestamp'

# Check resource usage
kubectl top pods -n todo-platform
kubectl top nodes

# Check all Flux resources
flux get all -A

# Check for Flux errors
flux logs --level=error -A

# Helm release status
helm list -n todo-platform
helm history todo-platform -n todo-platform
helm status todo-platform -n todo-platform

# Helm dry-run to preview changes
helm upgrade todo-platform ./helm-charts/todo-platform \
  --namespace todo-platform \
  --dry-run \
  --debug \
  -f clusters/local/apps/todo-platform/helmrelease.yaml
```

---

*See [setup-guide.md](setup-guide.md) for initial setup steps.*
*See [gitops-flow.md](gitops-flow.md) for Flux-specific debugging.*
*See [monitoring-logging.md](monitoring-logging.md) for observability troubleshooting.*
