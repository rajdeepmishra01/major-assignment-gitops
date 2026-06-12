# Project Overview

---

## Introduction

The **Todo Platform** is a cloud-native microservices application built as part of the Kubernetes DevOps Major Assignment.

The project demonstrates the complete software delivery lifecycle, including containerization, Kubernetes deployment, GitOps automation, observability, security, autoscaling, and CI/CD.

---

## Technology Stack

| Layer         | Technology           |
| ------------- | -------------------- |
| Frontend      | React + Vite         |
| Backend       | Node.js + Express    |
| Database      | PostgreSQL           |
| Containers    | Docker               |
| Orchestration | Kubernetes (k3d)     |
| Packaging     | Helm                 |
| GitOps        | FluxCD               |
| Monitoring    | Prometheus & Grafana |
| Logging       | Loki & Vector        |
| CI/CD         | GitHub Actions       |

---

## Platform Architecture

```text
Developer
    |
    v
GitHub Actions
    |
    v
GHCR
    |
    v
FluxCD
    |
    v
Kubernetes Cluster
    |
    +-- Frontend
    +-- Auth Service
    +-- Todo Service
    +-- PostgreSQL
```

---

## Project Objectives

| Objective        | Implementation           |
| ---------------- | ------------------------ |
| Containerization | Docker                   |
| Orchestration    | Kubernetes (k3d)         |
| Packaging        | Helm                     |
| GitOps           | FluxCD                   |
| Image Automation | Flux Image Automation    |
| Monitoring       | Prometheus & Grafana     |
| Logging          | Loki & Vector            |
| Security         | RBAC, NetworkPolicy, TLS |
| Autoscaling      | HPA & VPA                |
| Operations       | Jobs, CronJobs, CRDs     |
| CI/CD            | GitHub Actions           |

---

## Application Services

| Service      | Technology            | Purpose          |
| ------------ | --------------------- | ---------------- |
| Frontend     | React, Vite           | User Interface   |
| Auth Service | Node.js, Express, JWT | Authentication   |
| Todo Service | Node.js, Express      | Todo Management  |
| PostgreSQL   | PostgreSQL            | Data Persistence |