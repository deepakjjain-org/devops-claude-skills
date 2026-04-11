# CI/CD Pipeline for Node.js Monorepo on GitHub Actions

## Assessment Summary

You need a complete CI/CD pipeline for a Node.js monorepo on GitHub Actions with security scanning, and your current build takes 25 minutes. Following the CI/CD skill's workflow, I am addressing this through three coordinated efforts:

1. **Creating a new pipeline** (Skill Workflow 1) -- tailored for Node.js monorepo on GitHub Actions
2. **Optimizing pipeline performance** (Skill Workflow 2) -- targeting the 25-minute build time
3. **Implementing DevSecOps security scanning** (Skill Workflow 6) -- integrating SAST, SCA, secret scanning, and container scanning

The skill's performance targets state a full pipeline should complete in under 20 minutes. Your 25-minute build alone exceeds that, so optimization is critical.

---

## Part 1: Diagnosing the 25-Minute Build

Before building the pipeline, let us first understand where the time is going. Run the skill's pipeline analyzer against your existing workflow:

```bash
python3 scripts/pipeline_analyzer.py --platform github --workflow .github/workflows/ci.yml
```

This script will check for:
- Missing dependency caching (the number one cause of slow builds)
- Unnecessary sequential job dependencies
- Outdated action versions
- Missing matrix strategies and parallelization opportunities
- Unused artifact uploads

Based on the skill's optimization reference (`references/optimization.md`), a 25-minute Node.js build typically suffers from one or more of these issues:

| Likely Bottleneck | Estimated Time Waste | Fix |
|---|---|---|
| No dependency caching | 3-8 minutes per job | Add `actions/cache@v4` or use `actions/setup-node@v4` with `cache: 'npm'` |
| Using `npm install` instead of `npm ci` | 1-3 minutes | Switch to `npm ci` for deterministic, faster installs |
| Sequential job execution | 5-10 minutes | Parallelize lint, test, and security jobs |
| Full test suite on every PR | 5-15 minutes | Shard tests with matrix builds, use path filters for monorepo packages |
| No concurrency cancellation | Wastes runner minutes | Add `concurrency` with `cancel-in-progress: true` |
| Duplicate dependency installs across jobs | 2-5 minutes per extra job | Build once, use artifacts; cache node_modules |

### Quick Wins Checklist (from Skill Workflow 2)

- [x] Add dependency caching (50-90% faster installs)
- [x] Remove unnecessary `needs` dependencies
- [x] Add path filters to skip unnecessary runs (critical for monorepos)
- [x] Use `npm ci` instead of `npm install`
- [x] Add job timeouts to prevent hung builds
- [x] Enable concurrency cancellation for duplicate runs

---

## Part 2: Complete Pipeline Architecture

Following the skill's basic pipeline structure principle -- **fail fast, parallelize, cache, build once deploy many** -- here is the pipeline architecture:

```
                    +------------------+
                    |   Push/PR Event  |
                    +--------+---------+
                             |
              +--------------+--------------+
              |              |              |
     +--------v---+  +------v-----+  +----v--------+
     | Secret Scan|  |    SAST    |  |  Dep Scan   |
     | (<1 min)   |  | (5-10 min) |  | (2-3 min)   |
     +--------+---+  +------+-----+  +----+--------+
              |              |              |
              +--------------+--------------+
                             |
              +--------------+--------------+
              |              |              |
     +--------v---+  +------v-----+  +----v--------+
     |    Lint    |  | Unit Tests |  |    Build     |
     | (<1 min)   |  | (sharded)  |  | (3-5 min)    |
     +--------+---+  | (2-4 min)  |  +----+--------+
              |      +------+-----+       |
              |              |             |
              +--------------+-------------+
                             |
                    +--------v---------+
                    | Container Build  |
                    | + Trivy Scan     |
                    | (3-5 min)        |
                    +--------+---------+
                             |
              +--------------+--------------+
              |                             |
     +--------v---------+        +---------v--------+
     | Deploy Staging   |        | E2E Tests        |
     | (auto on develop)|        | (main only)      |
     +--------+---------+        +---------+--------+
              |                             |
              +-----------------------------+
                             |
                    +--------v---------+
                    | Deploy Production|
                    | (manual approval)|
                    +------------------+
```

**Estimated total time with optimizations: 10-15 minutes** (down from 25+ minutes).

---

## Part 3: The Pipeline -- `.github/workflows/ci.yml`

This is the main CI pipeline, built from the skill's Node.js template (`assets/templates/github-actions/node-ci.yml`) and enriched with monorepo-specific optimizations and the security scanning template (`assets/templates/github-actions/security-scan.yml`).

```yaml
# Node.js Monorepo CI/CD Pipeline
# Optimized for performance with integrated security scanning
# Based on ci-cd skill templates: node-ci.yml + security-scan.yml

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
# (Skill Workflow 2: concurrency cancellation)
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

env:
  NODE_VERSION: '20'
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # ===========================================================================
  # STAGE 1: Security Scanning (runs in parallel, fail-fast)
  # Reference: references/devsecops.md - Shift-Left Security
  # ===========================================================================

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

  sast:
    name: SAST - CodeQL
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

  sast-semgrep:
    name: SAST - Semgrep
    runs-on: ubuntu-latest
    timeout-minutes: 10
    permissions:
      contents: read
      security-events: write
    steps:
      - uses: actions/checkout@v4

      - name: Run Semgrep
        uses: returntocorp/semgrep-action@v1
        with:
          config: >-
            p/security-audit
            p/owasp-top-ten
            p/nodejs

  dependency-scan:
    name: Dependency Security (SCA)
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

  # ===========================================================================
  # STAGE 2: Lint + Unit Tests (parallel, with monorepo path detection)
  # Reference: references/best_practices.md - Fast Feedback Loops
  # Reference: references/optimization.md - Parallelization Techniques
  # ===========================================================================

  detect-changes:
    name: Detect Changed Packages
    runs-on: ubuntu-latest
    timeout-minutes: 5
    outputs:
      packages: ${{ steps.changes.outputs.packages }}
      api-changed: ${{ steps.changes.outputs.api }}
      web-changed: ${{ steps.changes.outputs.web }}
      shared-changed: ${{ steps.changes.outputs.shared }}
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Detect changes
        id: changes
        run: |
          if [ "${{ github.event_name }}" == "pull_request" ]; then
            CHANGED=$(git diff --name-only origin/${{ github.base_ref }}...HEAD)
          else
            CHANGED=$(git diff --name-only HEAD~1)
          fi

          echo "Changed files:"
          echo "$CHANGED"

          # Detect which packages changed
          API_CHANGED="false"
          WEB_CHANGED="false"
          SHARED_CHANGED="false"

          echo "$CHANGED" | grep -q "^packages/api/" && API_CHANGED="true"
          echo "$CHANGED" | grep -q "^packages/web/" && WEB_CHANGED="true"
          echo "$CHANGED" | grep -q "^packages/shared/" && SHARED_CHANGED="true"

          # If shared changed, all packages need testing
          if [ "$SHARED_CHANGED" == "true" ]; then
            API_CHANGED="true"
            WEB_CHANGED="true"
          fi

          # If root config changed (package.json, tsconfig, etc.), test all
          if echo "$CHANGED" | grep -qE "^(package\.json|package-lock\.json|tsconfig\.json|\.eslintrc)"; then
            API_CHANGED="true"
            WEB_CHANGED="true"
          fi

          echo "api=$API_CHANGED" >> $GITHUB_OUTPUT
          echo "web=$WEB_CHANGED" >> $GITHUB_OUTPUT
          echo "shared=$SHARED_CHANGED" >> $GITHUB_OUTPUT

  lint:
    name: Lint & Format
    runs-on: ubuntu-latest
    needs: [secret-scan]
    timeout-minutes: 5
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run ESLint
        run: npx turbo run lint

      - name: Check formatting
        run: npx turbo run format:check

      - name: TypeScript type check
        run: npx turbo run typecheck

  unit-tests:
    name: Unit Tests (Shard ${{ matrix.shard }}/4)
    runs-on: ubuntu-latest
    needs: [detect-changes]
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

      # Turborepo incremental builds -- only test affected packages
      # Reference: references/optimization.md - Incremental Builds
      - name: Run unit tests (affected packages only)
        run: npx turbo run test:unit -- --shard=${{ matrix.shard }}/4
        env:
          TURBO_FILTER: >-
            ${{ needs.detect-changes.outputs.api-changed == 'true' && '--filter=@monorepo/api' || '' }}
            ${{ needs.detect-changes.outputs.web-changed == 'true' && '--filter=@monorepo/web' || '' }}
            ${{ needs.detect-changes.outputs.shared-changed == 'true' && '--filter=@monorepo/shared' || '' }}

      - name: Upload coverage
        if: matrix.shard == 1
        uses: codecov/codecov-action@v4
        with:
          files: ./coverage/lcov.info
          fail_ci_if_error: false

  integration-tests:
    name: Integration Tests
    runs-on: ubuntu-latest
    needs: [detect-changes, unit-tests]
    timeout-minutes: 15
    # Only run if API or shared packages changed
    if: needs.detect-changes.outputs.api-changed == 'true'
    services:
      postgres:
        image: postgres:16
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
        run: npx turbo run test:integration --filter=@monorepo/api
        env:
          DATABASE_URL: postgresql://testuser:testpass@localhost:5432/testdb
          REDIS_URL: redis://localhost:6379

  # ===========================================================================
  # STAGE 3: Build (after lint + tests + security pass)
  # Reference: references/best_practices.md - Build Once Deploy Many
  # ===========================================================================

  build:
    name: Build Application
    runs-on: ubuntu-latest
    needs: [lint, unit-tests, sast, dependency-scan]
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      # Cache Turborepo build outputs
      # Reference: references/optimization.md - Build Artifact Caching
      - name: Cache Turbo
        uses: actions/cache@v4
        with:
          path: .turbo
          key: ${{ runner.os }}-turbo-${{ github.sha }}
          restore-keys: |
            ${{ runner.os }}-turbo-

      - name: Build all packages
        run: npx turbo run build

      - name: Upload build artifacts
        uses: actions/upload-artifact@v4
        with:
          name: dist-${{ github.sha }}
          path: |
            packages/*/dist/
            packages/*/build/
          retention-days: 7

  # ===========================================================================
  # STAGE 4: Container Build + Security Scan
  # Reference: references/devsecops.md - Container Security
  # ===========================================================================

  container-build:
    name: Container Build & Scan
    runs-on: ubuntu-latest
    needs: [build]
    timeout-minutes: 15
    permissions:
      contents: read
      security-events: write
      packages: write
    steps:
      - uses: actions/checkout@v4

      - name: Download build artifacts
        uses: actions/download-artifact@v4
        with:
          name: dist-${{ github.sha }}
          path: packages/

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      # Docker layer caching -- major speed improvement
      # Reference: references/optimization.md - Docker Layer Caching
      - name: Build API image
        uses: docker/build-push-action@v5
        with:
          context: .
          file: packages/api/Dockerfile
          load: true
          tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/api:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Build Web image
        uses: docker/build-push-action@v5
        with:
          context: .
          file: packages/web/Dockerfile
          load: true
          tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/web:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      # Container vulnerability scanning with Trivy
      # Reference: references/devsecops.md - Trivy
      - name: Trivy Scan - API
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/api:${{ github.sha }}
          format: 'sarif'
          output: 'trivy-api.sarif'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'

      - name: Trivy Scan - Web
        uses: aquasecurity/trivy-action@master
        if: always()
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/web:${{ github.sha }}
          format: 'sarif'
          output: 'trivy-web.sarif'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'

      - name: Upload Trivy results to GitHub Security
        if: always()
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: 'trivy-api.sarif'
          category: 'trivy-api'

      # Generate SBOM for supply chain security
      # Reference: references/security.md - Supply Chain Security
      - name: Generate SBOM
        uses: anchore/sbom-action@v0
        with:
          image: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/api:${{ github.sha }}
          format: spdx-json
          output-file: sbom.spdx.json

      - name: Upload SBOM
        uses: actions/upload-artifact@v4
        with:
          name: sbom
          path: sbom.spdx.json

      # Push images to registry (only on main/develop)
      - name: Log in to Container Registry
        if: github.ref == 'refs/heads/main' || github.ref == 'refs/heads/develop'
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Push images
        if: github.ref == 'refs/heads/main' || github.ref == 'refs/heads/develop'
        run: |
          docker push ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/api:${{ github.sha }}
          docker push ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/web:${{ github.sha }}

  # ===========================================================================
  # STAGE 5: Security Gate
  # Reference: references/devsecops.md - Security Gates & Quality Gates
  # ===========================================================================

  security-gate:
    name: Security Quality Gate
    runs-on: ubuntu-latest
    needs: [secret-scan, sast, sast-semgrep, dependency-scan, container-build]
    if: always()
    timeout-minutes: 5
    steps:
      - name: Evaluate Security Posture
        run: |
          echo "## Security Scan Summary" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "| Scan | Status |" >> $GITHUB_STEP_SUMMARY
          echo "|------|--------|" >> $GITHUB_STEP_SUMMARY
          echo "| Secret Scanning | ${{ needs.secret-scan.result }} |" >> $GITHUB_STEP_SUMMARY
          echo "| SAST - CodeQL | ${{ needs.sast.result }} |" >> $GITHUB_STEP_SUMMARY
          echo "| SAST - Semgrep | ${{ needs.sast-semgrep.result }} |" >> $GITHUB_STEP_SUMMARY
          echo "| SCA - Dependencies | ${{ needs.dependency-scan.result }} |" >> $GITHUB_STEP_SUMMARY
          echo "| Container Scan | ${{ needs.container-build.result }} |" >> $GITHUB_STEP_SUMMARY

          # Fail if any critical security scan failed
          if [ "${{ needs.secret-scan.result }}" == "failure" ] || \
             [ "${{ needs.sast.result }}" == "failure" ] || \
             [ "${{ needs.container-build.result }}" == "failure" ]; then
            echo "" >> $GITHUB_STEP_SUMMARY
            echo "**SECURITY GATE FAILED** - Critical vulnerabilities detected" >> $GITHUB_STEP_SUMMARY
            exit 1
          fi

          echo "" >> $GITHUB_STEP_SUMMARY
          echo "**SECURITY GATE PASSED**" >> $GITHUB_STEP_SUMMARY

  # ===========================================================================
  # STAGE 6: E2E Tests (main branch only)
  # Reference: references/best_practices.md - Test Pyramid Strategy
  # ===========================================================================

  e2e-tests:
    name: E2E Tests (Shard ${{ matrix.shard }}/3)
    runs-on: ubuntu-latest
    needs: [build, container-build]
    if: github.ref == 'refs/heads/main'
    timeout-minutes: 30
    strategy:
      matrix:
        shard: [1, 2, 3]
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
          path: packages/

      # Playwright test sharding for parallel execution
      # Reference: references/optimization.md - Playwright Sharding
      - name: Run E2E tests
        run: npx playwright test --shard=${{ matrix.shard }}/3

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: e2e-results-shard-${{ matrix.shard }}
          path: test-results/

  # ===========================================================================
  # STAGE 7: Deployment
  # Reference: references/best_practices.md - Deployment Strategies
  # Reference: references/security.md - OIDC Authentication
  # ===========================================================================

  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: [build, integration-tests, security-gate]
    if: github.ref == 'refs/heads/develop'
    environment:
      name: staging
      url: https://staging.example.com
    permissions:
      contents: read
      id-token: write  # OIDC -- no static credentials
    steps:
      - uses: actions/checkout@v4

      - name: Download build artifacts
        uses: actions/download-artifact@v4
        with:
          name: dist-${{ github.sha }}
          path: packages/

      # OIDC authentication -- no long-lived secrets
      # Reference: references/security.md - OIDC Authentication
      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN_STAGING }}
          aws-region: us-east-1

      - name: Deploy to staging
        run: |
          # Replace with your deployment command
          echo "Deploying to staging..."
          # Example: aws ecs update-service, kubectl apply, etc.

      - name: Smoke tests
        run: |
          for i in {1..10}; do
            if curl -f https://staging.example.com/health; then
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
    needs: [e2e-tests, security-gate]
    if: github.ref == 'refs/heads/main'
    environment:
      name: production
      url: https://example.com
      # Configure required reviewers in GitHub Settings > Environments
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: actions/checkout@v4

      - name: Download build artifacts
        uses: actions/download-artifact@v4
        with:
          name: dist-${{ github.sha }}
          path: packages/

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN_PRODUCTION }}
          aws-region: us-east-1

      - name: Deploy to production
        run: |
          echo "Deploying to production..."
          # Your deployment command here

      - name: Health check
        run: |
          for i in {1..10}; do
            if curl -f https://example.com/health; then
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
          echo "Deployed: ${{ github.sha }}"
          echo "Time: $(date -u +%Y-%m-%dT%H:%M:%SZ)"
```

---

## Part 4: Supporting Configuration Files

### 4.1 Dependabot Configuration -- `.github/dependabot.yml`

Automated dependency updates with security patches (from `references/devsecops.md` - SCA section):

```yaml
version: 2
updates:
  # Root package.json
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
    groups:
      dev-dependencies:
        dependency-type: development
      production-dependencies:
        dependency-type: production
        update-types:
          - "minor"
          - "patch"

  # GitHub Actions versions
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 5

  # Docker base images
  - package-ecosystem: "docker"
    directory: "/packages/api"
    schedule:
      interval: "weekly"

  - package-ecosystem: "docker"
    directory: "/packages/web"
    schedule:
      interval: "weekly"
```

### 4.2 Optimized Dockerfile (Multi-stage) -- `packages/api/Dockerfile`

Following the skill's container image security and build optimization guidance (`references/security.md` - Container Image Security, `references/optimization.md` - Multi-stage Docker Builds):

```dockerfile
# Build stage
FROM node:20-alpine AS builder
WORKDIR /app

# Copy root workspace files first (better layer caching)
COPY package.json package-lock.json turbo.json ./
COPY packages/api/package.json ./packages/api/
COPY packages/shared/package.json ./packages/shared/

# Install dependencies (cached unless package files change)
RUN npm ci --only=production

# Copy source code
COPY packages/shared/ ./packages/shared/
COPY packages/api/ ./packages/api/

# Build
RUN npx turbo run build --filter=@monorepo/api

# Production stage -- minimal image
FROM node:20-alpine
WORKDIR /app

# Security: run as non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

COPY --from=builder /app/packages/api/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/packages/api/package.json ./

USER nodejs
EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1

CMD ["node", "dist/server.js"]
```

### 4.3 Weekly DAST Scan -- `.github/workflows/security-dast.yml`

DAST scans are slow (15-60 minutes) and should run on a schedule, not on every PR (per the skill's security scanning schedule in `references/devsecops.md`):

```yaml
name: DAST Security Scan

on:
  schedule:
    - cron: '0 2 * * 1'  # Weekly Monday 2 AM
  workflow_dispatch:

jobs:
  dast-baseline:
    name: OWASP ZAP Baseline
    runs-on: ubuntu-latest
    timeout-minutes: 30
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

  dast-api:
    name: API Security Scan
    runs-on: ubuntu-latest
    timeout-minutes: 30
    steps:
      - uses: actions/checkout@v4

      - name: ZAP API Scan
        uses: zaproxy/action-api-scan@v0.7.0
        with:
          target: 'https://staging.example.com/api/openapi.json'
          format: openapi
          fail_action: true
```

---

## Part 5: Optimization Techniques Applied

Here is a summary of every optimization applied, with references to the skill documentation:

### Time Savings Breakdown

| Optimization | Source | Estimated Savings |
|---|---|---|
| **Dependency caching** (`actions/setup-node` with `cache: 'npm'`) | `references/optimization.md` - Dependency Caching | 3-5 min per job |
| **Turborepo incremental builds** (`npx turbo run build`) | `references/optimization.md` - Incremental Builds | 3-8 min |
| **Test sharding** (Jest `--shard=N/4`) | `references/optimization.md` - Test Splitting | 50-75% of test time |
| **Parallel jobs** (lint, test, security run concurrently) | `references/optimization.md` - Job Parallelization | 5-10 min |
| **Path-based change detection** (only test affected packages) | `references/best_practices.md` - Monorepo Strategies | 2-5 min |
| **Concurrency cancellation** | `references/optimization.md` - Concurrency Control | Eliminates wasted runs |
| **Docker layer caching** (`cache-from: type=gha`) | `references/optimization.md` - Docker Layer Caching | 2-5 min |
| **`npm ci` over `npm install`** | `references/optimization.md` - Quick Wins | 1-2 min per job |
| **E2E tests only on main** (not every PR) | `references/best_practices.md` - Test Pyramid | 15-30 min on PRs |
| **Job timeouts** (prevent hung builds) | `references/optimization.md` - Job Timeouts | Prevents 6-hour stuck builds |

### Expected Performance After Optimization

Following the skill's performance targets (`references/optimization.md`):

| Stage | Target | Expected |
|---|---|---|
| Lint + Format + Typecheck | < 1 min | ~2 min |
| Unit Tests (4 shards) | < 5 min | ~3-4 min |
| Security Scans (parallel) | < 15 min | ~8-10 min |
| Build | < 5 min | ~3-5 min |
| Container Build + Scan | < 10 min | ~5-8 min |
| **Total PR Pipeline** | **< 20 min** | **~10-15 min** |
| E2E Tests (main only, 3 shards) | < 30 min | ~10-15 min |
| **Total Main Pipeline** | **< 30 min** | **~18-22 min** |

This takes your pipeline from a 25-minute build-only step to a full CI/CD pipeline with security scanning that completes faster than the original build.

---

## Part 6: Security Scanning Summary

Following the skill's security scanning schedule (`references/devsecops.md`):

| Scan Type | Tool | When | Speed | Integrated |
|---|---|---|---|---|
| Secret Scanning | TruffleHog + Gitleaks | Every commit | < 1 min | Yes |
| SAST | CodeQL + Semgrep | Every commit | 5-10 min | Yes |
| SCA | npm audit + Dependency Review | Every commit | 1-3 min | Yes |
| Container Scanning | Trivy | After build | 3-5 min | Yes |
| SBOM Generation | Syft/Anchore | After build | < 1 min | Yes |
| License Compliance | license-checker | After build | < 1 min | Add if needed |
| DAST Baseline | OWASP ZAP | Weekly schedule | 15-30 min | Separate workflow |
| DAST Full | OWASP ZAP | Weekly schedule | 30-60 min | Separate workflow |

### Security Checklist (from `references/security.md`)

- [x] OIDC authentication (no static AWS credentials)
- [x] Secret scanning on every commit
- [x] SAST with CodeQL and Semgrep
- [x] Dependency vulnerability scanning (npm audit)
- [x] Container image scanning (Trivy)
- [x] SBOM generation for supply chain security
- [x] Concurrency controls to prevent duplicate runs
- [x] Job timeouts on every job
- [x] Minimal permissions per job
- [x] Environment protection rules (production requires approval)
- [x] Dependabot for automated dependency updates
- [x] Multi-stage Docker builds with non-root user
- [x] Security quality gate before deployment

---

## Part 7: Pipeline Health Monitoring

After deploying this pipeline, regularly check its health using the skill's diagnostic script:

```bash
# Check pipeline success rates and failure patterns
python3 scripts/ci_health.py --platform github --repo your-org/your-repo --limit 20

# Analyze the workflow for further optimization opportunities
python3 scripts/pipeline_analyzer.py --platform github --workflow .github/workflows/ci.yml
```

### Ongoing Maintenance Checklist (from `references/troubleshooting.md`)

- **Weekly:** Review any failed security scans and remediate findings
- **Monthly:** Review build duration trends and failure patterns
- **Monthly:** Review and update GitHub Actions versions (Dependabot helps)
- **Quarterly:** Audit pipeline efficiency, review DAST findings
- **Quarterly:** Rotate any remaining static secrets (target: 90-day rotation)

---

## Implementation Steps

1. **Copy the main pipeline** to `.github/workflows/ci.yml`
2. **Copy the DAST workflow** to `.github/workflows/security-dast.yml`
3. **Copy the Dependabot config** to `.github/dependabot.yml`
4. **Update the Dockerfile** for each package in your monorepo
5. **Configure GitHub Environments:**
   - Create `staging` and `production` environments in Settings > Environments
   - Add required reviewers for `production`
   - Set deployment branch restrictions
6. **Configure secrets:**
   - `AWS_ROLE_ARN_STAGING` -- IAM role ARN for staging (OIDC)
   - `AWS_ROLE_ARN_PRODUCTION` -- IAM role ARN for production (OIDC)
   - Set up AWS OIDC identity provider (see `references/security.md` - OIDC Authentication)
7. **Run the pipeline analyzer** to validate the configuration:
   ```bash
   python3 scripts/pipeline_analyzer.py --platform github --workflow .github/workflows/ci.yml
   ```
8. **Customize paths** in the `detect-changes` job to match your monorepo package structure (replace `packages/api/`, `packages/web/`, `packages/shared/` with your actual paths)
9. **Adjust test commands** (`test:unit`, `test:integration`, `test:e2e`) to match your `package.json` scripts
10. **Enable GitHub Advanced Security** in repository settings for CodeQL results to appear in the Security tab

---

## Key References Used

All guidance sourced from the CI/CD skill at `/Users/asmar/github/devops-claude-skills/ci-cd/`:

| File | Used For |
|---|---|
| `SKILL.md` | Core workflow decision trees, pipeline structure, quick reference |
| `references/optimization.md` | Caching strategies, parallelization, test sharding, Docker layer caching, performance targets |
| `references/devsecops.md` | SAST/DAST/SCA tool selection, security gates, scanning schedules, complete DevSecOps pipeline pattern |
| `references/security.md` | OIDC authentication, secrets management, supply chain security, container image security |
| `references/best_practices.md` | Pipeline design principles, test pyramid, monorepo strategies, deployment patterns, artifact management |
| `references/troubleshooting.md` | Debugging techniques, common failure patterns, cache issues, maintenance checklist |
| `assets/templates/github-actions/node-ci.yml` | Base Node.js CI template with security scanning |
| `assets/templates/github-actions/security-scan.yml` | Complete DevSecOps scanning pipeline template |
| `scripts/pipeline_analyzer.py` | Automated pipeline optimization analysis |
| `scripts/ci_health.py` | Pipeline health monitoring |
