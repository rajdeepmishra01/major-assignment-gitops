# Major Assignment GitOps Repository

This repository contains the Kubernetes, Helm, Flux, monitoring, and policy configuration for the Major Assignment DevOps platform.

## Project Components

- Kubernetes manifests
- Helm chart for the todo platform
- Flux GitOps configuration
- Monitoring configuration using Prometheus and Grafana
- Loki logging configuration
- RBAC, NetworkPolicy, ResourceQuota, LimitRange, PDB, and PriorityClass policies
- Advanced Kubernetes examples: HPA, VPA, CronJobs, CRDs, and troubleshooting notes

## Repository Structure

```text
major-assignment-gitops/
├── advanced/
├── clusters/
│   └── local/
│       ├── apps/
│       └── flux-system/
├── helm-charts/
│   └── todo-platform/
├── k8s/
├── monitoring/
└── policies/