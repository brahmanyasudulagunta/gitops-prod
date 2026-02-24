# gitops-prod

Kubernetes deployment manifests for the **gitops** application across all environments. This repo is the **single source of truth** for what runs in the cluster — ArgoCD watches it and syncs automatically.

[![ArgoCD](https://img.shields.io/badge/CD-ArgoCD-EF7B4D?style=for-the-badge&logo=argo&logoColor=white)](https://argoproj.github.io/cd)
[![Kubernetes](https://img.shields.io/badge/Runtime-Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)](https://kubernetes.io)

---

## Architecture (3-Repo GitOps)

| Repository | Role | Tools |
|------------|------|-------|
| [DevPlatform](https://github.com/brahmanyasudulagunta/DevPlatform) | Infrastructure provisioning & security | Terraform, Ansible, RBAC |
| [gitops](https://github.com/brahmanyasudulagunta/gitops) | Platform Control Plane (UI + API) | FastAPI, React, PostgreSQL |
| **gitops-prod** (this) | Deployment manifests (GitOps) | Kubernetes manifests, ArgoCD |

```
DevPlatform provisions namespaces (develop / production)
    ↓
Jenkins CI builds image → auto-updates develop/develop.yaml here
    ↓
ArgoCD watches this repo → syncs manifests to the Kubernetes cluster
```

---

## Repository Structure

```
gitops-prod/
├── environments/
│   ├── develop/
│   │   ├── develop.yaml        # Deployment (2 replicas)
│   │   ├── service.yaml        # ClusterIP service (port 80 → 3002)
│   │   └── serviceaccount.yaml # Pod identity (gitops-sa)
│   └── production/
│       ├── deployment.yaml     # Deployment (3 replicas, higher resources)
│       ├── service.yaml        # ClusterIP service (port 80 → 3002)
│       └── serviceaccount.yaml # Pod identity (gitops-sa)
└── README.md
```

---

## Environments

| Environment | Namespace | Replicas | CPU Request/Limit | Memory Request/Limit |
|-------------|-----------|----------|--------------------|----------------------|
| Develop | `develop` | 2 | 100m / 300m | 128Mi / 256Mi |
| Production | `production` | 3 | 200m / 500m | 256Mi / 512Mi |

---

## Deployment Lifecycle

```
1. Developer pushes code to "gitops" repo
        ↓
2. Jenkins CI builds image → pushes to DockerHub (ashrith2727/gitops:<BUILD_NUMBER>)
        ↓
3. Jenkins auto-updates develop/develop.yaml with new image tag
        ↓
4. ArgoCD syncs deployment to "develop" namespace
        ↓
5. Validate in develop (check health, logs, metrics)
        ↓
6. Manually update production/deployment.yaml → promote to production (3 replicas)
```

---

## ArgoCD Applications

| Application Name | Source Path | Target Namespace | Sync Policy |
|-----------------|------------|------------------|-------------|
| `gitops-develop` | `environments/develop` | `develop` | Automated (auto-sync + self-heal) |
| `gitops-production` | `environments/production` | `production` | Automated |

---

## Kubernetes Features

- **Readiness & Liveness probes** on `/health` endpoint (port 3002)
- **Resource requests and limits** on all containers
- **Service accounts** (`gitops-sa`) for pod identity
- **Rolling updates** (default Kubernetes strategy)
- **Production probe delays** — `initialDelaySeconds` and `periodSeconds` for safer rollouts

---

## Prerequisites

The namespaces (`develop`, `production`) must be provisioned by the [DevPlatform](https://github.com/brahmanyasudulagunta/DevPlatform) repo before deploying. DevPlatform sets up RBAC, network policies, and resource quotas for each namespace.
