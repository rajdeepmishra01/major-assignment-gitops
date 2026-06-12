# Local Development

---

## Overview

The **Todo Platform** can be executed locally using Docker Compose before deployment to Kubernetes.

The local environment consists of:

- Frontend (React)
- Auth Service
- Todo Service
- PostgreSQL

This setup allows developers to validate application functionality and API integration before container images are promoted through the CI/CD pipeline.

---

## Prerequisites

| Tool           | Required |
| -------------- | -------- |
| Docker         | Yes      |
| Docker Compose | Yes      |
| Git            | Yes      |

Verify installations:

```bash
docker --version
docker compose version
git --version
```

---

## Project Structure

```text
major-assignment-app/
|
+-- apps/
|   +-- frontend/
|   +-- auth-service/
|   +-- todo-service/
|   +-- database/
|
+-- docker-compose.yml
```

---

## Environment Configuration

Create environment files for backend services.

### Auth Service

```text
apps/auth-service/.env
```

### Todo Service

```text
apps/todo-service/.env
```

**Example:**

```env
DATABASE_URL=postgres://postgres:postgres@postgres:5432/todoapp
JWT_SECRET=your-secret-key
```

---

## Running the Application

Start all services:

```bash
docker compose up --build
```

Verify running containers:

```bash
docker ps
```

**Expected services:**

- `frontend`
- `auth-service`
- `todo-service`
- `postgres`


---

## Application Validation

Access the frontend at:

```text
http://localhost:5173
```

Verify the following flows:

- User Registration
- User Login
- Todo CRUD Operations


---

## API Health Checks

**Auth Service:**

```bash
curl http://localhost:5001/health
```

**Todo Service:**

```bash
curl http://localhost:5002/health
```

**Expected Response:**

```json
{
  "status": "UP"
}
```

---

## Database Persistence

Verify PostgreSQL persistence across container restarts:

```bash
docker compose down
docker compose up -d
```

Previously created todo items should remain available after restart.


---

## Development Workflow

```text
Code Change
     |
     v
Docker Compose
     |
     v
Local Testing
     |
     v
Git Commit
     |
     v
GitHub Actions
     |
     v
Kubernetes Deployment
```