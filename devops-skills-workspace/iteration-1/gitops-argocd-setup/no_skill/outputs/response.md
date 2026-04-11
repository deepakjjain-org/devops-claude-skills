# ArgoCD Multi-Cluster Setup: Dev/Staging/Prod with Differentiated Sync Policies

## Architecture Overview

The recommended approach is to run a **single ArgoCD management cluster** (often the staging cluster or a dedicated management cluster) that manages applications across all three clusters. This is simpler to operate than running ArgoCD in each cluster.

```
                    +---------------------+
                    |  ArgoCD Management  |
                    |  (dedicated or stg) |
                    +----------+----------+
                               |
              +----------------+----------------+
              |                |                |
         +----v----+     +----v----+     +-----v---+
         |   Dev   |     | Staging |     |  Prod   |
         | Cluster |     | Cluster |     | Cluster |
         +---------+     +---------+     +---------+
         auto-sync       auto-sync      manual sync
```

---

## Step 1: Install ArgoCD

Install ArgoCD on your management cluster using the official Helm chart or manifests.

### Option A: Helm Installation (Recommended)

```bash
# Add the ArgoCD Helm repo
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# Create namespace
kubectl create namespace argocd

# Install with custom values
helm install argocd argo/argo-cd \
  --namespace argocd \
  --version 7.7.x \
  --values argocd-values.yaml
```

### argocd-values.yaml

```yaml
server:
  replicas: 2
  ingress:
    enabled: true
    ingressClassName: nginx
    hosts:
      - argocd.yourdomain.com
    tls:
      - secretName: argocd-tls
        hosts:
          - argocd.yourdomain.com

controller:
  replicas: 1
  resources:
    requests:
      cpu: 500m
      memory: 512Mi
    limits:
      cpu: "2"
      memory: 2Gi

repoServer:
  replicas: 2
  resources:
    requests:
      cpu: 250m
      memory: 256Mi

redis-ha:
  enabled: true

configs:
  params:
    server.insecure: false
    application.resourceTrackingMethod: annotation
  rbac:
    policy.csv: |
      p, role:dev-team, applications, sync, dev/*, allow
      p, role:dev-team, applications, get, */*, allow
      p, role:platform-team, applications, *, */*, allow
      g, dev-team, role:dev-team
      g, platform-team, role:platform-team
```

### Option B: Plain Manifests

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

---

## Step 2: Register External Clusters

Each target cluster must be registered with ArgoCD. Use the ArgoCD CLI to add them.

```bash
# Login to ArgoCD
argocd login argocd.yourdomain.com --grpc-web

# Add dev cluster
argocd cluster add dev-cluster-context \
  --name dev \
  --label env=dev \
  --label tier=non-prod

# Add staging cluster
argocd cluster add staging-cluster-context \
  --name staging \
  --label env=staging \
  --label tier=non-prod

# Add prod cluster
argocd cluster add prod-cluster-context \
  --name prod \
  --label env=prod \
  --label tier=prod
```

This creates a ServiceAccount in each target cluster and stores the credentials as a Kubernetes secret in the ArgoCD namespace.

### Verify Cluster Connectivity

```bash
argocd cluster list
```

Expected output:
```
SERVER                          NAME     VERSION  STATUS      MESSAGE
https://dev-api.example.com     dev      1.29     Successful
https://staging-api.example.com staging  1.29     Successful
https://prod-api.example.com    prod     1.29     Successful
https://kubernetes.default.svc  in-cluster  1.29  Successful
```

---

## Step 3: Git Repository Structure

Use a **monorepo with environment overlays** using Kustomize or Helm values per environment.

```
gitops-repo/
├── apps/
│   ├── base/                          # Shared base manifests
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── hpa.yaml
│   │   └── kustomization.yaml
│   ├── overlays/
│   │   ├── dev/
│   │   │   ├── kustomization.yaml     # Dev-specific patches
│   │   │   ├── replicas-patch.yaml
│   │   │   └── config.env
│   │   ├── staging/
│   │   │   ├── kustomization.yaml
│   │   │   ├── replicas-patch.yaml
│   │   │   └── config.env
│   │   └── prod/
│   │       ├── kustomization.yaml
│   │       ├── replicas-patch.yaml
│   │       └── config.env
├── argocd/                            # ArgoCD application definitions
│   ├── appprojects/
│   │   ├── dev-project.yaml
│   │   ├── staging-project.yaml
│   │   └── prod-project.yaml
│   ├── applications/
│   │   ├── dev-app.yaml
│   │   ├── staging-app.yaml
│   │   └── prod-app.yaml
│   └── applicationsets/
│       └── multi-cluster-appset.yaml  # Alternative: single ApplicationSet
└── README.md
```

### Example Base Kustomization

```yaml
# apps/base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
  - hpa.yaml
```

### Example Dev Overlay

```yaml
# apps/overlays/dev/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
patches:
  - path: replicas-patch.yaml
configMapGenerator:
  - name: app-config
    envs:
      - config.env
```

```yaml
# apps/overlays/dev/replicas-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 1
```

### Example Prod Overlay

```yaml
# apps/overlays/prod/replicas-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  template:
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: my-app
```

---

## Step 4: ArgoCD Projects (RBAC Boundaries)

Projects provide security boundaries and control which clusters and repos each team can target.

### Dev Project

```yaml
# argocd/appprojects/dev-project.yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: dev
  namespace: argocd
spec:
  description: "Development environment"
  sourceRepos:
    - "https://github.com/your-org/gitops-repo.git"
  destinations:
    - server: "https://dev-api.example.com"
      namespace: "*"
  clusterResourceWhitelist:
    - group: "*"
      kind: "*"
  orphanedResources:
    warn: true
```

### Prod Project (Restrictive)

```yaml
# argocd/appprojects/prod-project.yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: prod
  namespace: argocd
spec:
  description: "Production environment - manual sync only"
  sourceRepos:
    - "https://github.com/your-org/gitops-repo.git"
  destinations:
    - server: "https://prod-api.example.com"
      namespace: "my-app"
    - server: "https://prod-api.example.com"
      namespace: "monitoring"
  clusterResourceWhitelist:
    - group: ""
      kind: Namespace
  namespaceResourceWhitelist:
    - group: "apps"
      kind: Deployment
    - group: ""
      kind: Service
    - group: ""
      kind: ConfigMap
    - group: ""
      kind: Secret
    - group: "networking.k8s.io"
      kind: Ingress
    - group: "autoscaling"
      kind: HorizontalPodAutoscaler
  orphanedResources:
    warn: true
  # Sync windows - only allow syncs during business hours
  syncWindows:
    - kind: allow
      schedule: "0 9-17 * * 1-5"   # Mon-Fri 9am-5pm
      duration: 8h
      applications:
        - "*"
      manualSync: true               # Allow manual sync within window
    - kind: deny
      schedule: "0 0 * * *"         # Deny all other times
      duration: 24h
      applications:
        - "*"
      manualSync: false              # Block even manual syncs outside window
```

---

## Step 5: Application Definitions with Sync Policies

This is the critical part -- configuring automatic sync for dev and manual sync for prod.

### Dev Application (Auto-Sync Enabled)

```yaml
# argocd/applications/dev-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app-dev
  namespace: argocd
  labels:
    env: dev
    app: my-app
  annotations:
    notifications.argoproj.io/subscribe.on-sync-succeeded.slack: dev-deployments
    notifications.argoproj.io/subscribe.on-sync-failed.slack: dev-alerts
spec:
  project: dev
  source:
    repoURL: "https://github.com/your-org/gitops-repo.git"
    targetRevision: main
    path: apps/overlays/dev
  destination:
    server: "https://dev-api.example.com"
    namespace: my-app
  syncPolicy:
    automated:                        # <-- AUTOMATIC SYNC
      prune: true                     # Remove resources not in Git
      selfHeal: true                  # Revert manual cluster changes
      allowEmpty: false               # Don't delete all if path is empty
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true
      - ApplyOutOfSyncOnly=true       # Only sync changed resources
    retry:
      limit: 3
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

### Staging Application (Auto-Sync, No Prune)

```yaml
# argocd/applications/staging-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app-staging
  namespace: argocd
  labels:
    env: staging
    app: my-app
  annotations:
    notifications.argoproj.io/subscribe.on-sync-succeeded.slack: staging-deployments
    notifications.argoproj.io/subscribe.on-sync-failed.slack: staging-alerts
spec:
  project: staging
  source:
    repoURL: "https://github.com/your-org/gitops-repo.git"
    targetRevision: main
    path: apps/overlays/staging
  destination:
    server: "https://staging-api.example.com"
    namespace: my-app
  syncPolicy:
    automated:                        # <-- AUTOMATIC SYNC
      prune: false                    # More cautious: don't auto-prune
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - ApplyOutOfSyncOnly=true
    retry:
      limit: 3
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

### Prod Application (Manual Sync Only)

```yaml
# argocd/applications/prod-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app-prod
  namespace: argocd
  labels:
    env: prod
    app: my-app
  annotations:
    notifications.argoproj.io/subscribe.on-sync-status-unknown.slack: prod-alerts
    notifications.argoproj.io/subscribe.on-health-degraded.slack: prod-alerts
    notifications.argoproj.io/subscribe.on-sync-succeeded.slack: prod-deployments
spec:
  project: prod
  source:
    repoURL: "https://github.com/your-org/gitops-repo.git"
    targetRevision: main              # Or use a release tag/branch
    path: apps/overlays/prod
  destination:
    server: "https://prod-api.example.com"
    namespace: my-app
  # NO syncPolicy.automated block = MANUAL SYNC REQUIRED
  syncPolicy:
    syncOptions:
      - CreateNamespace=false         # Namespaces managed separately in prod
      - PrunePropagationPolicy=foreground
      - PruneLast=true
      - ApplyOutOfSyncOnly=true
      - Validate=true
    retry:
      limit: 5
      backoff:
        duration: 10s
        factor: 2
        maxDuration: 5m
  # Show a diff preview before manual sync
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas            # Ignore if HPA manages replicas
```

**Key difference**: The `prod-app.yaml` has **no** `syncPolicy.automated` block. This means:
- ArgoCD will detect when the cluster is out of sync with Git
- It will show the diff in the UI
- A human must explicitly click "Sync" (or run `argocd app sync my-app-prod`) to apply changes

---

## Step 6: Alternative -- ApplicationSet (DRY Approach)

If you want to manage all three environments from a single definition, use an ApplicationSet with per-cluster overrides.

```yaml
# argocd/applicationsets/multi-cluster-appset.yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: my-app
  namespace: argocd
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
    - list:
        elements:
          - cluster: dev
            url: "https://dev-api.example.com"
            namespace: my-app
            project: dev
            autoSync: "true"
            prune: "true"
            selfHeal: "true"
            targetRevision: main
          - cluster: staging
            url: "https://staging-api.example.com"
            namespace: my-app
            project: staging
            autoSync: "true"
            prune: "false"
            selfHeal: "true"
            targetRevision: main
          - cluster: prod
            url: "https://prod-api.example.com"
            namespace: my-app
            project: prod
            autoSync: "false"
            prune: "false"
            selfHeal: "false"
            targetRevision: main
  template:
    metadata:
      name: "my-app-{{ .cluster }}"
      labels:
        env: "{{ .cluster }}"
        app: my-app
    spec:
      project: "{{ .project }}"
      source:
        repoURL: "https://github.com/your-org/gitops-repo.git"
        targetRevision: "{{ .targetRevision }}"
        path: "apps/overlays/{{ .cluster }}"
      destination:
        server: "{{ .url }}"
        namespace: "{{ .namespace }}"
      syncPolicy:
        syncOptions:
          - ApplyOutOfSyncOnly=true
          - PrunePropagationPolicy=foreground
        retry:
          limit: 3
          backoff:
            duration: 5s
            factor: 2
            maxDuration: 3m
  templatePatch: |
    {{- if eq .autoSync "true" }}
    spec:
      syncPolicy:
        automated:
          prune: {{ .prune }}
          selfHeal: {{ .selfHeal }}
    {{- end }}
```

**Note on ApplicationSet limitations**: The `templatePatch` field (available since ArgoCD 2.8+) is required to conditionally include or exclude the `automated` block. Without it, you cannot have some applications with auto-sync and others without from the same ApplicationSet.

---

## Step 7: Notifications for Manual Approval Workflow

Set up notifications so the team is alerted when prod is out of sync and awaiting manual approval.

### Install ArgoCD Notifications (included in ArgoCD 2.6+)

```yaml
# argocd-notifications-cm ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  service.slack: |
    token: $slack-token
    
  trigger.on-out-of-sync: |
    - when: app.status.sync.status == 'OutOfSync'
      send: [out-of-sync-notification]
      
  trigger.on-sync-succeeded: |
    - when: app.status.operationState.phase in ['Succeeded']
      send: [sync-succeeded-notification]
      
  trigger.on-sync-failed: |
    - when: app.status.operationState.phase in ['Error', 'Failed']
      send: [sync-failed-notification]
      
  trigger.on-health-degraded: |
    - when: app.status.health.status == 'Degraded'
      send: [health-degraded-notification]

  template.out-of-sync-notification: |
    slack:
      attachments: |
        [{
          "color": "#f4c030",
          "title": "{{ .app.metadata.name }} is OutOfSync",
          "text": "Application {{ .app.metadata.name }} in {{ .app.metadata.labels.env }} is out of sync.\n\nSync Status: {{ .app.status.sync.status }}\nHealth: {{ .app.status.health.status }}\n\n<{{ .context.argocdUrl }}/applications/{{ .app.metadata.name }}|View in ArgoCD>",
          "fields": [{
            "title": "Repository",
            "value": "{{ .app.spec.source.repoURL }}",
            "short": true
          },{
            "title": "Revision",
            "value": "{{ .app.status.sync.revision | trunc 7 }}",
            "short": true
          }]
        }]
        
  template.sync-succeeded-notification: |
    slack:
      attachments: |
        [{
          "color": "#18be52",
          "title": "{{ .app.metadata.name }} synced successfully",
          "text": "Application {{ .app.metadata.name }} in {{ .app.metadata.labels.env }} has been synced.\n\nRevision: {{ .app.status.sync.revision | trunc 7 }}"
        }]
        
  template.sync-failed-notification: |
    slack:
      attachments: |
        [{
          "color": "#e04f3a",
          "title": "{{ .app.metadata.name }} sync FAILED",
          "text": "Application {{ .app.metadata.name }} in {{ .app.metadata.labels.env }} failed to sync.\n\nMessage: {{ .app.status.operationState.message }}\n\n<{{ .context.argocdUrl }}/applications/{{ .app.metadata.name }}|View in ArgoCD>"
        }]
```

### Add Notification Annotations to Prod App

```yaml
metadata:
  annotations:
    # Notify on out-of-sync (triggers manual approval request)
    notifications.argoproj.io/subscribe.on-out-of-sync.slack: prod-approvals
    # Notify on successful sync
    notifications.argoproj.io/subscribe.on-sync-succeeded.slack: prod-deployments
    # Notify on failures
    notifications.argoproj.io/subscribe.on-sync-failed.slack: prod-alerts
    notifications.argoproj.io/subscribe.on-health-degraded.slack: prod-alerts
```

---

## Step 8: Promotion Workflow (Dev -> Staging -> Prod)

### Option A: Single Branch with Path-Based Overlays (Simpler)

All environments point to the `main` branch but different Kustomize overlay paths. Changes land in `apps/overlays/dev/` first, then are promoted by updating staging and prod overlays.

Workflow:
1. Developer opens PR modifying `apps/overlays/dev/`
2. PR merges to main -> ArgoCD auto-syncs dev
3. After validation, developer opens PR modifying `apps/overlays/staging/`
4. PR merges to main -> ArgoCD auto-syncs staging
5. After staging validation, developer opens PR modifying `apps/overlays/prod/`
6. PR merges to main -> ArgoCD detects OutOfSync for prod -> sends Slack notification
7. Platform engineer reviews diff in ArgoCD UI -> clicks Sync

### Option B: Branch-Based Promotion (More Isolation)

```
dev app     -> targetRevision: main
staging app -> targetRevision: release/staging
prod app    -> targetRevision: release/prod
```

Workflow:
1. Code merges to main -> auto-syncs to dev
2. When ready, merge main into `release/staging` -> auto-syncs to staging
3. When ready, merge `release/staging` into `release/prod` -> manual sync for prod

### Option C: Image Tag Promotion via Image Updater

Use ArgoCD Image Updater to automatically update image tags:

```yaml
# Dev: auto-update to latest tag
metadata:
  annotations:
    argocd-image-updater.argoproj.io/image-list: myapp=registry.example.com/my-app
    argocd-image-updater.argoproj.io/myapp.update-strategy: latest
```

```yaml
# Prod: only update to semver releases, still requires manual sync
metadata:
  annotations:
    argocd-image-updater.argoproj.io/image-list: myapp=registry.example.com/my-app
    argocd-image-updater.argoproj.io/myapp.update-strategy: semver
    argocd-image-updater.argoproj.io/myapp.allow-tags: "regexp:^v[0-9]+\\.[0-9]+\\.[0-9]+$"
```

---

## Step 9: Secrets Management

Never store plain secrets in Git. Use one of these approaches:

### Option A: Sealed Secrets

```bash
# Install Sealed Secrets controller in each cluster
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm install sealed-secrets sealed-secrets/sealed-secrets -n kube-system

# Encrypt a secret
kubeseal --format yaml < secret.yaml > sealed-secret.yaml
```

### Option B: External Secrets Operator (Recommended for Multi-Cluster)

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: my-app-secrets
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: my-app-secrets
  data:
    - secretKey: database-url
      remoteRef:
        key: /my-app/prod/database-url
```

### Option C: SOPS with Age Encryption

```yaml
# .sops.yaml in repo root
creation_rules:
  - path_regex: overlays/dev/.*secret.*\.yaml$
    age: "age1dev..."
  - path_regex: overlays/staging/.*secret.*\.yaml$
    age: "age1staging..."
  - path_regex: overlays/prod/.*secret.*\.yaml$
    age: "age1prod..."
```

---

## Step 10: Health Checks and Monitoring

### Custom Health Check for ArgoCD

```yaml
# In argocd-cm ConfigMap
data:
  resource.customizations.health.apps_Deployment: |
    hs = {}
    if obj.status ~= nil then
      if obj.status.availableReplicas ~= nil and obj.status.availableReplicas >= obj.spec.replicas then
        hs.status = "Healthy"
      else
        hs.status = "Progressing"
      end
    end
    return hs
```

### Prometheus Metrics

ArgoCD exposes metrics at `/metrics`. Key metrics to monitor:

```yaml
# Prometheus alert rules
groups:
  - name: argocd
    rules:
      - alert: ArgoAppOutOfSync
        expr: argocd_app_info{sync_status="OutOfSync", project="prod"} == 1
        for: 30m
        labels:
          severity: warning
        annotations:
          summary: "Production app {{ $labels.name }} has been OutOfSync for 30+ minutes"
          
      - alert: ArgoAppSyncFailed
        expr: argocd_app_sync_total{phase="Error"} > 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "ArgoCD sync failed for {{ $labels.name }}"
          
      - alert: ArgoAppUnhealthy
        expr: argocd_app_info{health_status!="Healthy", project="prod"} == 1
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: "Production app {{ $labels.name }} is unhealthy"
```

---

## Step 11: Applying Everything

```bash
# 1. Apply projects
kubectl apply -f argocd/appprojects/

# 2. Apply applications
kubectl apply -f argocd/applications/

# 3. Verify
argocd app list

# 4. Check dev auto-syncs
argocd app get my-app-dev

# 5. When ready, manually sync prod
argocd app sync my-app-prod --preview-changes
argocd app sync my-app-prod
```

---

## Quick Reference: Sync Policy Comparison

| Setting | Dev | Staging | Prod |
|---------|-----|---------|------|
| `automated` | Yes | Yes | **No** (manual) |
| `prune` | Yes | No | **No** |
| `selfHeal` | Yes | Yes | **No** |
| `CreateNamespace` | Yes | Yes | No |
| Sync Windows | None | None | Business hours only |
| Notifications | `#dev-deployments` | `#staging-deployments` | `#prod-approvals` |
| Branch/Revision | `main` | `main` | `main` (or release tag) |
| RBAC | Dev team can sync | Dev team can sync | **Platform team only** |

---

## Common Operations

### Manually Sync Prod with Dry Run

```bash
# Preview what will change
argocd app diff my-app-prod

# Sync with dry-run first
argocd app sync my-app-prod --dry-run

# Actual sync
argocd app sync my-app-prod

# Sync specific resources only
argocd app sync my-app-prod --resource apps:Deployment:my-app
```

### Rollback Prod

```bash
# List sync history
argocd app history my-app-prod

# Rollback to previous revision
argocd app rollback my-app-prod <history-id>
```

### Temporarily Disable Auto-Sync on Dev (e.g., during debugging)

```bash
# Disable auto-sync
argocd app set my-app-dev --sync-policy none

# Re-enable auto-sync
argocd app set my-app-dev --sync-policy automated --self-heal --auto-prune
```

---

## Security Recommendations

1. **RBAC**: Use SSO (OIDC) with group-based RBAC. Only platform/SRE teams should be able to sync prod.
2. **Audit Logging**: Enable ArgoCD audit logs and ship them to your SIEM.
3. **Network Policies**: Restrict ArgoCD's access to only the cluster API servers it needs.
4. **Git Webhooks**: Configure webhooks so ArgoCD detects changes immediately (default polling is every 3 minutes).
5. **Signed Commits**: Require GPG-signed commits on the GitOps repo, especially for prod paths.
6. **Repository Credentials**: Use SSH keys or deploy tokens, not personal access tokens.
7. **Drift Detection**: Enable `selfHeal` for non-prod to catch manual changes, but leave it off for prod to avoid surprises.

---

## Troubleshooting

| Problem | Cause | Solution |
|---------|-------|----------|
| App stuck in "Progressing" | Resource not becoming healthy | Check `argocd app get <app> --show-operation` and pod events |
| Sync fails with "ComparisonError" | Invalid manifests | Run `argocd app manifests <app>` to see rendered output |
| "Permission denied" on sync | RBAC misconfiguration | Check `argocd admin settings rbac can <role> sync applications <project>/*` |
| Cluster not reachable | ServiceAccount token expired | Re-add the cluster with `argocd cluster add` |
| Resources pruned unexpectedly | Path or filter misconfiguration | Set `prune: false` initially and use `argocd app diff` to verify |
| Webhook not triggering | Firewall or URL issue | Check ArgoCD server logs and ensure webhook URL is correct |
