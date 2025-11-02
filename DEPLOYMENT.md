# DakuWorks GitOps Deployment Guide

This repository contains Kubernetes manifests for deploying DakuWorks using GitOps with ArgoCD.

## 🚀 Quick Deployment

### Prerequisites

1. **ArgoCD installed** on the Kubernetes cluster
   - Access: http://argocd.elitessystems.com
   - Credentials managed by infrastructure team

2. **Secrets created** in the cluster (before ArgoCD sync):
   ```bash
   # Create the dakuworks-secrets (use strong passwords!)
   kubectl create secret generic dakuworks-secrets -n dakuworks-dev \
     --from-literal=database-user='dakuworks' \
     --from-literal=database-password='YOUR_SECURE_DB_PASSWORD' \
     --from-literal=jwt-secret='YOUR_JWT_SECRET_64_CHARS' \
     --from-literal=email-user='noreply@dakuworks.com' \
     --from-literal=email-password='YOUR_EMAIL_APP_PASSWORD'
   ```

### Deployment Steps

**Step 1: Apply ArgoCD Project**
```bash
kubectl apply -f argocd/projects/dakuworks.yaml
```

**Step 2: Apply ArgoCD Applications**
```bash
kubectl apply -f argocd/applications/backend.yaml
kubectl apply -f argocd/applications/frontend.yaml
```

**Step 3: Monitor Deployment**
```bash
# Watch ArgoCD sync status
kubectl get applications -n argocd -w

# Check application health
kubectl get application dakuworks-backend -n argocd -o jsonpath='{.status.sync.status}'
kubectl get application dakuworks-frontend -n argocd -o jsonpath='{.status.health.status}'
```

**Step 4: Verify Pods**
```bash
# Check all dakuworks pods
kubectl get pods -n dakuworks-dev

# Expected pods:
# - dakuworks-backend-xxx (2 replicas)
# - dakuworks-frontend-xxx (2 replicas)
# - postgres-xxx (1 replica)
# - redis-xxx (1 replica)
```

## 📁 Repository Structure

```
gitops-repo/
├── apps/
│   ├── namespace.yaml          # Namespace definition
│   ├── backend/                # Backend application
│   │   ├── deployment.yaml     # Backend deployment
│   │   ├── service.yaml        # Backend service
│   │   ├── ingress.yaml        # API ingress (/api/*)
│   │   ├── postgres.yaml       # PostgreSQL database
│   │   ├── redis.yaml          # Redis cache
│   │   ├── kustomization.yaml  # Kustomize config
│   │   └── secrets-template.yaml  # Secrets template (DO NOT COMMIT REAL SECRETS)
│   └── frontend/               # Frontend application
│       ├── deployment.yaml     # Frontend deployment
│       ├── service.yaml        # Frontend service
│       ├── ingress.yaml        # Frontend ingress (/)
│       └── kustomization.yaml  # Kustomize config
└── argocd/
    ├── projects/
    │   └── dakuworks.yaml      # ArgoCD project definition
    └── applications/
        ├── backend.yaml        # Backend ArgoCD app
        └── frontend.yaml       # Frontend ArgoCD app
```

## 🔄 GitOps Workflow

### Making Changes

1. **Update manifests** in this repository
2. **Commit and push** to GitHub
   ```bash
   git add .
   git commit -m "Update: description of changes"
   git push origin main
   ```
3. **ArgoCD automatically syncs** (within ~3 minutes)
4. **Monitor deployment**:
   - Via ArgoCD UI: http://argocd.elitessystems.com
   - Via kubectl: `kubectl get applications -n argocd`

### Updating Image Tags

After Jenkins builds new images:

```bash
# Update backend image
cd apps/backend
sed -i 's|image: mbuaku/dakuworks-backend:.*|image: mbuaku/dakuworks-backend:build-7|' deployment.yaml

# Update frontend image
cd apps/frontend
sed -i 's|image: mbuaku/dakuworks-frontend:.*|image: mbuaku/dakuworks-frontend:build-7|' deployment.yaml

# Commit and push
git add .
git commit -m "Update images to build-7"
git push

# ArgoCD will automatically deploy the new images
```

## 🔐 Secrets Management

### Required Secrets

DakuWorks requires the following secrets:

| Key | Description | Example |
|-----|-------------|---------|
| `database-user` | PostgreSQL username | `dakuworks` |
| `database-password` | PostgreSQL password | Generate with `openssl rand -base64 32` |
| `jwt-secret` | JWT signing secret | Generate with `openssl rand -base64 64` |
| `email-user` | SMTP email address | `noreply@dakuworks.com` |
| `email-password` | SMTP password/app password | Gmail App Password |

### Creating Secrets

**⚠️ Important**: Create secrets **before** deploying applications!

```bash
# Generate secure passwords
DB_PASSWORD=$(openssl rand -base64 32)
JWT_SECRET=$(openssl rand -base64 64)

# Create the secret
kubectl create secret generic dakuworks-secrets -n dakuworks-dev \
  --from-literal=database-user='dakuworks' \
  --from-literal=database-password="$DB_PASSWORD" \
  --from-literal=jwt-secret="$JWT_SECRET" \
  --from-literal=email-user='noreply@dakuworks.com' \
  --from-literal=email-password='YOUR_GMAIL_APP_PASSWORD'

# Verify secret was created
kubectl get secret dakuworks-secrets -n dakuworks-dev
```

### Updating Secrets

```bash
# Update a single key
kubectl patch secret dakuworks-secrets -n dakuworks-dev \
  --type merge \
  -p '{"stringData":{"email-password":"NEW_PASSWORD"}}'

# After updating secrets, restart pods to pick up changes
kubectl rollout restart deployment/dakuworks-backend -n dakuworks-dev
```

## 🌐 Ingress Configuration

DakuWorks uses the **shared NGINX Ingress Controller** at `192.168.101.200`.

### Routes

- **Frontend**: `dakuworks.com` → `dakuworks-frontend:80`
- **Backend API**: `dakuworks.com/api/*` → `dakuworks-backend:80`

### DNS Configuration

Add to `/etc/hosts` or DNS server:
```
192.168.101.200  dakuworks.com
```

### Testing Endpoints

```bash
# Frontend
curl -f http://dakuworks.com

# Backend health check
curl -f http://dakuworks.com/api/health

# Backend services endpoint
curl -f http://dakuworks.com/api/v1/services
```

## 📊 Monitoring Deployment

### Via kubectl

```bash
# Application sync status
kubectl get applications -n argocd

# Pod status
kubectl get pods -n dakuworks-dev

# Pod logs
kubectl logs -n dakuworks-dev -l app=dakuworks,tier=backend -f
kubectl logs -n dakuworks-dev -l app=dakuworks,tier=frontend -f

# Describe application for detailed status
kubectl describe application dakuworks-backend -n argocd
```

### Via ArgoCD UI

1. Open http://argocd.elitessystems.com
2. Login with admin credentials
3. View applications: `dakuworks-backend`, `dakuworks-frontend`
4. Check sync status, health, and deployment history

### Health Checks

```bash
# Check if all pods are running
kubectl get pods -n dakuworks-dev --no-headers | grep -v "Running"

# Check if ingress is configured
kubectl get ingress -n dakuworks-dev

# Test backend health endpoint
kubectl exec -n dakuworks-dev deployment/dakuworks-backend -- curl -f http://localhost:3000/health
```

## 🐛 Troubleshooting

### Application Not Syncing

```bash
# Check application status
kubectl describe application dakuworks-backend -n argocd

# Check ArgoCD logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller

# Manually trigger sync
kubectl patch application dakuworks-backend -n argocd \
  --type merge -p '{"operation":{"sync":{}}}'
```

### Pods Not Starting

```bash
# Check pod status and events
kubectl describe pod -n dakuworks-dev <pod-name>

# Check pod logs
kubectl logs -n dakuworks-dev <pod-name>

# Common issues:
# - Missing secrets: Create dakuworks-secrets
# - Image pull errors: Check Docker Hub credentials
# - Database connection: Check postgres pod is running
```

### Database Issues

```bash
# Check PostgreSQL pod
kubectl get pods -n dakuworks-dev -l app=postgres

# Connect to PostgreSQL
kubectl exec -it -n dakuworks-dev deployment/postgres -- psql -U dakuworks

# Check database logs
kubectl logs -n dakuworks-dev -l app=postgres
```

### Ingress Not Working

```bash
# Check ingress configuration
kubectl get ingress -n dakuworks-dev -o yaml

# Check NGINX ingress controller
kubectl get svc -n ingress-nginx

# Test ingress directly
curl -H "Host: dakuworks.com" http://192.168.101.200
```

## 🔄 Rollback

### Via Git

```bash
# Find the commit to rollback to
git log --oneline

# Revert to previous commit
git revert HEAD
git push

# ArgoCD will automatically deploy the previous version
```

### Via kubectl

```bash
# Rollback deployment
kubectl rollout undo deployment/dakuworks-backend -n dakuworks-dev

# Check rollout status
kubectl rollout status deployment/dakuworks-backend -n dakuworks-dev
```

## 📚 Additional Resources

- **Separation of Duties**: `/home/master/terraform/docs/SEPARATION-OF-DUTIES.md`
- **Universal Credentials**: `/home/master/terraform/docs/UNIVERSAL-CREDENTIALS-GUIDE.md`
- **GitOps Guide**: `/home/master/terraform/docs/GITOPS.md`
- **ArgoCD Docs**: https://argo-cd.readthedocs.io/

---

**Maintained by**: Daku Deploy Team
**Infrastructure managed by**: Claude Infra
**Last Updated**: November 2, 2025
