# Argo CD Image Updater Configuration

This document explains how Argo CD Image Updater is configured for Growth Engine to automatically detect and deploy new container images.

## Overview

Argo CD Image Updater monitors container registries for new images and automatically updates the GitOps repository when new versions are available. This enables continuous deployment without manual intervention.

## Architecture

```
GitHub Actions (CI)
    ↓
Build & Push Image to GHCR
    ↓
GitHub Container Registry (ghcr.io)
    ↓
Argo CD Image Updater (polls every 2 min)
    ↓
Detects New Image Tag
    ↓
Updates Kustomization.yaml in GitOps Repo
    ↓
Commits & Pushes to main branch
    ↓
ArgoCD Detects Git Change
    ↓
Auto-Sync to Kubernetes
```

## Configuration by Environment

### Development Environment

**Strategy**: `latest`
**Tag Pattern**: `sha-[a-f0-9]{7}` (e.g., `sha-abc123`)
**Behavior**: Automatically updates to the latest commit hash from main branch

```yaml
annotations:
  argocd-image-updater.argoproj.io/image-list: api=ghcr.io/equitybrix/growth-engine
  argocd-image-updater.argoproj.io/api.update-strategy: latest
  argocd-image-updater.argoproj.io/api.allow-tags: regexp:^sha-[a-f0-9]{7}$
  argocd-image-updater.argoproj.io/write-back-method: git
```

**When it updates:**
- Every time CI pushes a new `sha-*` tagged image to GHCR
- Typically after every merge to `main` branch

### QA Environment

**Strategy**: `semver`
**Tag Pattern**: `qa-v[0-9]+\.[0-9]+\.[0-9]+` (e.g., `qa-v1.2.0`)
**Behavior**: Automatically updates to the latest QA semver tag

```yaml
annotations:
  argocd-image-updater.argoproj.io/image-list: api=ghcr.io/equitybrix/growth-engine
  argocd-image-updater.argoproj.io/api.update-strategy: semver
  argocd-image-updater.argoproj.io/api.allow-tags: regexp:^qa-v[0-9]+\.[0-9]+\.[0-9]+$
  argocd-image-updater.argoproj.io/write-back-method: git
```

**When it updates:**
- When you create a new `qa-v*` tag (e.g., `git tag qa-v1.2.0`)
- Follows semantic versioning rules (qa-v1.2.1 > qa-v1.2.0)

### Demo Environment

**Strategy**: `semver`
**Tag Pattern**: `demo-v[0-9]+\.[0-9]+\.[0-9]+` (e.g., `demo-v1.2.0`)
**Behavior**: Automatically updates to the latest Demo semver tag

```yaml
annotations:
  argocd-image-updater.argoproj.io/image-list: api=ghcr.io/equitybrix/growth-engine
  argocd-image-updater.argoproj.io/api.update-strategy: semver
  argocd-image-updater.argoproj.io/api.allow-tags: regexp:^demo-v[0-9]+\.[0-9]+\.[0-9]+$
  argocd-image-updater.argoproj.io/write-back-method: git
```

**When it updates:**
- When you create a new `demo-v*` tag (e.g., `git tag demo-v1.2.0`)

### Production Environment

**Strategy**: `digest`
**Tag Pattern**: `v[0-9]+\.[0-9]+\.[0-9]+` (e.g., `v1.2.0`)
**Behavior**: Digest-based updates only (effectively manual control)

```yaml
annotations:
  argocd-image-updater.argoproj.io/image-list: api=ghcr.io/equitybrix/growth-engine
  argocd-image-updater.argoproj.io/api.update-strategy: semver
  argocd-image-updater.argoproj.io/api.allow-tags: regexp:^v[0-9]+\.[0-9]+\.[0-9]+$
  argocd-image-updater.argoproj.io/update-strategy: digest
  argocd-image-updater.argoproj.io/write-back-method: git
```

**When it updates:**
- Only when image digest changes (rare)
- Effectively requires manual sync via ArgoCD
- Provides safety for production deployments

## Git Write-Back Configuration

Image Updater commits changes back to this repository with the following format:

```
chore: Update growth-engine-dev image to sha-abc123

- Application: growth-engine-dev
- Image: ghcr.io/equitybrix/growth-engine
- Old Tag: sha-xyz789
- New Tag: sha-abc123

Updated by Argo CD Image Updater
```

**Git Configuration:**
- **Author**: argocd-image-updater
- **Email**: argocd-image-updater@equitybrix.com
- **Branch**: main

## Prerequisites

### 1. GitHub Token Secret

Image Updater needs a GitHub token with registry read permissions and repo write permissions:

```bash
# Create secret in argocd namespace
kubectl create secret generic github-token \
  --from-literal=token=ghp_xxxxxxxxxxxx \
  -n argocd

# Or update existing argocd-secret
kubectl patch secret argocd-secret \
  -n argocd \
  --type merge \
  -p '{"data":{"github.token":"'$(echo -n 'ghp_xxxxxxxxxxxx' | base64)'"}}'
```

### 2. Registry Credentials

Configure GHCR access in ArgoCD:

```bash
argocd repo add ghcr.io/equitybrix/growth-engine \
  --type git \
  --username <github-username> \
  --password <github-token>
```

## Monitoring Image Updater

### Check Image Updater Status

```bash
# View Image Updater logs
kubectl logs -f deployment/argocd-image-updater -n argocd

# Check application annotations
kubectl get app growth-engine-dev -n argocd -o yaml | grep image-updater

# View last updated image
argocd app get growth-engine-dev -o json | jq '.status.summary.images'
```

### Force Image Update Check

```bash
# Trigger immediate check (instead of waiting for poll interval)
argocd app get growth-engine-dev --refresh

# Or restart Image Updater
kubectl rollout restart deployment/argocd-image-updater -n argocd
```

### View Git Commits from Image Updater

```bash
# In this repository
git log --author="argocd-image-updater" --oneline

# See what Image Updater changed
git show <commit-hash>
```

## Deployment Workflow Examples

### Example 1: Deploy to Dev (Automatic)

```bash
# Developer merges PR to main
git checkout main
git pull
# ... make changes ...
git commit -m "feat: add new feature"
git push origin main

# CI/CD pipeline runs:
# 1. Builds Docker image
# 2. Pushes to ghcr.io/equitybrix/growth-engine:sha-abc123
# 3. Pushes to ghcr.io/equitybrix/growth-engine:latest

# Image Updater (within 2 minutes):
# 1. Detects new sha-abc123 tag
# 2. Updates environments/dev/kustomization.yaml
# 3. Commits: "chore: Update growth-engine-dev image to sha-abc123"
# 4. Pushes to main branch

# ArgoCD (within 3 minutes):
# 1. Detects Git change
# 2. Syncs growth-engine-dev application
# 3. Deploys to Kubernetes

# Total time: ~5 minutes from merge to deployment
```

### Example 2: Deploy to QA (Semi-Automatic)

```bash
# Create QA tag
git tag qa-v1.2.0
git push origin qa-v1.2.0

# CI/CD pipeline:
# 1. Builds image with tag qa-v1.2.0
# 2. Pushes to ghcr.io/equitybrix/growth-engine:qa-v1.2.0

# Image Updater (within 2 minutes):
# 1. Detects new qa-v1.2.0 tag
# 2. Compares with current version using semver
# 3. Updates environments/qa/kustomization.yaml
# 4. Commits and pushes change

# ArgoCD:
# 1. Auto-syncs to QA environment
```

### Example 3: Deploy to Production (Manual)

```bash
# Create production release
gh release create v1.2.0 --title "Release v1.2.0"

# CI/CD pipeline:
# 1. Builds image with tag v1.2.0
# 2. Pushes to GHCR

# Image Updater:
# Does NOT auto-update due to digest strategy

# Manual deployment:
argocd app get growth-engine-prod
argocd app sync growth-engine-prod
```

## Troubleshooting

### Image Not Updating

**Symptoms**: New image pushed to GHCR but environment not updating

**Checks**:
1. Verify tag matches pattern:
   ```bash
   # List tags in registry
   docker images ghcr.io/equitybrix/growth-engine --format "{{.Tag}}"

   # Check pattern
   echo "sha-abc123" | grep -E '^sha-[a-f0-9]{7}$'
   ```

2. Check Image Updater logs:
   ```bash
   kubectl logs deployment/argocd-image-updater -n argocd | grep growth-engine
   ```

3. Verify registry credentials:
   ```bash
   kubectl get secret github-token -n argocd -o yaml
   ```

### Git Commit Failures

**Symptoms**: Image Updater detects update but fails to commit

**Checks**:
1. Verify GitHub token permissions
2. Check branch protection rules
3. View Image Updater logs for Git errors

```bash
kubectl logs deployment/argocd-image-updater -n argocd | grep -i "git\|commit\|push"
```

### Wrong Version Selected

**Symptoms**: Image Updater selects unexpected version

**Debug**:
```bash
# List all tags matching pattern
argocd app get growth-engine-qa -o json | jq '.spec.source.targetRevision'

# Check semver comparison
# Image Updater follows semver rules: v1.2.1 > v1.2.0 > v1.1.9
```

## Configuration Reference

### Update Strategies

| Strategy | Description | Example |
|----------|-------------|---------|
| `latest` | Most recently pushed image | sha-abc123 |
| `semver` | Semantic versioning (highest version) | v1.2.3 |
| `digest` | Image digest (manual control) | sha256:abc... |
| `name` | Alphabetical order | build-123 |

### Tag Pattern Examples

```yaml
# Match commit hashes
regexp:^sha-[a-f0-9]{7}$

# Match semver with prefix
regexp:^v[0-9]+\.[0-9]+\.[0-9]+$

# Match qa tags
regexp:^qa-v[0-9]+\.[0-9]+\.[0-9]+$

# Match any tag starting with "prod-"
regexp:^prod-.*$
```

## Disabling Image Updater

To temporarily disable Image Updater for an application:

```yaml
# Add this annotation
argocd-image-updater.argoproj.io/image-list: ""
```

Or remove all `argocd-image-updater.argoproj.io/*` annotations.

## References

- [Argo CD Image Updater Docs](https://argocd-image-updater.readthedocs.io/)
- [GitHub Container Registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)
- [Semantic Versioning](https://semver.org/)
