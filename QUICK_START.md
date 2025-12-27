# Growth Engine GitOps - Quick Start

## TL;DR

```bash
# Non-Prod Setup (Dev, QA, Demo)
argocd login argocd-dev.equitybrix.net
kubectl apply -f argocd/projects/growth-engine.yaml
kubectl apply -f argocd/applications/growth-engine-dev.yaml
kubectl apply -f argocd/applications/growth-engine-qa.yaml
kubectl apply -f argocd/applications/growth-engine-demo.yaml

# Production Setup
argocd login argocd-prod.equitybrix.com
kubectl apply -f argocd/projects/growth-engine-prod.yaml
kubectl apply -f argocd/applications/growth-engine-prod.yaml
```

## Repository URLs

- **GitOps Repo**: https://github.com/equitybrix/growth-engine-gitops
- **Main Repo**: https://github.com/equitybrix/growth-engine

## ArgoCD Instances

| Environment | ArgoCD URL | Auto-Sync |
|-------------|------------|-----------|
| Dev, QA, Demo | https://argocd-dev.equitybrix.net/ | Yes |
| Production | https://argocd-prod.equitybrix.com/ | No (Manual) |

## Application URLs

| Environment | URL | Namespace |
|-------------|-----|-----------|
| **Dev** | https://dev.growth-engine.equitybrix.net | growth-engine-dev |
| **QA** | https://qa.growth-engine.equitybrix.net | growth-engine-qa |
| **Demo** | https://demo.growth-engine.equitybrix.net | growth-engine-demo |
| **Prod** | https://growth-engine.equitybrix.com | growth-engine-prod |

## Common Commands

### Deploy to Environment

```bash
# Dev (auto-syncs)
argocd app sync growth-engine-dev

# QA
argocd app sync growth-engine-qa

# Demo
argocd app sync growth-engine-demo

# Production (requires manual approval)
argocd app sync growth-engine-prod
```

### Check Status

```bash
# List all applications
argocd app list

# Get application details
argocd app get growth-engine-dev

# View differences
argocd app diff growth-engine-dev
```

### Rollback

```bash
# View history
argocd app history growth-engine-prod

# Rollback to revision
argocd app rollback growth-engine-prod 5
```

### Update Image Tag

```bash
# Update dev environment
cd environments/dev
kustomize edit set image ghcr.io/equitybrix/growth-engine=ghcr.io/equitybrix/growth-engine:sha-abc123
git add . && git commit -m "Update dev image" && git push
```

## Environment Configurations

| Environment | Replicas | Resources | Components |
|-------------|----------|-----------|------------|
| **Dev** | 1 | 256Mi/100m | Monitoring |
| **QA** | 2 | 512Mi/250m | Monitoring |
| **Demo** | 2 | 512Mi/250m | Monitoring |
| **Prod** | 3 (HPA 2-10) | 1Gi/500m | Monitoring, Autoscaling, Security, PDB |

## Secrets Required

Create these secrets in each namespace:

```bash
kubectl create secret generic growth-engine-secrets \
  --from-literal=MONGODB_URL='...' \
  --from-literal=REDIS_URL='...' \
  --from-literal=LITELLM_API_KEY='...' \
  -n growth-engine-{dev|qa|demo|prod}
```

## Deployment Flow

```
Developer → Merge to main → CI Pipeline → GHCR
                                          ↓
                                    Update GitOps Repo
                                          ↓
                              ArgoCD Auto-Sync (Non-Prod)
                                          ↓
                              ArgoCD Manual-Sync (Prod)
```

## Tag-Based Deployments

| Git Tag | Target Environment | Example |
|---------|-------------------|---------|
| `sha-<hash>` | Dev (from main) | `sha-abc123` |
| `qa-v*` | QA | `qa-v1.0.0` |
| `demo-v*` | Demo | `demo-v1.0.0` |
| `v*` | Production | `v1.0.0` |

## Troubleshooting

```bash
# Application won't sync
argocd app get growth-engine-dev --hard-refresh
argocd app diff growth-engine-dev

# Pods not starting
kubectl get events -n growth-engine-dev --sort-by='.lastTimestamp'
kubectl describe pod <pod-name> -n growth-engine-dev
kubectl logs -f deployment/growth-engine-api -n growth-engine-dev

# Check secrets
kubectl get secrets -n growth-engine-dev
```

## Support

- **Documentation**: [DEPLOYMENT_GUIDE.md](DEPLOYMENT_GUIDE.md)
- **Issues**: #devops-support
- **Incidents**: PagerDuty on-call
