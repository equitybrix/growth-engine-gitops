# Growth Engine GitOps Repository

This repository contains Kubernetes manifests for deploying the Growth Engine platform using GitOps principles with ArgoCD.

## Repository Structure

```
growth-engine-gitops/
├── argocd/                    # ArgoCD configuration
│   ├── applications/          # Application definitions
│   └── projects/              # Project definitions
├── base/                      # Base Kubernetes manifests
├── components/                # Reusable components
│   ├── monitoring/            # Prometheus ServiceMonitor, alerts
│   ├── autoscaling/           # HPA configuration
│   └── security/              # Network policies, PSP
└── environments/              # Environment-specific overlays
    ├── dev/                   # Development environment
    ├── qa/                    # QA environment
    ├── demo/                  # Demo environment
    └── prod/                  # Production environment
```

## Environments

| Environment | Namespace | ArgoCD Instance | URL |
|-------------|-----------|-----------------|-----|
| **dev** | growth-engine-dev | ArgoCD Non-Prod | https://argocd-dev.equitybrix.net/ |
| **qa** | growth-engine-qa | ArgoCD Non-Prod | https://argocd-dev.equitybrix.net/ |
| **demo** | growth-engine-demo | ArgoCD Non-Prod | https://argocd-dev.equitybrix.net/ |
| **prod** | growth-engine-prod | ArgoCD Prod | https://argocd-prod.equitybrix.com/ |

## Deployment Workflow

### Non-Prod Environments (Dev, QA, Demo)

1. **Dev**: Auto-deploys on merge to `main` branch
   ```bash
   git tag qa-v1.0.0
   git push origin qa-v1.0.0
   ```

2. **QA**: Deploy via `qa-v*` tag
   ```bash
   git tag qa-v1.0.0
   git push origin qa-v1.0.0
   ```

3. **Demo**: Deploy via `demo-v*` tag
   ```bash
   git tag demo-v1.0.0
   git push origin demo-v1.0.0
   ```

### Production Environment

Deploy via `v*` tag (requires manual approval):
```bash
gh release create v1.0.0 \
    --title "Release v1.0.0" \
    --notes "Production release notes"
```

## ArgoCD Setup

### Non-Prod Cluster
```bash
# Login to non-prod ArgoCD
argocd login argocd-dev.equitybrix.net

# Apply project
kubectl apply -f argocd/projects/growth-engine.yaml

# Apply applications
kubectl apply -f argocd/applications/growth-engine-dev.yaml
kubectl apply -f argocd/applications/growth-engine-qa.yaml
kubectl apply -f argocd/applications/growth-engine-demo.yaml
```

### Prod Cluster
```bash
# Login to prod ArgoCD
argocd login argocd-prod.equitybrix.com

# Apply project
kubectl apply -f argocd/projects/growth-engine.yaml

# Apply application
kubectl apply -f argocd/applications/growth-engine-prod.yaml
```

## Manual Deployment

You can manually sync applications using ArgoCD CLI:

```bash
# Sync dev environment
argocd app sync growth-engine-dev

# Sync production (with confirmation)
argocd app sync growth-engine-prod --prune
```

## Rollback

```bash
# View deployment history
argocd app history growth-engine-prod

# Rollback to specific revision
argocd app rollback growth-engine-prod <REVISION_NUMBER>
```

## Contributing

1. All changes should go through the main `growth-engine` repository
2. CI/CD pipeline automatically updates this GitOps repo
3. Never commit directly to this repository manually
4. For emergency fixes, create a PR and get approval

## Contact

- **DevOps Team**: #devops-support
- **Platform Team**: #platform-support
