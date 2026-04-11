# Complete CI/CD Pipeline for Node.js Monorepo on GitHub Actions

## Overview

Based on your requirements -- a Node.js monorepo on GitHub Actions with security scanning and a 25-minute build -- this guide provides a production-ready pipeline that addresses three core concerns:

1. **Performance optimization** -- target: reduce from 25 minutes to under 12 minutes
2. **Security scanning** -- SAST, SCA, secret scanning, container scanning
3. **Monorepo awareness** -- only build/test what changed

The approach follows the CI/CD skill's core workflow: **Fail Fast -> Parallelize -> Cache -> Secure -> Deploy**.

---

## Part 1: Diagnosing the 25-Minute Build

Before implementing, run the pipeline analyzer to identify specific bottlenecks:

```bash
python3 scripts/pipeline_analyzer.py --platform github --workflow .github/workflows/ci.yml
```

Common causes for slow Node.js monorepo builds:

| Symptom | Likely Cause | Expected Savings |
|---------|-------------|-----------------|
| 3-5 min on dependency install | No caching or using `npm install` instead of `npm ci` | 2-4 min saved |
| Sequential jobs | Unnecessary `needs` dependencies between lint, test, build | 5-10 min saved |
| Full test suite on every PR | No path filters, no test sharding | 3-8 min saved |
| Redundant installs across jobs | Each job reinstalls from scratch | 2-3 min saved |
| No concurrency cancellation | Old runs keep running when new commits push | Wasted runner minutes |

### Quick Wins Checklist

Apply these immediately for the biggest impact:

- [ ] Add dependency caching (50-90% faster installs)
- [ ] Remove unnecessary `needs` dependencies between jobs
- [ ] Add path filters to skip unnecessary runs
- [ ] Use `npm ci` instead of `npm install` everywhere
- [ ] Add job timeouts to prevent hung builds
- [ ] Enable concurrency cancellation for duplicate runs
- [ ] Use Turborepo or Nx for affected-only builds

---

## Part 2: Optimized Pipeline Architecture

### Pipeline Stage Flow

```
Commit Push / PR
    |
    v
+---------------------------+  (parallel, <1 min each)
| secret-scan | lint | sast |
| dep-scan    |             |
+---------------------------+
    |
    v
+---------------------------+  (parallel, sharded, 3-5 min)
| unit-tests (shard 1/4)   |
| unit-tests (shard 2/4)   |
| unit-tests (shard 3/4)   |
| unit-tests (shard 4/4)   |
+---------------------------+
    |
    v
+---------------------------+  (parallel, 3-5 min)
| build (affected packages) |
| integration-tests         |
+---------------------------+
    |
    v
+---------------------------+  (main branch only)
| container-scan            |
| e2e-tests (sharded)      |
+---------------------------+
    |
    v
+---------------------------+
| deploy-staging -> deploy-production (with approval)
+---------------------------+
```

### Target Timing Breakdown

| Stage | Current (est.) | Optimized Target |
|-------|---------------|-----------------|
| Dependency Install | 3-5 min | 15-30 sec (cached) |
| Lint + Format | 2 min | 30 sec (parallel) |
| Unit Tests | 8-10 min | 2-3 min (sharded x4) |
| Integration Tests | 5-7 min | 3-4 min (parallel) |
| Build | 5-7 min | 2-3 min (affected only) |
| Security Scans | N/A (new) | 2-5 min (parallel) |
| **Total** | **25 min** | **8-12 min** |

---

## Part 3: Complete Workflow Configuration

### File: `.github/workflows/ci.yml`

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
    paths-ignore:
      - '**.md'
      - 'docs/**'
      - '.vscode/**'
  pull_request:
    branches: [main]
    paths-ignore:
      - '**.md'
      - 'docs/**'

# Cancel in-progress runs for the same branch/PR
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

env:
  NODE_VERSION: '20'
  TURBO_TOKEN: ${{ secrets.TURBO_TOKEN }}
  TURBO_TEAM: ${{ vars.TURBO_TEAM }}

jobs:
  # ============================================================
  # STAGE 1: Fast Feedback (parallel, <2 min)
  # ============================================================

  # Security: Secret Scanning
  secret-scan:
    name: Secret Scanning
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: TruffleHog Secret Scan
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}
          head: HEAD

      - name: Gitleaks
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  # Security: SAST (Static Application Security Testing)
  sast:
    name: SAST Analysis
    runs-on: ubuntu-latest
    timeout-minutes: 15
    permissions:
      contents: read
      security-events: write
    steps:
      - uses: actions/checkout@v4

      - name: Initialize CodeQL
        uses: github/codeql-action/init@v3
        with:
          languages: javascript
          queries: security-and-quality

      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v3

      - name: Run Semgrep
        uses: returntocorp/semgrep-action@v1
        with:
          config: >-
            p/security-audit
            p/owasp-top-ten
            p/nodejs

  # Security: Dependency Scanning (SCA)
  dependency-scan:
    name: Dependency Security
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: npm audit
        run: npm audit --audit-level=high

      - name: Dependency Review (PR only)
        if: github.event_name == 'pull_request'
        uses: actions/dependency-review-action@v4
        with:
          fail-on-severity: high

  # Determine affected packages (monorepo optimization)
  detect-changes:
    name: Detect Changes
    runs-on: ubuntu-latest
    timeout-minutes: 5
    outputs:
      packages: ${{ steps.filter.outputs.changes }}
      api-changed: ${{ steps.filter.outputs.api }}
      web-changed: ${{ steps.filter.outputs.web }}
      shared-changed: ${{ steps.filter.outputs.shared }}
    steps:
      - uses: actions/checkout@v4

      - uses: dorny/paths-filter@v3
        id: filter
        with:
          filters: |
            api:
              - 'packages/api/**'
              - 'packages/shared/**'
            web:
              - 'packages/web/**'
              - 'packages/shared/**'
            shared:
              - 'packages/shared/**'

  # Lint (runs in parallel with security scans)
  lint:
    name: Lint & Format
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      # Use Turborepo to lint only affected packages
      - name: Lint affected packages
        run: npx turbo run lint --filter=[HEAD^1]

      - name: Check formatting
        run: npx turbo run format:check --filter=[HEAD^1]

  # ============================================================
  # STAGE 2: Unit Tests (sharded for speed, 2-3 min)
  # ============================================================

  unit-test:
    name: Unit Tests (shard ${{ matrix.shard }}/${{ strategy.job-total }})
    runs-on: ubuntu-latest
    needs: [secret-scan, lint]
    timeout-minutes: 15
    strategy:
      matrix:
        shard: [1, 2, 3, 4]
      fail-fast: false
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run unit tests (shard ${{ matrix.shard }}/4)
        run: npx turbo run test:unit -- --shard=${{ matrix.shard }}/4

      - name: Upload coverage
        if: matrix.shard == 1
        uses: codecov/codecov-action@v4
        with:
          files: ./coverage/lcov.info
          fail_ci_if_error: false

  # ============================================================
  # STAGE 3: Build & Integration Tests (parallel, 3-5 min)
  # ============================================================

  build:
    name: Build
    runs-on: ubuntu-latest
    needs: [unit-test, sast, dependency-scan]
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      # Turborepo: build only affected packages with remote caching
      - name: Build affected packages
        run: npx turbo run build --filter=[HEAD^1]

      - name: Upload build artifacts
        uses: actions/upload-artifact@v4
        with:
          name: dist-${{ github.sha }}
          path: |
            packages/*/dist/
          retention-days: 7

  integration-test:
    name: Integration Tests
    runs-on: ubuntu-latest
    needs: [unit-test]
    timeout-minutes: 20

    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
      redis:
        image: redis:7-alpine
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run integration tests
        run: npx turbo run test:integration
        env:
          DATABASE_URL: postgres://testuser:testpass@localhost:5432/testdb
          REDIS_URL: redis://localhost:6379

  # ============================================================
  # STAGE 4: Container Build & Security (main branch, 3-5 min)
  # ============================================================

  docker-build:
    name: Docker Build & Scan
    runs-on: ubuntu-latest
    needs: [build, integration-test]
    if: github.ref == 'refs/heads/main' || github.ref == 'refs/heads/develop'
    timeout-minutes: 20
    permissions:
      contents: read
      security-events: write
      packages: write
    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          load: true
          tags: |
            ghcr.io/${{ github.repository }}:${{ github.sha }}
            ghcr.io/${{ github.repository }}:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max

      # Container security scanning with Trivy
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ghcr.io/${{ github.repository }}:${{ github.sha }}
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'

      - name: Upload Trivy results to GitHub Security tab
        if: always()
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: 'trivy-results.sarif'

      # Generate SBOM
      - name: Generate SBOM
        uses: anchore/sbom-action@v0
        with:
          image: ghcr.io/${{ github.repository }}:${{ github.sha }}
          format: spdx-json
          output-file: sbom.spdx.json

      - name: Push image
        run: docker push ghcr.io/${{ github.repository }}:${{ github.sha }}

  # ============================================================
  # STAGE 5: E2E Tests (main branch only, sharded)
  # ============================================================

  e2e-test:
    name: E2E Tests (shard ${{ matrix.shardIndex }}/${{ matrix.shardTotal }})
    runs-on: ubuntu-latest
    needs: [build]
    if: github.ref == 'refs/heads/main'
    timeout-minutes: 30
    strategy:
      matrix:
        shardIndex: [1, 2, 3]
        shardTotal: [3]
      fail-fast: false
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright browsers
        run: npx playwright install --with-deps chromium

      - name: Download build artifacts
        uses: actions/download-artifact@v4
        with:
          name: dist-${{ github.sha }}

      - name: Run E2E tests
        run: npx playwright test --shard=${{ matrix.shardIndex }}/${{ matrix.shardTotal }}

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: playwright-report-${{ matrix.shardIndex }}
          path: playwright-report/

  # ============================================================
  # STAGE 6: Security Gate
  # ============================================================

  security-gate:
    name: Security Quality Gate
    runs-on: ubuntu-latest
    needs: [secret-scan, sast, dependency-scan, docker-build]
    if: always() && (github.ref == 'refs/heads/main' || github.ref == 'refs/heads/develop')
    timeout-minutes: 5
    steps:
      - name: Evaluate security posture
        run: |
          echo "## Security Scan Summary" >> $GITHUB_STEP_SUMMARY

          FAILED=false

          if [ "${{ needs.secret-scan.result }}" == "failure" ]; then
            echo "- FAILED: Secret Scanning" >> $GITHUB_STEP_SUMMARY
            FAILED=true
          else
            echo "- PASSED: Secret Scanning" >> $GITHUB_STEP_SUMMARY
          fi

          if [ "${{ needs.sast.result }}" == "failure" ]; then
            echo "- FAILED: SAST (CodeQL + Semgrep)" >> $GITHUB_STEP_SUMMARY
            FAILED=true
          else
            echo "- PASSED: SAST (CodeQL + Semgrep)" >> $GITHUB_STEP_SUMMARY
          fi

          if [ "${{ needs.dependency-scan.result }}" == "failure" ]; then
            echo "- FAILED: Dependency Scanning" >> $GITHUB_STEP_SUMMARY
            FAILED=true
          else
            echo "- PASSED: Dependency Scanning" >> $GITHUB_STEP_SUMMARY
          fi

          if [ "${{ needs.docker-build.result }}" == "failure" ]; then
            echo "- FAILED: Container Scanning" >> $GITHUB_STEP_SUMMARY
            FAILED=true
          else
            echo "- PASSED: Container Scanning" >> $GITHUB_STEP_SUMMARY
          fi

          if [ "$FAILED" == "true" ]; then
            echo ""
            echo "**Security gate FAILED** - address findings before deploying" >> $GITHUB_STEP_SUMMARY
            exit 1
          else
            echo ""
            echo "**Security gate PASSED**" >> $GITHUB_STEP_SUMMARY
          fi

  # ============================================================
  # STAGE 7: Deployment
  # ============================================================

  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: [build, integration-test, security-gate]
    if: github.ref == 'refs/heads/develop'
    environment:
      name: staging
      url: https://staging.example.com
    permissions:
      contents: read
      id-token: write  # OIDC
    steps:
      - uses: actions/checkout@v4

      - name: Download build artifacts
        uses: actions/download-artifact@v4
        with:
          name: dist-${{ github.sha }}

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN_STAGING }}
          aws-region: us-east-1

      - name: Deploy
        run: |
          # Replace with your deployment command
          echo "Deploying to staging..."
          # aws s3 sync dist/ s3://${{ secrets.STAGING_BUCKET }}
          # kubectl set image deployment/app app=ghcr.io/${{ github.repository }}:${{ github.sha }}

      - name: Smoke tests
        run: |
          for i in {1..10}; do
            if curl -sf https://staging.example.com/health; then
              echo "Health check passed"
              exit 0
            fi
            echo "Attempt $i failed, retrying..."
            sleep 10
          done
          echo "Health check failed"
          exit 1

  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: [e2e-test, security-gate]
    if: github.ref == 'refs/heads/main'
    environment:
      name: production  # Requires manual approval in GitHub Settings
      url: https://example.com
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: actions/checkout@v4

      - name: Download build artifacts
        uses: actions/download-artifact@v4
        with:
          name: dist-${{ github.sha }}

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN_PRODUCTION }}
          aws-region: us-east-1

      - name: Deploy
        run: |
          echo "Deploying to production..."
          # Replace with your deployment command

      - name: Health check
        run: |
          for i in {1..10}; do
            if curl -sf https://example.com/health; then
              echo "Health check passed"
              exit 0
            fi
            echo "Attempt $i failed, retrying..."
            sleep 10
          done
          echo "Health check failed"
          exit 1

      - name: Create deployment record
        run: |
          echo "Deployed version: ${{ github.sha }}"
          echo "Deployment time: $(date -u +%Y-%m-%dT%H:%M:%SZ)"
```

---

## Part 4: Monorepo-Specific Optimizations

### Option A: Turborepo (Recommended)

Turborepo gives you the biggest performance gain for monorepos by only running tasks for packages that actually changed.

**Add to your root `package.json`:**

```json
{
  "devDependencies": {
    "turbo": "^2"
  }
}
```

**`turbo.json` configuration:**

```json
{
  "$schema": "https://turbo.build/schema.json",
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**"]
    },
    "test:unit": {
      "dependsOn": ["^build"]
    },
    "test:integration": {
      "dependsOn": ["build"]
    },
    "lint": {},
    "format:check": {}
  }
}
```

Key CI commands:

```bash
# Build only packages changed since last commit
npx turbo run build --filter=[HEAD^1]

# Build a specific package and its dependencies
npx turbo run build --filter=@myorg/api...

# Run all tasks with remote caching (saves across CI runs)
npx turbo run build test:unit lint --filter=[HEAD^1]
```

**Enable remote caching** for even faster builds across CI runs by setting `TURBO_TOKEN` and `TURBO_TEAM` as repository secrets.

### Option B: Nx

If you prefer Nx:

```bash
# Run affected targets only
npx nx affected --target=build --base=origin/main
npx nx affected --target=test --base=origin/main
npx nx affected --target=lint --base=origin/main
```

---

## Part 5: Caching Strategy (Deep Dive)

### Layer 1: npm Dependency Cache

The `actions/setup-node@v4` with `cache: 'npm'` handles this automatically. It caches `~/.npm` and restores based on `package-lock.json` hash.

### Layer 2: Turborepo Remote Cache

Set up Vercel Remote Cache or self-hosted:

```yaml
env:
  TURBO_TOKEN: ${{ secrets.TURBO_TOKEN }}
  TURBO_TEAM: ${{ vars.TURBO_TEAM }}
```

This caches build outputs across CI runs -- if a package hasn't changed, Turbo restores its output from cache in seconds.

### Layer 3: Docker Layer Cache

```yaml
- uses: docker/build-push-action@v5
  with:
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

### Layer 4: TypeScript Incremental Compilation

Add to `tsconfig.json`:

```json
{
  "compilerOptions": {
    "incremental": true,
    "tsBuildInfoFile": ".tsbuildinfo"
  }
}
```

Cache the build info:

```yaml
- uses: actions/cache@v4
  with:
    path: |
      packages/*/.tsbuildinfo
    key: ts-build-${{ hashFiles('packages/*/src/**/*.ts') }}
```

### Layer 5: Playwright Browser Cache (for E2E)

```yaml
- uses: actions/cache@v4
  with:
    path: ~/.cache/ms-playwright
    key: playwright-${{ hashFiles('**/package-lock.json') }}
```

---

## Part 6: Security Scanning Summary

The pipeline integrates five types of security scanning, following DevSecOps best practices:

| Scan Type | Tool | When | Speed | What It Finds |
|-----------|------|------|-------|---------------|
| Secret Scanning | TruffleHog + Gitleaks | Every commit | <1 min | Exposed credentials, API keys |
| SAST | CodeQL + Semgrep | Every commit | 5-15 min | Code vulnerabilities (XSS, injection, etc.) |
| SCA | npm audit + Dependency Review | Every commit | 1-3 min | Vulnerable dependencies |
| Container Scan | Trivy | Main/develop push | 3-5 min | Image vulnerabilities |
| SBOM | Syft | Main/develop push | 1-2 min | Full dependency inventory |

### Security Gate Pattern

The `security-gate` job evaluates all scan results and blocks deployment if critical issues are found. This ensures no vulnerable code reaches production.

### Adding DAST (Optional, Scheduled)

For runtime vulnerability testing, add a weekly DAST scan. Create a separate workflow file:

**`.github/workflows/dast.yml`:**

```yaml
name: DAST Security Scan

on:
  schedule:
    - cron: '0 3 * * 1'  # Weekly Monday 3 AM
  workflow_dispatch:

jobs:
  dast:
    name: OWASP ZAP Scan
    runs-on: ubuntu-latest
    timeout-minutes: 60
    steps:
      - uses: actions/checkout@v4

      - name: ZAP Baseline Scan
        uses: zaproxy/action-baseline@v0.10.0
        with:
          target: 'https://staging.example.com'
          rules_file_name: '.zap/rules.tsv'
          fail_action: true

      - name: Upload ZAP report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: zap-report
          path: report_html.html
```

---

## Part 7: Automated Dependency Updates

**`.github/dependabot.yml`:**

```yaml
version: 2
updates:
  # npm dependencies
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
    groups:
      dev-dependencies:
        dependency-type: "development"
      production-dependencies:
        dependency-type: "production"

  # GitHub Actions
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 5
```

---

## Part 8: Dockerfile (Optimized for Security & Speed)

```dockerfile
# Build stage
FROM node:20-alpine AS builder
WORKDIR /app

# Install dependencies first (better caching)
COPY package*.json ./
COPY packages/*/package*.json ./packages/
RUN npm ci --only=production

# Copy source and build
COPY . .
RUN npx turbo run build

# Production stage (minimal image)
FROM node:20-alpine
WORKDIR /app

# Security: run as non-root
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

COPY --from=builder --chown=nodejs:nodejs /app/packages/api/dist ./dist
COPY --from=builder --chown=nodejs:nodejs /app/node_modules ./node_modules

USER nodejs
EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1

CMD ["node", "dist/server.js"]
```

---

## Part 9: Required GitHub Configuration

### Repository Settings

1. **Branch Protection (Settings > Branches > Branch protection rules):**
   - Require pull request reviews (at least 1)
   - Require status checks to pass before merging: `lint`, `unit-test`, `build`, `sast`, `dependency-scan`, `secret-scan`
   - Require branches to be up to date
   - Include administrators

2. **Environments (Settings > Environments):**
   - `staging` -- auto-deploy from `develop`
   - `production` -- require manual approval (add required reviewers), restrict to `main` branch

3. **Secrets (Settings > Secrets and variables > Actions):**
   - `AWS_ROLE_ARN_STAGING` -- OIDC role for staging
   - `AWS_ROLE_ARN_PRODUCTION` -- OIDC role for production
   - `TURBO_TOKEN` -- Turborepo remote cache token (optional)

4. **Variables (Settings > Secrets and variables > Actions > Variables):**
   - `TURBO_TEAM` -- Turborepo team name

5. **Security Features (Settings > Code security and analysis):**
   - Enable Dependabot alerts
   - Enable Dependabot security updates
   - Enable secret scanning
   - Enable push protection

### OIDC Setup for AWS

Create an IAM OIDC identity provider and role. Trust policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
      },
      "StringLike": {
        "token.actions.githubusercontent.com:sub": "repo:YOUR_ORG/YOUR_REPO:*"
      }
    }
  }]
}
```

This eliminates static AWS credentials entirely -- a major security improvement.

---

## Part 10: Monitoring Pipeline Health

### Track these metrics monthly:

| Metric | Target | How to Track |
|--------|--------|-------------|
| Build duration | <12 min (full), <8 min (PR) | GitHub Actions timing |
| Cache hit rate | >80% | `actions/cache` output |
| Failure rate | <5% | `scripts/ci_health.py` |
| Security findings | 0 critical, <5 high | GitHub Security tab |
| Deployment frequency | Multiple per week | Release history |
| Flaky test rate | <2% | Track retried tests |

### Run health checks periodically:

```bash
python3 scripts/ci_health.py --platform github --repo your-org/your-repo --limit 20
```

---

## Implementation Plan

### Week 1: Foundation
1. Add `turbo.json` and configure Turborepo
2. Deploy the main `.github/workflows/ci.yml` with caching and parallelization
3. Enable concurrency cancellation
4. Add path filters for monorepo packages
5. **Expected result: build time drops to ~12-15 min**

### Week 2: Test Optimization
1. Implement Jest test sharding (4 shards)
2. Add Playwright E2E sharding (3 shards)
3. Set up Turborepo remote caching
4. Split integration tests from unit tests
5. **Expected result: build time drops to ~8-12 min**

### Week 3: Security Integration
1. Add secret scanning (TruffleHog + Gitleaks)
2. Enable CodeQL and Semgrep SAST
3. Add npm audit and dependency review
4. Configure Dependabot
5. Set up security gate job

### Week 4: Deployment & Hardening
1. Configure OIDC for AWS authentication
2. Set up staging and production environments with approval gates
3. Add Docker build with Trivy container scanning
4. Enable branch protection with required status checks
5. Set up DAST weekly scan on staging
6. Add pipeline health monitoring

---

## Key Principles Applied

These recommendations follow the CI/CD skill's design philosophy:

1. **Fail fast** -- Secret scanning and linting run first, giving developers feedback in under 2 minutes
2. **Parallelize** -- Security scans, lint, and change detection all run concurrently in Stage 1
3. **Cache aggressively** -- Five layers of caching (npm, Turbo, Docker, TypeScript, Playwright)
4. **Build once, deploy many** -- Artifacts built once and reused across staging and production
5. **Shift security left** -- SAST and SCA run on every PR, not just before production
6. **Gate deployments** -- Security quality gate blocks deployment on critical findings
7. **Use OIDC** -- No static credentials; short-lived tokens via OIDC federation
8. **Monorepo awareness** -- Only build, test, and lint packages that actually changed
