# GitOps Production Repository

This repository contains the Kubernetes deployment manifests for the **gitops-demo** application. It is the **single source of truth** for what runs in the Kubernetes cluster.

## How It Works

1. [Jenkins CI pipeline](https://github.com/brahmanyasudulagunta/gitops) builds a new Docker image on every code push
2. Jenkins updates the image tag in `canary.yaml` via automated commit
3. **ArgoCD** watches this repo and syncs changes to the Kubernetes cluster
4. After canary validation, the stable `deployment.yaml` is promoted manually

## Repository Structure

```
environments/
└── dev/
    ├── deployment.yaml    # Stable deployment (2 replicas)
    ├── canary.yaml        # Canary deployment (1 replica, gets new versions first)
    ├── service.yaml       # ClusterIP service (port 80 → 3000)
    ├── secret.yaml        # App secrets (manage via kubectl, not Git)
    ├── serviceacc.yaml    # Service account for pod identity
    ├── role.yaml          # RBAC role (least privilege)
    └── rolebinding.yaml   # Binds role to service account
```

## Prerequisites

Before deploying, create the `app-secrets` secret manually:

```bash
kubectl create secret generic app-secrets \
  --namespace=dev \
  --from-literal=APP_ENV=development
```

## Related Repository

- **CI Repository (source code):** [gitops](https://github.com/brahmanyasudulagunta/gitops)
