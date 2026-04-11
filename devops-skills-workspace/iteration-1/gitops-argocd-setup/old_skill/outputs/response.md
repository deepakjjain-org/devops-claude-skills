# ArgoCD Multi-Cluster Setup: Dev / Staging / Prod

## Workflow Followed

Based on the GitOps Workflows skill decision tree: you do not have GitOps installed, you want UI + easy onboarding, so **ArgoCD is the right choice** (Workflow 1: Initial Setup, then Workflow 4: Multi-Cluster Deployments). Your requirement for automatic sync on dev but manual approval on prod maps directly to the recommended environment promotion pattern: `Dev (auto-sync) -> Staging (auto-sync) -> Production (manual approval)`.

---

## Phase 1: Install ArgoCD 3.x on a Management Cluster

ArgoCD should run on a single management cluster (hub-and-spoke pattern) that manages all three target clusters. This gives you centralized visibility and avoids running three separate ArgoCD instances.

```bash
# Create the argocd namespace
kubectl create namespace argocd

# Install ArgoCD 3.x (latest stable: v3.1.9)
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v3.1.9/manifests/install.yaml

# Retrieve the initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Port-forward to access the UI (or set up an Ingress -- see below)
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

### ArgoCD 3.x Configuration (Best Practices)

Apply these ConfigMaps for production readiness. ArgoCD 3.x defaults to annotation-based tracking and includes resource exclusions for performance:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  # Annotation-based tracking (ArgoCD 3.x default -- more reliable than labels)
  application.resourceTrackingMethod: annotation

  # Resource exclusions for performance (reduce API load)
  resource.exclusions: |
    - apiGroups:
      - "*"
      kinds:
      - Lease
      clusters:
      - "*"
---
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
  - host: argocd.example.com
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
    - argocd.example.com
    secretName: argocd-server-tls
```

Reference: `assets/argocd/install-argocd-3.x.yaml`

---

## Phase 2: Register Your Three Clusters

Add each cluster to ArgoCD with labels that identify their environment. These labels are critical for ApplicationSets later.

```bash
# Login to ArgoCD CLI
argocd login argocd.example.com --username admin --password <password>

# Register clusters (assumes kubeconfig contexts are set up)
argocd cluster add dev-cluster-context \
  --name dev-cluster \
  --label environment=dev

argocd cluster add staging-cluster-context \
  --name staging-cluster \
  --label environment=staging

argocd cluster add prod-cluster-context \
  --name prod-cluster \
  --label environment=production

# Verify clusters are registered
argocd cluster list
```

**Important (2025 Best Practice)**: Use workload identity (AWS IRSA, GCP Workload Identity, Azure AD Workload Identity) instead of long-lived service account tokens for cluster authentication.

---

## Phase 3: Design Your Git Repository Structure

Use the monorepo pattern with Kustomize overlays for environment differences:

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
│   │       │   └── replica-patch.yaml       # 1 replica
│   │       ├── staging/
│   │       │   ├── kustomization.yaml
│   │       │   └── replica-patch.yaml       # 2 replicas
│   │       └── production/
│   │           ├── kustomization.yaml
│   │           └── replica-patch.yaml       # 3+ replicas, resource limits
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
└── clusters/
    ├── dev/
    ├── staging/
    └── production/
```

**Key principles**:
- Use Kustomize bases + overlays so you do not repeat YAML across environments.
- Separate infrastructure from applications.
- Pin image tags to specific versions (never use `:latest`).

You can validate this structure with:

```bash
python3 scripts/validate_gitops_repo.py /path/to/gitops-repo
```

Reference: `references/repo_patterns.md`

---

## Phase 4: Create ApplicationSets (The Core of Multi-Cluster)

This is where you implement the different sync policies per environment. You need **three separate ApplicationSets** -- one per environment -- so you can control the sync policy independently.

### Dev ApplicationSet (Auto-Sync Enabled)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: dev-apps
  namespace: argocd
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
  - cluster:
      selector:
        matchLabels:
          environment: dev
  template:
    metadata:
      name: 'dev-{{.name}}-apps'
      labels:
        environment: dev
    spec:
      project: dev-project
      source:
        repoURL: https://github.com/your-org/gitops-repo
        targetRevision: main
        path: 'apps/{{.name}}/overlays/dev'
      destination:
        server: '{{.server}}'
        namespace: '{{.name}}'
      syncPolicy:
        automated:          # <-- AUTO-SYNC: changes deploy automatically
          prune: true       # Remove resources deleted from Git
          selfHeal: true    # Revert manual cluster changes
        syncOptions:
        - CreateNamespace=true
        retry:
          limit: 3
          backoff:
            duration: 5s
            maxDuration: 3m0s
            factor: 2
```

### Staging ApplicationSet (Auto-Sync Enabled)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: staging-apps
  namespace: argocd
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
  - cluster:
      selector:
        matchLabels:
          environment: staging
  template:
    metadata:
      name: 'staging-{{.name}}-apps'
      labels:
        environment: staging
    spec:
      project: staging-project
      source:
        repoURL: https://github.com/your-org/gitops-repo
        targetRevision: main
        path: 'apps/{{.name}}/overlays/staging'
      destination:
        server: '{{.server}}'
        namespace: '{{.name}}'
      syncPolicy:
        automated:          # <-- AUTO-SYNC for staging too
          prune: true
          selfHeal: true
        syncOptions:
        - CreateNamespace=true
```

### Production ApplicationSet (Manual Sync -- No automated block)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: prod-apps
  namespace: argocd
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
  - cluster:
      selector:
        matchLabels:
          environment: production
  template:
    metadata:
      name: 'prod-{{.name}}-apps'
      labels:
        environment: production
    spec:
      project: prod-project
      source:
        repoURL: https://github.com/your-org/gitops-repo
        targetRevision: main
        path: 'apps/{{.name}}/overlays/production'
      destination:
        server: '{{.server}}'
        namespace: '{{.name}}'
      # NOTE: No syncPolicy.automated block!
      # This means production requires MANUAL sync (approval).
      syncPolicy:
        syncOptions:
        - CreateNamespace=true
```

**The key difference**: The `syncPolicy.automated` block is present on dev and staging but **absent on production**. When ArgoCD detects a Git change for prod, the application will show as "OutOfSync" in the UI, but it will not deploy until someone explicitly clicks Sync or runs `argocd app sync`.

You can also generate these ApplicationSets using the provided script:

```bash
# Generate a cluster-based ApplicationSet
python3 scripts/applicationset_generator.py cluster \
  --name dev-apps \
  --repo-url https://github.com/your-org/gitops-repo \
  --path apps/ \
  --cluster-label "environment: dev" \
  --output dev-appset.yaml
```

Reference: `assets/applicationsets/cluster-generator.yaml`, `references/multi_cluster.md`

---

## Phase 5: Set Up RBAC (Fine-Grained, ArgoCD 3.x)

Use ArgoCD 3.x fine-grained RBAC to control who can sync to production:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.csv: |
    # Dev team: full access to dev, read-only on staging/prod
    p, role:developer, applications, *, dev-project/*, allow
    p, role:developer, applications, get, staging-project/*, allow
    p, role:developer, applications, get, prod-project/*, allow

    # QA/Staging team: can sync staging, read-only on prod
    p, role:qa, applications, *, staging-project/*, allow
    p, role:qa, applications, get, prod-project/*, allow

    # SRE/Platform team: can sync production (manual approval)
    p, role:sre, applications, *, prod-project/*, allow
    p, role:sre, applications, *, staging-project/*, allow
    p, role:sre, applications, *, dev-project/*, allow

    # Map groups to roles
    g, dev-team, role:developer
    g, qa-team, role:qa
    g, sre-team, role:sre

  policy.default: role:readonly
```

This ensures only SRE/platform engineers can trigger the manual production sync.

---

## Phase 6: Implement Environment Promotion Workflow

The recommended promotion flow:

```
Dev (auto-sync) --> Staging (auto-sync) --> Production (manual approval via PR)
```

### How It Works

1. **Developer** commits code, CI builds image tagged `v1.2.3`.
2. **CI pipeline** updates the image tag in `apps/<app>/overlays/dev/kustomization.yaml`.
3. **Dev auto-syncs** -- ArgoCD detects the change and deploys to dev automatically.
4. **After dev validation**, developer creates a PR updating `apps/<app>/overlays/staging/kustomization.yaml` with the same image tag.
5. **Staging auto-syncs** after PR merge.
6. **After staging validation**, developer (or automated tests) creates a PR updating `apps/<app>/overlays/production/kustomization.yaml`.
7. **PR is reviewed and approved** by the SRE team.
8. **After merge, production shows OutOfSync**. An SRE manually clicks Sync in the ArgoCD UI or runs:

```bash
argocd app sync prod-prod-cluster-apps
```

You can validate promotion workflows with:

```bash
python3 scripts/promotion_validator.py --source dev --target staging --repo-path /path/to/gitops-repo
python3 scripts/promotion_validator.py --source staging --target production --repo-path /path/to/gitops-repo
```

---

## Phase 7: Secrets Management

Never commit plain secrets to Git. The recommended approach for 2025:

### Option A: SOPS + age (Preferred for Git-centric workflow)

```bash
# Generate an age key
age-keygen -o key.txt
# Public key: age1...

# Create .sops.yaml with per-environment keys
cat <<EOF > .sops.yaml
creation_rules:
  - path_regex: production/.*
    encrypted_regex: ^(data|stringData)$
    age: age1prod_key_here...
  - path_regex: staging/.*
    encrypted_regex: ^(data|stringData)$
    age: age1staging_key_here...
  - path_regex: dev/.*
    encrypted_regex: ^(data|stringData)$
    age: age1dev_key_here...
EOF

# Encrypt a secret
sops -e secret.yaml > secret.enc.yaml
git add secret.enc.yaml .sops.yaml
```

### Option B: External Secrets Operator (Preferred for cloud-native)

If you already use AWS Secrets Manager, Azure Key Vault, or GCP Secret Manager, ESO can pull secrets dynamically and rotate them automatically.

Audit your secrets posture with:

```bash
python3 scripts/secret_audit.py /path/to/gitops-repo
```

Reference: `references/secret_management.md`, `assets/secrets/sops-age-config.yaml`

---

## Phase 8: Health Monitoring and Drift Detection

After setup, continuously monitor your deployment health.

### Health Checks

```bash
# Check all ArgoCD application health
python3 scripts/check_argocd_health.py \
  --server https://argocd.example.com \
  --token $ARGOCD_TOKEN \
  --show-healthy

# Check a specific application
python3 scripts/check_argocd_health.py \
  --server https://argocd.example.com \
  --token $ARGOCD_TOKEN \
  --app prod-prod-cluster-frontend
```

### Drift Detection

Detect when someone makes manual changes to the cluster that diverge from Git:

```bash
python3 scripts/sync_drift_detector.py --argocd --app prod-apps
```

### Quick CLI Commands

```bash
# List all applications across all clusters
argocd app list

# Check what is out of sync on production
argocd app diff prod-prod-cluster-frontend

# View sync history
argocd app get prod-prod-cluster-frontend --show-operation

# Manually sync a production app (after approval)
argocd app sync prod-prod-cluster-frontend
```

Reference: `references/troubleshooting.md`

---

## Summary Checklist

| Step | Action | Status |
|------|--------|--------|
| 1 | Install ArgoCD 3.x on management cluster | |
| 2 | Apply ArgoCD 3.x ConfigMaps (annotation tracking, resource exclusions) | |
| 3 | Expose ArgoCD via Ingress with TLS | |
| 4 | Register dev, staging, prod clusters with environment labels | |
| 5 | Create Git repo with monorepo + Kustomize overlay structure | |
| 6 | Create dev ApplicationSet with `automated` sync policy | |
| 7 | Create staging ApplicationSet with `automated` sync policy | |
| 8 | Create prod ApplicationSet WITHOUT `automated` (manual sync) | |
| 9 | Configure RBAC so only SRE team can sync production | |
| 10 | Set up secrets management (SOPS+age or External Secrets Operator) | |
| 11 | Validate repo structure with `validate_gitops_repo.py` | |
| 12 | Test end-to-end: commit to dev, auto-deploys; promote to prod, requires manual sync | |
| 13 | Set up monitoring with `check_argocd_health.py` and drift detection | |

---

## Key References from the Skill

| Resource | Purpose |
|----------|---------|
| `assets/argocd/install-argocd-3.x.yaml` | ArgoCD 3.x installation template |
| `assets/applicationsets/cluster-generator.yaml` | ApplicationSet cluster generator template |
| `assets/secrets/sops-age-config.yaml` | SOPS + age configuration template |
| `scripts/check_argocd_health.py` | Diagnose ArgoCD sync issues |
| `scripts/sync_drift_detector.py` | Detect configuration drift |
| `scripts/validate_gitops_repo.py` | Validate repository structure |
| `scripts/applicationset_generator.py` | Generate ApplicationSet manifests |
| `scripts/promotion_validator.py` | Validate promotion workflows |
| `scripts/secret_audit.py` | Audit secrets management |
| `references/multi_cluster.md` | Multi-cluster deployment patterns |
| `references/repo_patterns.md` | Repository structure guidance |
| `references/best_practices.md` | CNCF GitOps principles, 2025 recommendations |
| `references/secret_management.md` | Secrets management comparison and setup |
| `references/troubleshooting.md` | Common ArgoCD issues and debugging |
