# Demo Script — Todo Platform

> Live walkthrough guide for the evaluator demo session.

---

## Overview

This script guides the presenter through a live demonstration of the entire platform — from source code to GitOps deployment, monitoring, logging, and scaling. The demo should take approximately 20–30 minutes.

**Preparation checklist before the demo:**
- [ ] k3d cluster is running: `kubectl get nodes`
- [ ] All pods are Running: `kubectl get pods -A`
- [ ] Flux is reconciled: `flux get helmreleases -A`
- [ ] Grafana is accessible: `kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80`
- [ ] Terminal windows open: at least 2 (cluster + gitops repo)
- [ ] Browser open to `https://localhost:8443`

---

## Step 1 — Show Both Repositories

**What to show:**

Open GitHub in the browser and navigate to both repositories.

1. `major-assignment-app` — Application source code repo
   - Show the folder structure: `apps/frontend/`, `apps/auth-service/`, `apps/todo-service/`, `apps/database/`
   - Show `docker-compose.yml` (local development)
   - Show `sonar-project.properties` (SonarCloud integration)
   - Show `.github/workflows/pipeline.yml` (the CI pipeline)

2. `major-assignment-gitops` — GitOps configuration repo
   - Show the folder structure: `clusters/`, `helm-charts/`, `monitoring/`, `policies/`, `security/`
   - Explain: *"This repo is the single source of truth for everything in the cluster. No kubectl apply is done manually."*

---

## Step 2 — Show Application Code

**What to show:**

In the `major-assignment-app` repo:

1. Open `apps/auth-service/src/routes/` — show register and login routes
2. Open `apps/todo-service/src/routes/` — show CRUD routes with JWT middleware
3. Open `apps/frontend/src/` — show the React components
4. Open `apps/database/init.sql` — show the schema (users and todos tables)

**Key talking points:**
- Auth service issues JWT tokens signed with `JWT_SECRET`
- Todo service validates JWT on every request before touching the database
- Todos are scoped per user via `user_id` in the database
- The frontend communicates with the backend only through the Kubernetes ingress

---

## Step 3 — Show GitHub Actions CI Pipeline

**What to show:**

In the `major-assignment-app` repository:
1. Navigate to **Actions** tab
2. Click on the latest successful run
3. Walk through each step:
   - Install dependencies
   - Lint and build
   - Unit tests (auth-service and todo-service)
   - Coverage generation
   - SonarCloud scan
   - Docker build
   - Trivy vulnerability scan
   - Push to GHCR

**Key talking points:**
- Every push to `main` triggers this pipeline automatically
- Tests run for both backend services using Vitest
- SonarCloud provides static analysis and code quality metrics
- Trivy scans the Docker images for known CVEs (report-only, does not block)
- Images are tagged with the GitHub Actions run number (e.g., `:33`)

```bash
# Show latest pipeline run
gh run list --repo rajdeepmishra01/major-assignment-app --limit 5
```

---

## Step 4 — Show Docker Images in GHCR

**What to show:**

1. Navigate to GitHub Profile → Packages
2. Show three packages: `frontend`, `auth-service`, `todo-service`
3. Show multiple version tags (the build numbers from CI)

**Key talking points:**
- Images are private packages in GHCR
- Tags correspond to GitHub Actions run numbers
- Flux uses these tags for image automation

```bash
# List recent image tags (if gh CLI or crane is available)
crane ls ghcr.io/rajdeepmishra01/frontend | tail -10
```

---

## Step 5 — Show GitOps Helm Values with Image Tags

**What to show:**

Open `clusters/local/apps/todo-platform/helmrelease.yaml` in the browser or editor.

```yaml
values:
  images:
    frontend: ghcr.io/rajdeepmishra01/frontend:33 # {"$imagepolicy": "flux-system:frontend"}
    authService: ghcr.io/rajdeepmishra01/auth-service:33 # {"$imagepolicy": "flux-system:auth-service"}
    todoService: ghcr.io/rajdeepmishra01/todo-service:33 # {"$imagepolicy": "flux-system:todo-service"}
```

**Key talking points:**
- The `# {"$imagepolicy": ...}` comment is a Flux marker
- The image-automation-controller reads this file and updates the tag
- When Flux detects a new tag in GHCR, it commits an update to this exact line
- This is the link between CI (GHCR push) and CD (Kubernetes deployment)

---

## Step 6 — Show Flux Image Automation

```bash
# Show ImageRepository status (is GHCR being scanned?)
flux get image repository -A

# Show ImagePolicy status (which tag is selected?)
flux get image policy -A

# Show ImageUpdateAutomation status (is it committing updates?)
flux get image update -A
```

**Expected output:**
```
NAME           LAST SCAN    READY   MESSAGE
auth-service   2s ago       True    successful scan: found 33 tags
frontend       2s ago       True    successful scan: found 33 tags
todo-service   2s ago       True    successful scan: found 33 tags
```

---

## Step 7 — Push a Small Change to Trigger the Pipeline

**What to show:**

In the terminal, make a small, visible change to trigger the CI:

```bash
cd major-assignment-app

# Add a comment or update a version string
echo "// demo: $(date)" >> apps/todo-service/src/app.js

git add .
git commit -m "demo: trigger pipeline for evaluation"
git push origin main
```

Then switch to the GitHub Actions tab and watch the pipeline start.

**Key talking points:**
- The commit triggers GitHub Actions automatically
- A new image will be built and pushed to GHCR
- Flux will detect the new tag within 1 minute

---

## Step 8 — Show Flux Detecting the New Image

After the CI pipeline finishes (new image pushed to GHCR):

```bash
# Force image scan (don't wait for 1-minute interval)
flux reconcile image repository frontend -n flux-system
flux reconcile image repository auth-service -n flux-system
flux reconcile image repository todo-service -n flux-system

# Show image automation status
flux get image repository -A
flux get image policy -A
```

Show the automated commit in the GitOps repo:

```bash
cd major-assignment-gitops
git pull
git log --oneline -5
```

**Expected output:**
```
a1b2c3d Update image tags
e5f6g7h Previous commit
...
```

Show the changed file:
```bash
git show HEAD
```

---

## Step 9 — Show HelmRelease Reconciliation

```bash
# Check HelmRelease is reconciling
flux get helmreleases -A

# Trigger manual reconciliation if needed
flux reconcile source git flux-system
flux reconcile helmrelease todo-platform -n flux-system

# Watch the reconciliation
flux get helmreleases -A --watch
```

**Expected output:**
```
NAME            REVISION   SUSPENDED   READY   MESSAGE
todo-platform   0.1.10     False       True    Release reconciliation succeeded
```

---

## Step 10 — Show Pods Rolling Out

```bash
# Watch the rolling update
kubectl rollout status deployment/todo-platform-frontend -n todo-platform
kubectl rollout status deployment/todo-platform-auth-service -n todo-platform
kubectl rollout status deployment/todo-platform-todo-service -n todo-platform

# Watch pods in real time
watch kubectl get pods -n todo-platform

# Verify new image tag is running on all pods
kubectl get pods -n todo-platform -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].image}{"\n"}{end}'
```

**Key talking points:**
- Kubernetes performs a rolling update: new pods come up before old pods are terminated
- The readiness probe ensures traffic is only routed to healthy pods
- Zero downtime during deployment

---

## Step 11 — Open the Application in Browser

```bash
# Ensure ingress is working
kubectl get ingress -n todo-platform
kubectl get certificate -n todo-platform
```

Open `https://localhost:8443` in the browser.

Accept the self-signed certificate warning (click Advanced → Proceed).

**Key talking points:**
- Traefik terminates TLS using the cert-manager certificate
- The self-signed certificate is expected in a local development environment
- In production, this would be a Let's Encrypt certificate

---

## Step 12 — Register, Login, and Create a Todo

Live demonstration in the browser:

1. Click **Register** → enter username and password → submit
2. Click **Login** → enter the same credentials → submit
3. Verify the JWT token is returned (show in browser DevTools → Network tab)
4. Create a new todo: *"Kubernetes GitOps demo todo"*
5. Mark the todo as complete
6. Delete the todo
7. Create another todo

Alternatively, use the terminal:

```bash
# Register
curl -k -X POST https://localhost:8443/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"demo","password":"demo123"}'

# Login
TOKEN=$(curl -k -s -X POST https://localhost:8443/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"demo","password":"demo123"}' | jq -r '.token')

# Create todo
curl -k -X POST https://localhost:8443/api/todos \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"title":"GitOps demo todo"}'

# List todos
curl -k https://localhost:8443/api/todos \
  -H "Authorization: Bearer $TOKEN"
```

---

## Step 13 — Show Prometheus Targets

```bash
# Port-forward Prometheus
kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-prometheus 9090:9090 &
```

Open `http://localhost:9090/targets` in the browser.

Show:
- `serviceMonitor/monitoring/todo-platform/0` → auth-service endpoints (UP)
- `serviceMonitor/monitoring/todo-platform/1` → todo-service endpoints (UP)
- kube-state-metrics, node-exporter (UP)

**Key talking points:**
- Prometheus automatically discovers scrape targets via ServiceMonitor
- The ServiceMonitor selector labels must exactly match the service labels
- Each pod replica is a separate scrape target

---

## Step 14 — Show Grafana Dashboard

```bash
# Port-forward Grafana (if not already running)
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80 &
```

Open `http://localhost:3000` → log in with `admin` / `prom-operator`.

Open the **Todo Platform** dashboard. Walk through:

1. **Service Health** row — all services showing as UP (green)
2. **Request Metrics** row — live request rates
3. **Node.js Runtime** row — heap and CPU usage
4. **Business Metrics** row — total users, todos, completion rate
5. **Autoscaling** row — HPA current/desired replicas
6. **Log Panels** row — live logs per service

**Key talking points:**
- The dashboard mixes infrastructure metrics (from Prometheus) with application metrics (from `prom-client`)
- Business metrics use `sum()` aggregation because multiple replicas expose the same database counters
- The Loki log panels stream live logs without leaving Grafana

---

## Step 15 — Show Loki Logs

In Grafana → **Explore** → select datasource: **Loki**:

```logql
# Live logs from all app services
{namespace="todo-platform"}

# Filter for a specific service
{namespace="todo-platform", container="auth-service"}

# Show only error lines
{namespace="todo-platform"} |= "error"
```

**Key talking points:**
- Vector DaemonSet collects logs from every pod on every node
- Logs are stored in Loki with labels for namespace, pod, container, node
- LogQL is the query language — similar in concept to PromQL but for logs
- Replaced Promtail with Vector due to inotify descriptor limits in k3d

---

## Step 16 — Show HPA, VPA, CronJob, and PVC

```bash
# HPA — current state
kubectl get hpa -n todo-platform
kubectl describe hpa -n todo-platform

# VPA — recommendations
kubectl get vpa -n todo-platform
kubectl describe vpa todo-service-vpa -n todo-platform

# CronJob and Jobs
kubectl get cronjob -n todo-platform
kubectl get jobs -n todo-platform

# Persistence
kubectl get pvc -n todo-platform
kubectl get pv
```

**HPA explanation:**
- Currently at 2 replicas (minimum) — CPU is low in the demo
- If CPU exceeds 50%, it will scale up automatically (up to 6 replicas)
- Scaling is handled by the Kubernetes metrics server

**VPA explanation:**
- Running in Off/recommendation mode — it never restarts pods automatically
- Shows the recommended CPU and memory values for `todo-service`
- These recommendations help right-size resource requests

**CronJob explanation:**
- Simulates a PostgreSQL backup running every 6 hours
- Each run creates a Kubernetes `Job` object
- The job container runs `pg_dump` against the database

**PVC explanation:**
- `postgres-pvc` is a 1Gi `PersistentVolumeClaim`
- Backed by `local-path` storage (host directory on a k3d agent node)
- If the postgres pod restarts or reschedules, the data persists

---

## Step 17 — Key Learnings and Challenges

Conclude the demo by explaining:

### Key Learnings

1. **GitOps means Git is the source of truth** — the cluster state is always a reflection of the Git repo. If it's not in Git, Flux will revert it.

2. **Helm makes Kubernetes manageable** — instead of 15 separate YAML files, the entire application is one Helm chart with parameterized values.

3. **Image automation closes the loop** — the `$imagepolicy` marker comment is a small but powerful feature that makes CI→CD fully automatic.

4. **ServiceMonitor labels are critical** — a single label mismatch causes Prometheus to silently miss scrape targets. Always check with `kubectl get svc --show-labels`.

5. **VPA is not just autoscaling** — in recommendation mode, VPA is a resource analysis tool that tells you whether your requests and limits are correctly sized.

6. **Vector vs Promtail** — tool selection matters in constrained environments. Promtail hit OS file descriptor limits; Vector handled the same workload without issues.

### Challenges Faced

1. Flux image automation committed updates but pods were not rolling — fixed by explicitly reconciling the HelmRelease.
2. Prometheus had no targets — ServiceMonitor selector did not match service labels.
3. Loki not reachable from Grafana — URL was missing the namespace (needed `loki.logging:3100`, not just `loki:3100`).
4. Grafana business metrics doubled — multiple replicas exposing same DB counters; fixed with `sum()` aggregation.
5. SonarCloud coverage upload failed — LCOV path prefix did not match `sonar.javascript.lcov.reportPaths`.
6. Trivy found HIGH CVEs in base images — reconfigured to `exit-code: 0` so CI does not block.

---

## Quick Reference: Demo Commands

```bash
# Cluster health
kubectl get nodes -o wide
kubectl get pods -A

# Application pods
kubectl get pods -n todo-platform
kubectl get svc -n todo-platform
kubectl get ingress -n todo-platform

# Flux status
flux get sources git -A
flux get kustomizations -A
flux get helmreleases -A
flux get image repository -A
flux get image policy -A
flux get image update -A

# Force reconciliation
flux reconcile source git flux-system
flux reconcile helmrelease todo-platform -n flux-system

# Monitoring
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80
kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-prometheus 9090:9090

# Autoscaling
kubectl get hpa -n todo-platform
kubectl describe vpa todo-service-vpa -n todo-platform

# CronJob
kubectl get cronjob -n todo-platform
kubectl get jobs -n todo-platform

# Application test
curl -k https://localhost:8443/
curl -k -X POST https://localhost:8443/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"demo","password":"demo123"}'
```

---

*See [troubleshooting.md](troubleshooting.md) if issues arise during the demo.*
