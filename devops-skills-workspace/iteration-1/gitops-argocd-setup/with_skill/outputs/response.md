# ArgoCD Multi-Cluster Setup: Dev / Staging / Prod

## Overview

This guide walks you through setting up ArgoCD 3.x to manage three Kubernetes clusters (dev, staging, prod) with differentiated sync policies: **automatic sync for dev** and **manual approval for prod**. It follows the GitOps workflow from the `gitops-workflows` skill, specifically Workflow 1 (ArgoCD Initial Setup) and Workflow 4 (Multi-Cluster Deployments).

---

## Architecture Decision

Based on the skill's decision tree:

- You don't have GitOps installed yet -> Choose a tool
- You want UI + easy onboarding -> **ArgoCD** (correct choice for your use case)
- You have 3 clusters -> Use **ApplicationSets** with per-environment sync policies
- Recommended pattern: **Hub-and-spoke** -- install ArgoCD on one management cluster (or your prod cluster) and have it manage all three clusters

```
                    +------------------+
                    |   ArgoCD Server  |
                    |  (mgmt cluster)  |
                    +--------+---------+
                             |
              +--------------+--------------+
              |              |              |
        +-----+----+  +-----+----+  +------+-----+
        | Dev       |  | Staging  |  | Production |
        | Cluster   |  | Cluster  |  | Cluster    |
        | auto-sync |  | auto-sync|  | manual     |
        +-----------+  +----------+  +------------+
```

---

## Step 1: Install ArgoCD 3.x

Install ArgoCD on your management cluster (this can be a dedicated cluster or your prod cluster).

```bash
# Switch to your management cluster context
kubectl config use-context mgmt-cluster

# Create namespace
kubectl create namespace argocd

# Install ArgoCD 3.x (latest stable: v3.1.9)
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v3.1.9/manifests/install.yaml

# Get the initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Port forward to access UI (for initial setup)
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Access: https://localhost:8080
```

### ArgoCD 3.x Configuration Best Practices

Apply the following ConfigMaps for optimal ArgoCD 3.x behavior. These are drawn from the skill's installation template (`assets/argocd/install-argocd-3.x.yaml`):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cmd-params-cm
  namespace: argocd
data:
  server.enable.gzip: "true"
  resource.exclusions: |
    - apiGroups:
      - ""
      kinds:
      - Endpoints
      - EndpointSlice
      clusters:
      - "*"
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  # Annotation-based tracking is the default in ArgoCD 3.x (more reliable than labels)
  application.resourceTrackingMethod: annotation
  resource.exclusions: |
    - apiGroups:
      - "*"
      kinds:
      - Lease
      clusters:
      - "*"
```

### Expose ArgoCD (Ingress -- Recommended for Production)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  ingressClassName: nginx
  rules:
  - host: argocd.yourcompany.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              number: 443
  tls:
  - hosts:
    - argocd.yourcompany.com
    secretName: argocd-server-tls
```

### Install the ArgoCD CLI

```bash
# macOS
brew install argocd

# Login
argocd login argocd.yourcompany.com --username admin --password <password>
```

---

## Step 2: Register Your 3 Clusters

Register each target cluster with ArgoCD so it can deploy to them:

```bash
# Register dev cluster
argocd cluster add dev-cluster-context \
  --name dev \
  --label environment=dev

# Register staging cluster
argocd cluster add staging-cluster-context \
  --name staging \
  --label environment=staging

# Register production cluster
argocd cluster add prod-cluster-context \
  --name prod \
  --label environment=production
```

The `--label` flags are critical -- they enable ApplicationSets to target clusters by environment.

**2025 Best Practice**: Use **workload identity** (AWS IRSA, GCP Workload Identity, Azure AD Workload Identity) instead of long-lived service account tokens when connecting clusters. This eliminates static credential management.

### Verify Cluster Registration

```bash
argocd cluster list
```

Expected output:
```
SERVER                          NAME     VERSION  STATUS
https://dev.k8s.local           dev      1.29     Successful
https://staging.k8s.local       staging  1.29     Successful
https://prod.k8s.local          prod     1.29     Successful
https://kubernetes.default.svc  in-cluster        Successful
```

---

## Step 3: Set Up Your Git Repository Structure

Following the skill's recommended monorepo pattern with Kustomize overlays (best for <20 apps):

```
gitops-repo/
├── apps/
│   ├── frontend/
│   │   ├── base/
│   │   │   ├── deployment.yaml
│   │   │   ├── service.yaml
│   │   │   └── kustomization.yaml
│   │   └── overlays/
│   │       ├── dev/
│   │       │   ├── kustomization.yaml
│   │       │   └── replica-patch.yaml
│   │       ├── staging/
│   │       │   ├── kustomization.yaml
│   │       │   └── replica-patch.yaml
│   │       └── production/
│   │           ├── kustomization.yaml
│   │           ├── replica-patch.yaml
│   │           └── resource-limits-patch.yaml
│   ├── backend/
│   │   ├── base/
│   │   └── overlays/
│   │       ├── dev/
│   │       ├── staging/
│   │       └── production/
│   └── database/
│       ├── base/
│       └── overlays/
│           ├── dev/
│           ├── staging/
│           └── production/
├── infrastructure/
│   ├── ingress/
│   ├── monitoring/
│   └── secrets/
├── clusters/
│   ├── dev/
│   │   └── kustomization.yaml       # References apps/*/overlays/dev
│   ├── staging/
│   │   └── kustomization.yaml       # References apps/*/overlays/staging
│   └── production/
│       └── kustomization.yaml       # References apps/*/overlays/production
├── applicationsets/
│   ├── dev-apps.yaml                # Auto-sync ApplicationSet
│   ├── staging-apps.yaml            # Auto-sync ApplicationSet
│   └── prod-apps.yaml               # Manual-sync ApplicationSet
└── .sops.yaml                        # SOPS encryption config
```

### Example: Base Deployment

```yaml
# apps/frontend/base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: myregistry/frontend:v1.0.0   # Always pin versions, never use :latest
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 256Mi
```

### Example: Dev Overlay (Fewer Replicas, Lower Resources)

```yaml
# apps/frontend/overlays/dev/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
patches:
  - path: replica-patch.yaml
```

```yaml
# apps/frontend/overlays/dev/replica-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 1
```

### Example: Production Overlay (Higher Replicas, Strict Limits)

```yaml
# apps/frontend/overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
patches:
  - path: replica-patch.yaml
  - path: resource-limits-patch.yaml
```

```yaml
# apps/frontend/overlays/production/replica-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 3
```

### Validate Your Repo Structure

Use the skill's validation script to check your repository follows best practices:

```bash
python3 gitops-workflows/scripts/validate_gitops_repo.py /path/to/gitops-repo
```

This will verify YAML validity, detect deprecated Kustomize syntax, and audit secrets management.

---

## Step 4: Configure ArgoCD Projects with RBAC

Create ArgoCD Projects to enforce boundaries between environments. This leverages ArgoCD 3.x's fine-grained RBAC:

```yaml
# argocd-projects.yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: dev
  namespace: argocd
spec:
  description: Development environment
  sourceRepos:
    - 'https://github.com/yourorg/gitops-repo.git'
  destinations:
    - namespace: '*'
      server: https://dev.k8s.local
  clusterResourceWhitelist:
    - group: '*'
      kind: '*'
---
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: staging
  namespace: argocd
spec:
  description: Staging environment
  sourceRepos:
    - 'https://github.com/yourorg/gitops-repo.git'
  destinations:
    - namespace: '*'
      server: https://staging.k8s.local
  clusterResourceWhitelist:
    - group: '*'
      kind: '*'
---
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: production
  namespace: argocd
spec:
  description: Production environment - manual sync only
  sourceRepos:
    - 'https://github.com/yourorg/gitops-repo.git'
  destinations:
    - namespace: '*'
      server: https://prod.k8s.local
  clusterResourceWhitelist:
    - group: '*'
      kind: '*'
```

### Fine-Grained RBAC (ArgoCD 3.x)

```csv
# argocd-rbac-cm ConfigMap (data.policy.csv)

# Dev team can sync dev freely
p, role:dev-team, applications, *, dev/*, allow
p, role:dev-team, applications, get, staging/*, allow
p, role:dev-team, applications, get, production/*, allow

# Staging: dev team can view, SRE can sync
p, role:sre-team, applications, *, staging/*, allow

# Production: only SRE/leads can sync, and only after approval
p, role:sre-team, applications, sync, production/*, allow
p, role:sre-team, applications, get, production/*, allow
p, role:dev-team, applications, get, production/*, deny-sync

# Per-resource permissions (new in ArgoCD 3.0)
p, role:dev-team, applications/resources, *, dev/*/Deployment/*, allow
```

---

## Step 5: Create ApplicationSets with Differentiated Sync Policies

This is the core of your requirement: **auto-sync for dev, manual approval for prod**.

### Dev ApplicationSet (Automatic Sync)

```yaml
# applicationsets/dev-apps.yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: dev-apps
  namespace: argocd
spec:
  goTemplate: true
  generators:
  - cluster:
      selector:
        matchLabels:
          environment: dev
  template:
    metadata:
      name: 'dev-{{.name}}'
      labels:
        environment: dev
    spec:
      project: dev
      source:
        repoURL: https://github.com/yourorg/gitops-repo.git
        targetRevision: main
        path: 'apps/{{.name}}/overlays/dev'
      destination:
        server: '{{.server}}'
        namespace: '{{.name}}'
      syncPolicy:
        automated:               # <-- AUTOMATIC SYNC
          prune: true            # Remove resources deleted from Git
          selfHeal: true         # Revert manual cluster changes
        syncOptions:
        - CreateNamespace=true
        retry:
          limit: 5
          backoff:
            duration: 5s
            factor: 2
            maxDuration: 3m
```

### Staging ApplicationSet (Automatic Sync with Self-Heal)

```yaml
# applicationsets/staging-apps.yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: staging-apps
  namespace: argocd
spec:
  goTemplate: true
  generators:
  - cluster:
      selector:
        matchLabels:
          environment: staging
  template:
    metadata:
      name: 'staging-{{.name}}'
      labels:
        environment: staging
    spec:
      project: staging
      source:
        repoURL: https://github.com/yourorg/gitops-repo.git
        targetRevision: main
        path: 'apps/{{.name}}/overlays/staging'
      destination:
        server: '{{.server}}'
        namespace: '{{.name}}'
      syncPolicy:
        automated:               # <-- AUTOMATIC SYNC
          prune: true
          selfHeal: true
        syncOptions:
        - CreateNamespace=true
        retry:
          limit: 3
          backoff:
            duration: 10s
            factor: 2
            maxDuration: 5m
```

### Production ApplicationSet (Manual Sync -- No Auto)

```yaml
# applicationsets/prod-apps.yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: prod-apps
  namespace: argocd
spec:
  goTemplate: true
  generators:
  - cluster:
      selector:
        matchLabels:
          environment: production
  template:
    metadata:
      name: 'prod-{{.name}}'
      labels:
        environment: production
    spec:
      project: production
      source:
        repoURL: https://github.com/yourorg/gitops-repo.git
        targetRevision: main
        path: 'apps/{{.name}}/overlays/production'
      destination:
        server: '{{.server}}'
        namespace: '{{.name}}'
      syncPolicy:
        # NO automated block = MANUAL SYNC ONLY
        syncOptions:
        - CreateNamespace=true
        - PruneLast=true         # Prune after sync (safer for prod)
        - ApplyOutOfSyncOnly=true  # Only apply changed resources
        retry:
          limit: 2
          backoff:
            duration: 30s
            factor: 2
            maxDuration: 5m
```

**Key difference**: The production ApplicationSet has **no `automated` block** under `syncPolicy`. This means:
- Changes merged to `main` will show as "OutOfSync" in the ArgoCD UI
- An operator must manually click "Sync" in the UI or run `argocd app sync prod-<appname>`
- This is your manual approval gate

### Generate ApplicationSets with the Skill's Script

You can also use the provided script to generate these manifests:

```bash
# Generate cluster-based ApplicationSet for dev
python3 gitops-workflows/scripts/applicationset_generator.py cluster \
  --name dev-apps \
  --repo-url https://github.com/yourorg/gitops-repo.git \
  --path apps/ \
  --cluster-label environment=dev \
  --output applicationsets/dev-apps.yaml

# Generate for production
python3 gitops-workflows/scripts/applicationset_generator.py cluster \
  --name prod-apps \
  --repo-url https://github.com/yourorg/gitops-repo.git \
  --path apps/ \
  --cluster-label environment=production \
  --output applicationsets/prod-apps.yaml
```

Note: After generation, you will need to edit the prod ApplicationSet to remove the `automated` sync policy block.

---

## Step 6: Environment Promotion Workflow

Following the skill's best-practices recommendation:

```
Dev (auto-sync) --> Staging (auto-sync) --> Production (manual approval via PR)
```

### How Promotion Works

1. **Developer** pushes code, CI builds a new image (e.g., `myregistry/frontend:v1.2.0`)
2. **CI pipeline** updates the image tag in `apps/frontend/base/deployment.yaml` (or uses Kustomize image transformer)
3. **PR merged to main** -- ArgoCD auto-syncs dev and staging within minutes
4. **Production** shows "OutOfSync" in ArgoCD UI
5. **SRE/Lead reviews** the diff in ArgoCD UI (`argocd app diff prod-frontend`)
6. **Manual sync** is triggered after approval (`argocd app sync prod-frontend`)

### Validate Promotions

Use the skill's promotion validator:

```bash
python3 gitops-workflows/scripts/promotion_validator.py \
  --source dev \
  --target staging
```

### Optional: Separate Production Promotion via PR

For even stricter control, use a branch-based approach for production:

```
main branch        --> dev and staging (auto-sync)
release/* branch   --> production (manual sync)
```

In this model, production only picks up changes when someone creates a release branch or PR targeting the production path.

---

## Step 7: Secrets Management

Following the skill's 2025 recommendation, use **SOPS + age** for encrypting secrets in Git:

### Setup Per-Environment Encryption Keys

```bash
# Generate one age key per environment
age-keygen -o dev-key.txt
age-keygen -o staging-key.txt
age-keygen -o prod-key.txt
```

### Configure .sops.yaml

```yaml
# .sops.yaml (in repo root)
creation_rules:
  - path_regex: production/.*\.yaml$
    encrypted_regex: ^(data|stringData)$
    age: age1<your-prod-public-key>

  - path_regex: staging/.*\.yaml$
    encrypted_regex: ^(data|stringData)$
    age: age1<your-staging-public-key>

  - path_regex: dev/.*\.yaml$
    encrypted_regex: ^(data|stringData)$
    age: age1<your-dev-public-key>
```

### Deploy Age Keys to ArgoCD

ArgoCD needs the age private keys to decrypt secrets during sync:

```bash
# Create Kubernetes secrets with age keys in the argocd namespace
kubectl create secret generic helm-secrets-private-keys \
  --from-file=dev-key.txt \
  --from-file=staging-key.txt \
  --from-file=prod-key.txt \
  -n argocd
```

### Git Pre-Commit Hook (Prevent Plain Secrets)

```bash
#!/bin/bash
# .git/hooks/pre-commit
if git diff --cached --name-only | grep -E 'secret.*\.yaml$'; then
  echo "WARNING: Potential secret file detected."
  echo "Ensure it is encrypted with SOPS before committing."
  exit 1
fi
```

### Audit Secrets

```bash
python3 gitops-workflows/scripts/secret_audit.py /path/to/gitops-repo
```

This will detect plain secrets in Git (HIGH severity) and verify SOPS/age/ESO configuration.

---

## Step 8: Monitoring and Health Checks

### Check ArgoCD Health

```bash
python3 gitops-workflows/scripts/check_argocd_health.py \
  --server https://argocd.yourcompany.com \
  --token $ARGOCD_TOKEN
```

### Detect Configuration Drift

```bash
python3 gitops-workflows/scripts/sync_drift_detector.py \
  --argocd \
  --app prod-frontend
```

### Essential ArgoCD Commands

```bash
# List all applications
argocd app list

# Check a specific app
argocd app get prod-frontend

# View what would change (diff before syncing prod)
argocd app diff prod-frontend

# Manually sync production (after review)
argocd app sync prod-frontend

# Check sync history
argocd app history prod-frontend
```

### Set Up Prometheus Alerts for Sync Failures

```yaml
# prometheus-argocd-alerts.yaml
groups:
- name: argocd
  rules:
  - alert: ArgoAppOutOfSync
    expr: argocd_app_info{sync_status="OutOfSync", project="production"} == 1
    for: 30m
    labels:
      severity: warning
    annotations:
      summary: "Production app {{ $labels.name }} is out of sync for 30+ minutes"

  - alert: ArgoAppDegraded
    expr: argocd_app_info{health_status="Degraded"} == 1
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "ArgoCD app {{ $labels.name }} is degraded"

  - alert: ArgoSyncFailed
    expr: argocd_app_sync_total{phase="Error"} > 0
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "ArgoCD sync failed for {{ $labels.name }}"
```

---

## Step 9: Apply Everything

```bash
# 1. Apply ArgoCD projects
kubectl apply -f argocd-projects.yaml

# 2. Apply ApplicationSets
kubectl apply -f applicationsets/dev-apps.yaml
kubectl apply -f applicationsets/staging-apps.yaml
kubectl apply -f applicationsets/prod-apps.yaml

# 3. Verify applications were created
argocd app list

# 4. Verify dev auto-syncs
argocd app get dev-frontend
# STATUS should show "Synced" and "Healthy"

# 5. Verify prod is manual (shows OutOfSync until you sync)
argocd app get prod-frontend
# STATUS should show "OutOfSync" until manually synced
```

---

## Summary: Sync Policy Matrix

| Environment | Sync Mode   | Self-Heal | Prune | Approval      |
|-------------|-------------|-----------|-------|---------------|
| **Dev**     | Automated   | Yes       | Yes   | None (auto)   |
| **Staging** | Automated   | Yes       | Yes   | None (auto)   |
| **Prod**    | Manual      | No        | Yes*  | Manual via UI/CLI |

*Production uses `PruneLast=true` so deletions happen after all other sync operations, reducing risk.

---

## Key ArgoCD 3.x Changes to Be Aware Of

- **Annotation-based tracking** is now the default (was label-based). More reliable, auto-migrates on sync.
- **Fine-grained RBAC** allows per-resource permissions (e.g., allow devs to sync Deployments but not Secrets in prod).
- **Resource exclusions** are now on by default, excluding high-churn resources like Endpoints and Leases for better performance.
- **Secrets operators endorsed**: ArgoCD 3.0 officially recommends External Secrets Operator or SOPS over config management plugins.

---

## Troubleshooting Quick Reference

| Issue | Command |
|-------|---------|
| App won't sync | `argocd app get <app> --show-operation` |
| View diff before sync | `argocd app diff <app>` |
| Force sync after 3.x upgrade | `argocd app sync <app> --force` |
| Check controller logs | `kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller` |
| Detect drift | `python3 scripts/sync_drift_detector.py --argocd --app <app>` |
| Audit secrets | `python3 scripts/secret_audit.py /path/to/repo` |
| Validate repo structure | `python3 scripts/validate_gitops_repo.py /path/to/repo` |

---

## Next Steps

1. **Progressive delivery** -- Once stable, consider adding Argo Rollouts for canary deployments in production (see `references/progressive_delivery.md`)
2. **Image automation** -- Set up CI to automatically update image tags in Git after successful builds
3. **Notifications** -- Configure ArgoCD notifications to Slack/Teams for sync events
4. **Disaster recovery** -- Document your bootstrap procedure; Git is your source of truth, so any cluster can be rebuilt from the repo
