# Growth Engine Deployment Guide

This guide walks you through setting up ArgoCD for the Growth Engine project with separate non-prod and prod instances.

## Prerequisites

- Access to ArgoCD Non-Prod: https://argocd-dev.equitybrix.net/
- Access to ArgoCD Prod: https://argocd-prod.equitybrix.com/
- `argocd` CLI installed
- `kubectl` access to both clusters
- Repository access to `equitybrix/growth-engine-gitops`

---

## Step 1: Setup ArgoCD Non-Prod (Dev, QA, Demo)

### Login to ArgoCD Non-Prod

```bash
# Login via CLI
argocd login argocd-dev.equitybrix.net

# Or login via kubectl port-forward if needed
kubectl port-forward svc/argocd-server -n argocd 8080:443
argocd login localhost:8080
```

### Apply Growth Engine Project

```bash
# Apply the project definition
kubectl apply -f argocd/projects/growth-engine.yaml
```

### Create Dev Environment Application

```bash
# Apply dev application
kubectl apply -f argocd/applications/growth-engine-dev.yaml

# Verify application is created
argocd app list | grep growth-engine-dev

# Sync the application
argocd app sync growth-engine-dev

# Watch the sync progress
argocd app wait growth-engine-dev --timeout 600
```

### Create QA Environment Application

```bash
# Apply QA application
kubectl apply -f argocd/applications/growth-engine-qa.yaml

# Sync QA
argocd app sync growth-engine-qa
argocd app wait growth-engine-qa --timeout 600
```

### Create Demo Environment Application

```bash
# Apply demo application
kubectl apply -f argocd/applications/growth-engine-demo.yaml

# Sync demo
argocd app sync growth-engine-demo
argocd app wait growth-engine-demo --timeout 600
```

### Verify Non-Prod Deployments

```bash
# Check all applications
argocd app list

# Check dev environment
argocd app get growth-engine-dev

# Verify pods are running
kubectl get pods -n growth-engine-dev
kubectl get pods -n growth-engine-qa
kubectl get pods -n growth-engine-demo

# Check services
kubectl get svc -n growth-engine-dev
kubectl get svc -n growth-engine-qa
kubectl get svc -n growth-engine-demo

# Check ingresses
kubectl get ingress -n growth-engine-dev
kubectl get ingress -n growth-engine-qa
kubectl get ingress -n growth-engine-demo
```

---

## Step 2: Setup ArgoCD Prod (Production)

### Login to ArgoCD Prod

```bash
# Login to production ArgoCD instance
argocd login argocd-prod.equitybrix.com
```

### Apply Growth Engine Production Project

```bash
# Apply the production project definition
kubectl apply -f argocd/projects/growth-engine-prod.yaml
```

### Create Production Application

```bash
# Apply production application
kubectl apply -f argocd/applications/growth-engine-prod.yaml

# Verify application is created
argocd app list | grep growth-engine-prod

# Get application details
argocd app get growth-engine-prod
```

**Note:** Production uses **manual sync** - it will not auto-deploy.

### Deploy to Production (Manual)

```bash
# Sync production manually
argocd app sync growth-engine-prod

# Watch the sync progress
argocd app wait growth-engine-prod --timeout 600

# Verify deployment
kubectl get pods -n growth-engine-prod
kubectl get svc -n growth-engine-prod
kubectl get ingress -n growth-engine-prod
```

### Verify Production Deployment

```bash
# Check application status
argocd app get growth-engine-prod

# Check deployment health
kubectl get deployment growth-engine-api -n growth-engine-prod

# Check HPA (autoscaling)
kubectl get hpa -n growth-engine-prod

# Check PodDisruptionBudget
kubectl get pdb -n growth-engine-prod

# View logs
kubectl logs -f deployment/growth-engine-api -n growth-engine-prod
```

---

## Step 3: Create Kubernetes Secrets

Before the applications can fully function, you need to create secrets in each namespace.

### Create Secrets for Dev Environment

```bash
# Create secret
kubectl create secret generic growth-engine-secrets \
  --from-literal=MONGODB_URL='mongodb://mongodb.shared-services.svc.cluster.local:27017/growth_engine_dev' \
  --from-literal=REDIS_URL='redis://redis.shared-services.svc.cluster.local:6379/0' \
  --from-literal=LITELLM_API_KEY='your-litellm-api-key' \
  -n growth-engine-dev
```

### Create Secrets for QA Environment

```bash
kubectl create secret generic growth-engine-secrets \
  --from-literal=MONGODB_URL='mongodb://mongodb.shared-services.svc.cluster.local:27017/growth_engine_qa' \
  --from-literal=REDIS_URL='redis://redis.shared-services.svc.cluster.local:6379/1' \
  --from-literal=LITELLM_API_KEY='your-litellm-api-key' \
  -n growth-engine-qa
```

### Create Secrets for Demo Environment

```bash
kubectl create secret generic growth-engine-secrets \
  --from-literal=MONGODB_URL='mongodb://mongodb.shared-services.svc.cluster.local:27017/growth_engine_demo' \
  --from-literal=REDIS_URL='redis://redis.shared-services.svc.cluster.local:6379/2' \
  --from-literal=LITELLM_API_KEY='your-litellm-api-key' \
  -n growth-engine-demo
```

### Create Secrets for Production Environment

```bash
kubectl create secret generic growth-engine-secrets \
  --from-literal=MONGODB_URL='mongodb://prod-mongodb.production.svc.cluster.local:27017/growth_engine_prod' \
  --from-literal=REDIS_URL='redis://prod-redis.production.svc.cluster.local:6379/0' \
  --from-literal=LITELLM_API_KEY='your-production-litellm-api-key' \
  -n growth-engine-prod
```

**Recommended:** Use External Secrets Operator or Sealed Secrets for production.

---

## Step 4: Verify All Environments

### Health Checks

```bash
# Dev
curl https://dev.growth-engine.equitybrix.net/health

# QA
curl https://qa.growth-engine.equitybrix.net/health

# Demo
curl https://demo.growth-engine.equitybrix.net/health

# Production
curl https://growth-engine.equitybrix.com/health
```

### ArgoCD Dashboard

Visit the ArgoCD dashboards to see application status:

- **Non-Prod**: https://argocd-dev.equitybrix.net/applications
- **Prod**: https://argocd-prod.equitybrix.com/applications

---

## Step 5: CI/CD Integration

The CI/CD pipeline will automatically update image tags in this repository. Configure GitHub Actions secrets:

```bash
# In growth-engine repository, add GitHub secret:
# GITOPS_TOKEN - Personal access token with repo write access
```

When CI/CD runs, it will:
1. Build and push Docker image to GHCR
2. Update `environments/{env}/kustomization.yaml` with new image tag
3. Commit and push to this repo
4. ArgoCD auto-syncs (except production)

---

## Common Operations

### Update Image Tag Manually

```bash
cd environments/dev
kustomize edit set image ghcr.io/equitybrix/growth-engine=ghcr.io/equitybrix/growth-engine:sha-abc123

git add .
git commit -m "Update dev to sha-abc123"
git push origin main
```

### Rollback Deployment

```bash
# View history
argocd app history growth-engine-prod

# Rollback to specific revision
argocd app rollback growth-engine-prod <REVISION_NUMBER>
```

### View Application Logs

```bash
# ArgoCD application logs
argocd app logs growth-engine-dev

# Pod logs
kubectl logs -f deployment/growth-engine-api -n growth-engine-dev
```

### Force Sync

```bash
# Force sync with prune
argocd app sync growth-engine-dev --prune --force
```

---

## Troubleshooting

### Application Out of Sync

```bash
# View differences
argocd app diff growth-engine-dev

# Force hard refresh
argocd app get growth-engine-dev --hard-refresh
```

### Pods Not Starting

```bash
# Check events
kubectl get events -n growth-engine-dev --sort-by='.lastTimestamp'

# Describe pod
kubectl describe pod <pod-name> -n growth-engine-dev

# Check secrets exist
kubectl get secrets -n growth-engine-dev
```

### Image Pull Errors

```bash
# Verify image exists in GHCR
docker pull ghcr.io/equitybrix/growth-engine:latest

# Check image pull secret (if using private registry)
kubectl get secret -n growth-engine-dev
```

---

## Security Notes

1. **Never commit secrets** to this repository
2. Use External Secrets Operator or Sealed Secrets for production
3. Production requires manual sync approval
4. Network policies are enabled in production
5. All pods run as non-root with read-only filesystem

---

## Support

- **ArgoCD Issues**: #platform-team
- **Deployment Issues**: #devops-support
- **Production Incidents**: PagerDuty on-call
