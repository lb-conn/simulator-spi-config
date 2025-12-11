# Simulator-SPI GitOps Configuration

GitOps repository for managing Simulator-SPI deployments across multiple environments using ArgoCD and Kustomize.

## 📁 Repository Structure

```
simulator-spi-config/
├── argocd-apps/              # ArgoCD Application definitions
│   ├── dev.yaml              # Dev environment Application
│   ├── qa.yaml               # QA environment Application
│   ├── staging.yaml          # Staging environment Application
│   └── prd.yaml              # Production environment Application
├── base/                     # Base Kubernetes manifests
│   ├── kustomization.yaml
│   ├── shared-configmap.yaml
│   ├── simulator-spi-deployment.yaml
│   └── simulator-spi-service.yaml
└── environments/             # Environment-specific overlays
    ├── dev/
    │   ├── kustomization.yaml
    │   └── shared-configmap.yaml
    ├── qa/
    │   ├── kustomization.yaml
    │   └── shared-configmap.yaml
    ├── staging/
    │   ├── kustomization.yaml
    │   └── shared-configmap.yaml
    └── prd/
        ├── kustomization.yaml
        └── shared-configmap.yaml
```

## 🏗️ Architecture

### Application

- **simulator-spi**: SPI simulator service for lb-conn
- Deploy to namespace: `simulator-spi`
- Exposes HTTP port: 3000
- Pull images from GHCR: `ghcr.io/lb-conn/simulator-spi/spi`

### Environments

| Environment | Replicas | Resources | Sync Policy | Project |
|-------------|----------|-----------|-------------|---------|
| **Dev** | 1 | 50m CPU / 200Mi RAM | Automated | default |
| **QA** | 1 | 100m CPU / 300Mi RAM | Automated | default |
| **Staging** | 2 | 200m CPU / 512Mi RAM | Manual | default |
| **Production** | 2 | 200m CPU / 512Mi RAM | Manual | default |

## 🚀 Deployment Flow

```
main branch → DEV (auto)
    ↓
   QA tag → QA (auto after approval)
    ↓
  STG tag → STG (PR → approval → merge → manual sync)
    ↓
  PRD tag → PRD (PR → approval → merge → manual sync)
```

### 1. Development Environment

**Trigger**: Push to `main` branch in simulator-spi repo

**Process**:
1. CI/CD builds Docker images with tag `main-{sha}`
2. GitHub Actions updates `environments/dev/kustomization.yaml`
3. Commits directly to main branch
4. ArgoCD auto-syncs (prune + selfHeal enabled)

**Deployment Name**: `simulator-spi`

### 2. QA Environment

**Trigger**: Manual promotion workflow or `qa-*` tag

**Process**:
1. Validate images exist in GHCR
2. Update `environments/qa/kustomization.yaml` with `qa-{sha}` tag
3. Direct commit to main
4. ArgoCD auto-syncs

**Deployment Name**: `simulator-spi`

### 3. Staging Environment

**Trigger**: Manual promotion workflow or `stg-*` tag

**Process**:
1. Create Pull Request updating `environments/staging/kustomization.yaml`
2. Require approvals (QA Team + Tech Lead)
3. After merge, manually trigger ArgoCD sync
4. 2 replicas for HA
5. Production-mirror configuration

**Deployment Name**: `simulator-spi`

### 4. Production Environment

**Trigger**: Manual promotion workflow or `prd-*` tag

**Process**:
1. Create Pull Request updating `environments/prd/kustomization.yaml`
2. Require approvals (Tech Lead + Manager)
3. After merge, manually trigger ArgoCD sync
4. 2 replicas for HA

**Deployment Name**: `simulator-spi`

## 📦 How to Update

### Update Dev (Automatic)

Automated via CI/CD workflow after successful build on `main` branch.

### Update QA (Manual)

**Using GitHub CLI:**
```bash
# In simulator-spi repository
# Promote specific commit
gh workflow run promote-qa.yml -f COMMIT_SHA=abc123def456

# Or promote latest main
gh workflow run promote-qa.yml

# Or use specific image tag
gh workflow run promote-qa.yml -f IMAGE_TAG=abc123de
```

**What happens:**
- ✅ Validates image exists in GHCR
- ✅ Updates `environments/qa/kustomization.yaml` directly
- ✅ Commits to main branch
- ✅ ArgoCD auto-syncs (~ 3 minutes)

### Update Staging (Manual via PR)

**Using GitHub CLI:**
```bash
# In simulator-spi repository
# Promote specific commit
gh workflow run promote-staging.yml -f COMMIT_SHA=abc123def456
```

**After PR is merged:**
```bash
# Manually trigger ArgoCD sync
argocd app sync simulator-spi-staging
```

### Update Production (Manual via PR)

**Using GitHub CLI:**
```bash
# In simulator-spi repository
# Promote specific commit
gh workflow run promote-prd.yml -f COMMIT_SHA=abc123def456
```

**After PR is merged:**
```bash
# Manually trigger ArgoCD sync
argocd app sync simulator-spi-prd
```

## 🔧 Local Testing

### Validate Kustomization

```bash
# Dev
kustomize build environments/dev

# QA
kustomize build environments/qa

# Staging
kustomize build environments/staging

# Production
kustomize build environments/prd
```

### Apply Manually (for testing)

```bash
# Dev
kustomize build environments/dev | kubectl apply -f -

# Check deployment
kubectl get pods -n simulator-spi
kubectl get deployments -n simulator-spi
```

## 📊 ArgoCD Setup

### Install Applications

```bash
# Apply Applications (in respective ArgoCD instances)
# Dev ArgoCD
kubectl apply -f argocd-apps/dev.yaml -n gitops

# QA ArgoCD
kubectl apply -f argocd-apps/qa.yaml -n argocd

# Staging ArgoCD
kubectl apply -f argocd-apps/staging.yaml -n argocd

# Production ArgoCD
kubectl apply -f argocd-apps/prd.yaml -n argocd
```

### View in ArgoCD UI

```bash
# Dev
argocd app get simulator-spi-dev

# QA
argocd app get simulator-spi-qa

# Staging
argocd app get simulator-spi-staging

# Production
argocd app get simulator-spi-prd
```

### Manual Sync

```bash
# Staging (manual sync required)
argocd app sync simulator-spi-staging

# Production (manual sync required)
argocd app sync simulator-spi-prd
```

## 🐛 Troubleshooting

### Application Out of Sync

```bash
# Check diff
argocd app diff simulator-spi-dev

# Force sync
argocd app sync simulator-spi-dev --force

# Refresh
argocd app get simulator-spi-dev --refresh
```

### Pod Failing to Start

```bash
# Check pod logs
kubectl logs -n simulator-spi simulator-spi-xxx

# Check events
kubectl describe pod -n simulator-spi simulator-spi-xxx

# Check configmap
kubectl get configmap -n simulator-spi
kubectl get configmap shared-config -n simulator-spi -o yaml
```

### Image Pull Errors

```bash
# Check regcred secret
kubectl get secret regcred -n simulator-spi -o yaml

# Recreate secret if needed (use existing regcred in cluster)
```

### Rollback

```bash
# Option 1: Git revert
git revert HEAD
git push

# Option 2: Kubectl rollback
kubectl rollout undo deployment/simulator-spi -n simulator-spi

# Option 3: Update image tag in kustomization.yaml to previous version
```

## 📚 Additional Resources

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Kustomize Documentation](https://kustomize.io/)
- [Main Repository](https://github.com/lb-conn/simulator-spi)

## 🤝 Contributing

1. Create feature branch
2. Update manifests
3. Test locally with `kustomize build`
4. Create Pull Request
5. Wait for approvals (staging/prd only)
6. Merge to main

## 📝 Notes

- **Single Application** - One deployment per environment
- **Shared resources** - ConfigMap is shared across all pods
- **Environment progression** - Dev → QA → Staging → Production flow
- **Health checks** - HTTP probes on /health endpoint (if available)
- **Port** - Application uses port 3000 (not 8080 like simulator-dict)
