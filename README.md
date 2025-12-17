# Portal Backoffice GitOps Configuration

GitOps repository for managing Portal Backoffice deployments across multiple environments using ArgoCD and Kustomize.

## 📁 Repository Structure

```
portal_backoffice-config/
├── argocd-apps/              # ArgoCD Application definitions
│   ├── dev.yaml              # Dev environment Application
│   ├── qa.yaml               # QA environment Application
│   └── staging.yaml          # Staging environment Application
├── base/                     # Base Kubernetes manifests
│   ├── kustomization.yaml
│   ├── shared-configmap.yaml
│   └── (deployment and service will be added when source code is available)
└── environments/             # Environment-specific overlays
    ├── dev/
    │   └── kustomization.yaml
    ├── qa/
    │   └── kustomization.yaml
    └── staging/
        └── kustomization.yaml
```

## 🏗️ Architecture

### Application

- **portal-backoffice**: Backoffice portal application
- Deploy to namespace: `portal-backoffice`
- TODO: Ports and image registry will be defined when source code is available
- TODO: Pull images from GHCR: `ghcr.io/london-bridge/portal_backoffice/`

### Environments

| Environment | Replicas | Resources | Sync Policy | Project |
|-------------|----------|-----------|-------------|---------|
| **Dev** | TBD | TBD | Automated | default |
| **QA** | TBD | TBD | Automated | default |
| **Staging** | TBD | TBD | Manual | default |

## 🚀 Deployment Flow

```
main branch → DEV (auto)
    ↓
   QA tag → QA (auto after approval)
    ↓
  STG tag → STG (PR → approval → merge → manual sync)
```

### 1. Development Environment

**Trigger**: Push to `main` branch in portal_backoffice repo

**Process**:
1. CI/CD builds Docker images with tag `main-{sha}`
2. GitHub Actions updates `environments/dev/kustomization.yaml`
3. Commits directly to main branch
4. ArgoCD auto-syncs (prune + selfHeal enabled)

**Deployment Name**: `portal-backoffice`

### 2. QA Environment

**Trigger**: Manual promotion workflow or `qa-*` tag

**Process**:
1. Validate images exist in GHCR
2. Update `environments/qa/kustomization.yaml` with `qa-{sha}` tag
3. Direct commit to main
4. ArgoCD auto-syncs

**Deployment Name**: `portal-backoffice`

### 3. Staging Environment

**Trigger**: Manual promotion workflow or `stg-*` tag

**Process**:
1. Create Pull Request updating `environments/staging/kustomization.yaml`
2. Require approvals (QA Team + Tech Lead)
3. After merge, manually trigger ArgoCD sync
4. Multiple replicas for HA
5. Production-mirror configuration

**Deployment Name**: `portal-backoffice`

## 📦 How to Update

### Update Dev (Automatic)

Automated via CI/CD workflow after successful build on `main` branch.

### Update QA (Manual)

**Using GitHub CLI:**
```bash
# In portal_backoffice repository
# Promote specific commit
gh workflow run promote-qa.yml -f COMMIT_SHA=abc123def456

# Or promote latest main
gh workflow run promote-qa.yml
```

**What happens:**
- ✅ Validates image exists in GHCR
- ✅ Updates `environments/qa/kustomization.yaml` directly
- ✅ Commits to main branch
- ✅ ArgoCD auto-syncs (~ 3 minutes)

### Update Staging (Manual via PR)

**Using GitHub CLI:**
```bash
# In portal_backoffice repository
# Promote specific commit
gh workflow run promote-staging.yml -f COMMIT_SHA=abc123def456
```

**What happens:**
- ✅ Validates image exists in GHCR
- ✅ Creates branch `promote-staging-{sha}`
- ✅ Updates `environments/staging/kustomization.yaml`
- ✅ Creates Pull Request with detailed info
- ⏳ **Requires approval** before merge
- ⚠️ **Manual ArgoCD sync required** after merge

**After PR is merged:**
```bash
# Manually trigger ArgoCD sync
argocd app sync portal-backoffice-staging
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
```

### Apply Manually (for testing)

```bash
# Dev
kustomize build environments/dev | kubectl apply -f -

# Check deployment
kubectl get pods -n portal-backoffice
kubectl get deployments -n portal-backoffice
```

## 📊 ArgoCD Setup

### Install Applications

```bash
# Apply Applications (in respective ArgoCD instances)
# Dev ArgoCD
kubectl apply -f argocd-apps/dev.yaml -n argocd

# QA ArgoCD
kubectl apply -f argocd-apps/qa.yaml -n argocd

# Staging ArgoCD
kubectl apply -f argocd-apps/staging.yaml -n argocd
```

### View in ArgoCD UI

```bash
# Dev
argocd app get portal-backoffice-dev

# QA
argocd app get portal-backoffice-qa

# Staging
argocd app get portal-backoffice-staging
```

### Manual Sync

```bash
# Staging (manual sync required)
argocd app sync portal-backoffice-staging
```

## 🐛 Troubleshooting

### Application Out of Sync

```bash
# Check diff
argocd app diff portal-backoffice-dev

# Force sync
argocd app sync portal-backoffice-dev --force

# Refresh
argocd app get portal-backoffice-dev --refresh
```

### Pod Failing to Start

```bash
# Check pod logs
kubectl logs -n portal-backoffice portal-backoffice-xxx

# Check events
kubectl describe pod -n portal-backoffice portal-backoffice-xxx

# Check configmap
kubectl get configmap -n portal-backoffice
kubectl get configmap shared-config -n portal-backoffice -o yaml
```

### Image Pull Errors

```bash
# Check regcred secret
kubectl get secret regcred -n portal-backoffice -o yaml

# Recreate secret if needed (use existing regcred in cluster)
```

### Rollback

```bash
# Option 1: Git revert
git revert HEAD
git push

# Option 2: Kubectl rollback
kubectl rollout undo deployment/portal-backoffice -n portal-backoffice

# Option 3: Update image tag in kustomization.yaml to previous version
```

## 📝 TODO

- [ ] Adicionar deployment.yaml quando código fonte estiver disponível
- [ ] Adicionar service.yaml quando código fonte estiver disponível
- [ ] Configurar variáveis de ambiente específicas
- [ ] Configurar recursos (CPU/Memory) por ambiente
- [ ] Configurar health checks
- [ ] Configurar ingress se necessário
- [ ] Configurar secrets e external secrets
- [ ] Configurar service account se necessário

## 📚 Additional Resources

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Kustomize Documentation](https://kustomize.io/)
- [Main Repository](https://github.com/london-bridge/portal_backoffice)

## 🤝 Contributing

1. Create feature branch
2. Update manifests
3. Test locally with `kustomize build`
4. Create Pull Request
5. Wait for approvals (staging only)
6. Merge to main
