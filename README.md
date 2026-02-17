# gitops-prod

Kubernetes deployment manifests for the **gitops-demo** application across all environments. This repo is the **single source of truth** for what runs in the cluster — ArgoCD watches it and syncs automatically.

[![ArgoCD](https://img.shields.io/badge/CD-ArgoCD-EF7B4D?logo=argo&logoColor=white)](https://argoproj.github.io/cd)
[![Kubernetes](https://img.shields.io/badge/Runtime-Kubernetes-326CE5?logo=kubernetes&logoColor=white)](https://kubernetes.io)

---

## Role in the Platform

This is the **deployment layer** of a three-repo architecture:

| Repository | Role | Tools |
|------------|------|-------|
| [DevPlatform](https://github.com/brahmanyasudulagunta/DevPlatform) | Infrastructure provisioning & security | Terraform, Ansible, RBAC |
| [gitops](https://github.com/brahmanyasudulagunta/gitops) | Application development & CI | Node.js, Docker, Jenkins |
| **gitops-prod** (this) | Deployment manifests (GitOps) | Kubernetes manifests, ArgoCD |

### How It Connects

```
DevPlatform provisions namespaces (develop / staging / production)
    ↓
gitops CI pipeline builds image → auto-updates develop/canary.yaml here
    ↓
ArgoCD watches this repo → syncs manifests to the Kubernetes cluster
```

---

## Repository Structure

```
gitops-prod/
├── argocd/
│   ├── develop-app.yaml       # ArgoCD Application (auto-sync + self-heal)
│   ├── staging-app.yaml       # ArgoCD Application (auto-sync + self-heal)
│   └── production-app.yaml    # ArgoCD Application (manual sync only)
├── environments/
│   ├── develop/
│   │   ├── deployment.yaml    # Stable deployment (2 replicas)
│   │   ├── canary.yaml        # Canary deployment (1 replica, gets new versions first)
│   │   ├── service.yaml       # ClusterIP service (port 80 → 3000)
│   │   └── serviceaccount.yaml  # Pod identity for gitops-app-sa
│   ├── staging/
│   │   ├── deployment.yaml    # Stable deployment (2 replicas)
│   │   ├── service.yaml       # ClusterIP service (port 80 → 3000)
│   │   └── serviceaccount.yaml  # Pod identity for gitops-app-sa
│   └── production/
│       ├── deployment.yaml    # Stable deployment (3 replicas, higher resources)
│       ├── service.yaml       # ClusterIP service (port 80 → 3000)
│       └── serviceaccount.yaml  # Pod identity for gitops-app-sa
└── README.md
```

---

## Environments

| Environment | Namespace | Replicas (Stable) | Canary? | CPU Request/Limit | Memory Request/Limit |
|-------------|-----------|-------------------|---------|--------------------|----------------------|
| Develop | `develop` | 2 | ✅ 1 replica | 100m / 300m | 128Mi / 256Mi |
| Staging | `staging` | 2 | ❌ | 100m / 300m | 128Mi / 256Mi |
| Production | `production` | 3 | ❌ | 200m / 500m | 256Mi / 512Mi |

### Develop (Canary Strategy)

- **`canary.yaml`** — 1 replica that gets new image versions first (auto-updated by Jenkins CI)
- **`deployment.yaml`** — 2 replica stable deployment (manually promoted after canary validation)
- **`service.yaml`** — ClusterIP service that routes to both canary and stable pods (both use `app: gitops-demo` label)

### Staging

- **`deployment.yaml`** — 2 replica stable deployment (manually promoted from develop)
- **`service.yaml`** — ClusterIP service

### Production

- **`deployment.yaml`** — 3 replica stable deployment with higher resources and configured probe delays
- **`service.yaml`** — ClusterIP service
- Production probes include `initialDelaySeconds` and `periodSeconds` for safer rollouts

---

## Deployment Lifecycle

```
1. Developer pushes code to "gitops" repo
        ↓
2. Jenkins CI builds image → pushes to DockerHub (ashrith2727/gitops:<BUILD_NUMBER>)
        ↓
3. Jenkins auto-updates develop/canary.yaml with new image tag
        ↓
4. ArgoCD syncs canary deployment to "develop" namespace (1 replica)
        ↓
5. Validate canary in develop (check health, logs, metrics)
        ↓
6. Manually update develop/deployment.yaml → promote to stable (2 replicas)
        ↓
7. Manually update staging/deployment.yaml → promote to staging (2 replicas)
        ↓
8. Manually update production/deployment.yaml → promote to production (3 replicas)
```

---

## ArgoCD Configuration

Each environment needs its own ArgoCD Application pointing to the corresponding path:

| Application Name | Source Path | Target Namespace | Sync Policy |
|-----------------|------------|------------------|-------------|
| `gitops-demo-develop` | `environments/develop` | `develop` | Automated |
| `gitops-demo-staging` | `environments/staging` | `staging` | Automated |
| `gitops-demo-production` | `environments/production` | `production` | Manual |

> **Note:** Production uses manual sync to enforce approval before deploying to production.

---

## Kubernetes Features Used

- **Readiness & Liveness probes** on `/health` endpoint (port 3000)
- **Resource requests and limits** on all containers
- **Service accounts** (`gitops-app-sa`) for pod identity
- **Rolling updates** (default Kubernetes strategy)
- **Canary deployments** in develop for safe testing of new versions

---

## Prerequisites

The namespaces (`develop`, `staging`, `production`) must be provisioned by the [DevPlatform](https://github.com/brahmanyasudulagunta/DevPlatform) repo before deploying. DevPlatform also sets up RBAC and network policies for each namespace.
