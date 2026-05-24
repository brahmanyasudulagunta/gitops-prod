# Gitops-Manifests

The continuous delivery (CD) tier of the platform. This repository contains the declarative Kubernetes manifests and ArgoCD configurations for deploying applications across all environments. It acts as the **single source of truth** for what runs in the cluster.

[![ArgoCD](https://img.shields.io/badge/CD-ArgoCD-EF7B4D?style=for-the-badge&logo=argo&logoColor=white)](https://argoproj.github.io/cd)
[![Kubernetes](https://img.shields.io/badge/Runtime-Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)](https://kubernetes.io)

---

## 3-Repo GitOps Architecture Role

This repository coordinates the GitOps delivery pattern:

| Repository | Purpose | Primary Operator / Owner |
| :--- | :--- | :--- |
| **`Platform-Infrastructure`** | Provisions secure environments, resource limits, and network policies. | Platform / DevOps Team |
| **`Application-Code`** | Contains the FastAPI, React, and PostgreSQL application source code. | Software Development Team |
| **`Gitops-Manifests`** (this) | Stores environment-specific K8s manifests, watched by ArgoCD. | GitOps Deployment Engine |

```
Platform-Infrastructure provisions namespaces  →  Gitops-Manifests deploys apps into them via ArgoCD
```

---

## Core Workflows & Lifecycles

```
1. Developer pushes code to Application-Code repository.
                       ↓
2. Jenkins CI builds a Docker image and pushes to DockerHub: ashrith2727/backend-gitops:<TAG>
                       ↓
3. Jenkins auto-updates Gitops-Manifests (e.g. environments/develop/backend.yaml) with the new tag.
                       ↓
4. ArgoCD detects the change in this repository and synchronizes state into the Kubernetes cluster.
```

---

## Repository Structure

```
Gitops-Manifests/
├── environments/
│   ├── develop/
│   │   ├── backend.yaml        # FastAPI backend pod and service configuration
│   │   ├── frontend.yaml       # React frontend pod and service configuration
│   │   ├── postgres.yaml       # Database pod, persistent volumes, and configuration
│   │   └── serviceaccount.yaml # Pod-level access identities (gitops-sa)
│   └── production/
│       ├── backend.yaml        # FastAPI backend specs (higher replicas/limits)
│       ├── frontend.yaml       # React frontend specs (higher replicas/limits)
│       ├── postgres.yaml       # Production database specs
│       └── serviceaccount.yaml # Production pod identities
├── argocd/
│   ├── develop-app.yaml        # ArgoCD App pointing to environments/develop
│   └── production-app.yaml     # ArgoCD App pointing to environments/production
└── README.md
```

---

## Target Environment Specifications

| Environment | Target Namespace | Service Replicas | CPU Allocation (Req/Lim) | Memory Allocation (Req/Lim) | Sync Policy |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **develop** | `develop` | 1 per service | 100m / 300m | 128Mi / 256Mi | Automated (Self-Heal) |
| **production** | `production` | 3 per service | 200m / 500m | 256Mi / 512Mi | Automated |

---

## Kubernetes Delivery Configurations

* **Automated Sync & Drift Correction:** Watched by ArgoCD controllers. Any manual configurations inside the live cluster are immediately overridden to preserve this repository's configurations.
* **Resiliency Probes:** Health monitoring is configured for all backend pods using standard Kubernetes Readiness and Liveness probes on `/health` (port `8001`).
* **Safe Rolling Upgrades:** Incorporates standard rolling updates with readiness delays configured to guarantee zero-downtime upgrades.
* **Namespace Isolation:** All deployments operate within namespaces dynamically configured by the `Platform-Infrastructure` repository.
