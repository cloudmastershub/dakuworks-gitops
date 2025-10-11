# DakuWorks GitOps Repository

This repository contains Kubernetes manifests for deploying DakuWorks application using GitOps practices with ArgoCD.

## Structure

```
gitops-repo/
├── argocd/
│   ├── projects/       # ArgoCD project definitions
│   └── applications/   # ArgoCD application definitions
├── apps/
│   ├── frontend/       # Frontend K8s manifests
│   └── backend/        # Backend K8s manifests + infrastructure
└── environments/
    ├── dev/           # Dev environment overlays
    ├── staging/       # Staging environment overlays
    └── prod/          # Production environment overlays
```

## Applications

### Frontend
- **Deployment**: Next.js app (2 replicas)
- **Service**: ClusterIP on port 80
- **Ingress**: Routes dakuworks.com/* to frontend

### Backend
- **Deployment**: Express API (2 replicas)
- **Service**: ClusterIP on port 80
- **Ingress**: Routes dakuworks.com/api/* to backend
- **Database**: PostgreSQL 14
- **Cache**: Redis 7

## Secrets Required

Before deploying, create the `dakuworks-secrets` secret in the `dakuworks-dev` namespace:

```bash
kubectl create secret generic dakuworks-secrets \
  --namespace=dakuworks-dev \
  --from-literal=database-user=dakuworks \
  --from-literal=database-password=your_db_password \
  --from-literal=jwt-secret=your_jwt_secret \
  --from-literal=email-user=your_email@gmail.com \
  --from-literal=email-password=your_app_password
```

## Deploying with ArgoCD

1. Apply the ArgoCD project:
```bash
kubectl apply -f argocd/projects/dakuworks.yaml
```

2. Apply the ArgoCD applications:
```bash
kubectl apply -f argocd/applications/frontend.yaml
kubectl apply -f argocd/applications/backend.yaml
```

ArgoCD will automatically sync and deploy the applications.

## Manual Deployment (without ArgoCD)

```bash
# Create namespace
kubectl create namespace dakuworks-dev

# Create secrets
kubectl create secret generic dakuworks-secrets --namespace=dakuworks-dev ...

# Deploy backend (includes postgres and redis)
kubectl apply -k apps/backend/

# Deploy frontend
kubectl apply -k apps/frontend/
```

## Updating Applications

### Update Image Tag

Edit the deployment.yaml file and update the image tag:
```yaml
image: mbuaku/dakuworks-backend:v1.2.3
```

Commit and push. ArgoCD will automatically sync the changes.

### Manual Sync

```bash
argocd app sync dakuworks-frontend
argocd app sync dakuworks-backend
```

## Monitoring

Check application status:
```bash
kubectl get all -n dakuworks-dev
```

Check ArgoCD sync status:
```bash
argocd app list
argocd app get dakuworks-frontend
argocd app get dakuworks-backend
```

## Traffic Flow

```
Internet
  → Cloudflared Tunnel
  → MetalLB LoadBalancer (192.168.101.200)
  → nginx-ingress
  → dakuworks.com/ → Frontend
  → dakuworks.com/api/ → Backend
```

## Notes

- All services use tier labels (frontend/backend) for organization
- Security contexts enforce non-root users
- Resource limits prevent resource exhaustion
- Health checks ensure pod availability
- Secrets are injected as environment variables

## License

MIT
