# Complete CI/CD Pipeline for Node.js Monorepo on GitHub Actions

## Overview

Based on your requirements -- a Node.js monorepo on GitHub Actions with security scanning and a 25-minute build that needs optimization -- here is a comprehensive plan following a structured approach: **Design the pipeline -> Optimize performance -> Integrate security scanning -> Set up deployment**.

Your 25-minute build time is above the recommended target of under 20 minutes for a full pipeline. With the optimizations below, you should be able to cut this to 10-12 minutes for most PR runs, and under 18 minutes for full main-branch runs including E2E tests.

---

## Step 1: Analyze Your Current Pipeline

Before making changes, run the pipeline analyzer script to identify specific bottlenecks in your existing configuration:

```bash
python3 scripts/pipeline_analyzer.py --platform github --workflow .github/workflows/ci.yml
```

This will identify missing caching opportunities, unnecessary sequential execution, outdated action versions, unused artifacts, and overly broad triggers. You can also analyze recent runs to find patterns:

```bash
python3 scripts/pipeline_analyzer.py --platform github --repo your-org/your-repo --analyze-runs 10
```

---

## Step 2: Optimized Pipeline Architecture

For a Node.js monorepo, the pipeline should follow a **fail-fast** structure where cheap validation runs first and expensive operations run later and in parallel. Here is the recommended job flow:

```
                    +----------------+
                    | secret-scan    |  (~1 min)
                    +-------+--------+
                            |
              +-------------+-------------+
              |             |             |
        +-----v----+  +----v-----+  +----v-----------+
        |  lint     |  |  sast    |  | dependency-scan|   (parallel, ~5 min)
        +-----+----+  +----+-----+  +----+-----------+
              |             |             |
              +-------------+-------------+
                            |
              +-------------+-------------+
              |                           |
        +-----v-----------+    +----------v--------+
        | test (sharded   |    | build             |   (parallel, ~5 min)
        |  across matrix) |    |                   |
        +---------+-------+    +----------+--------+
                  |                       |
                  +-----------+-----------+
                              |
                    +---------v---------+
                    | e2e (main only)   |  (~10 min, main branch only)
                    +---------+---------+
                              |
              +---------------+---------------+
              |                               |
     +--------v--------+          +-----------v---------+
     | deploy-staging   |          | deploy-production   |
     | (develop branch) |          | (main, with approval)|
     +-----------------+          +---------------------+
```

---

## Step 3: Complete Workflow Configuration

Here is the full GitHub Actions workflow file. Save this as `.github/workflows/ci.yml`:

```yaml
name: Node.js Monorepo CI/CD

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

jobs:
  # ============================================================
  # STAGE 1: Secret Scanning (fast gate, blocks everything)
  # ============================================================
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

  # ============================================================
  # STAGE 2: Fast Validation (parallel after secret scan)
  # ============================================================
  lint:
    name: Lint & Format
    runs-on: ubuntu-latest
    needs: [secret-scan]
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run linter
        run: npm run lint

      - name: Check formatting
        run: npm run format:check

  sast:
    name: SAST Analysis
    runs-on: ubuntu-latest
    needs: [secret-scan]
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

  dependency-scan:
    name: Dependency Security
    runs-on: ubuntu-latest
    needs: [secret-scan]
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: npm audit
        run: |
          npm audit --audit-level=moderate --json > npm-audit.json || true
          npm audit --audit-level=high

      - name: Dependency Review (PR only)
        if: github.event_name == 'pull_request'
        uses: actions/dependency-review-action@v4
        with:
          fail-on-severity: high

      - name: Upload audit results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: npm-audit-report
          path: npm-audit.json

  # ============================================================
  # STAGE 3: Monorepo-Aware Testing (sharded for speed)
  # ============================================================
  detect-changes:
    name: Detect Changed Packages
    runs-on: ubuntu-latest
    needs: [secret-scan]
    timeout-minutes: 5
    outputs:
      packages: ${{ steps.changes.outputs.packages }}
      shared-changed: ${{ steps.changes.outputs.shared-changed }}
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Detect changed packages
        id: changes
        run: |
          if [ "${{ github.event_name }}" == "pull_request" ]; then
            CHANGED=$(git diff --name-only origin/${{ github.base_ref }}...HEAD | cut -d'/' -f1-2 | sort -u)
          else
            CHANGED=$(git diff --name-only HEAD~1 | cut -d'/' -f1-2 | sort -u)
          fi
          echo "Changed paths: $CHANGED"

          # Check if shared code changed (requires full test run)
          if echo "$CHANGED" | grep -q "^packages/shared"; then
            echo "shared-changed=true" >> $GITHUB_OUTPUT
          else
            echo "shared-changed=false" >> $GITHUB_OUTPUT
          fi

          echo "packages=$CHANGED" >> $GITHUB_OUTPUT

  test:
    name: Test (Shard ${{ matrix.shard }}/4)
    runs-on: ubuntu-latest
    needs: [lint, detect-changes]
    timeout-minutes: 15
    strategy:
      matrix:
        shard: [1, 2, 3, 4]
      fail-fast: false
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run unit tests (sharded)
        run: npx jest --shard=${{ matrix.shard }}/4 --ci --coverage
        env:
          CI: true

      - name: Upload coverage
        if: matrix.shard == 1
        uses: codecov/codecov-action@v4
        with:
          files: ./coverage/lcov.info
          fail_ci_if_error: false

  integration-test:
    name: Integration Tests
    runs-on: ubuntu-latest
    needs: [lint, detect-changes]
    timeout-minutes: 15
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_DB: testdb
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
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run integration tests
        run: npm run test:integration
        env:
          DATABASE_URL: postgresql://postgres:testpass@localhost:5432/testdb
          REDIS_URL: redis://localhost:6379

  # ============================================================
  # STAGE 4: Build (after lint and security checks pass)
  # ============================================================
  build:
    name: Build
    runs-on: ubuntu-latest
    needs: [lint, sast, dependency-scan]
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      # For monorepos using Turborepo -- only build affected packages
      - name: Build (incremental)
        run: npx turbo run build --filter=[HEAD^1]

      - name: Upload build artifacts
        uses: actions/upload-artifact@v4
        with:
          name: dist-${{ github.sha }}
          path: |
            packages/*/dist/
            apps/*/dist/
            apps/*/.next/
          retention-days: 7

  # ============================================================
  # STAGE 5: Container Build & Security Scan
  # ============================================================
  container-build:
    name: Container Build & Scan
    runs-on: ubuntu-latest
    needs: [build, test, integration-test]
    timeout-minutes: 15
    permissions:
      contents: read
      security-events: write
      packages: write
    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          load: true
          tags: myapp:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:${{ github.sha }}
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'

      - name: Upload Trivy results to GitHub Security
        if: always()
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: 'trivy-results.sarif'

      - name: Generate SBOM
        uses: anchore/sbom-action@v0
        with:
          image: myapp:${{ github.sha }}
          format: spdx-json
          output-file: sbom.spdx.json

      - name: Upload SBOM
        uses: actions/upload-artifact@v4
        with:
          name: sbom
          path: sbom.spdx.json

  # ============================================================
  # STAGE 6: E2E Tests (main branch only)
  # ============================================================
  e2e:
    name: E2E Tests
    runs-on: ubuntu-latest
    needs: [container-build]
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
          node-version: 20
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright browsers
        run: npx playwright install --with-deps

      - name: Download build artifacts
        uses: actions/download-artifact@v4
        with:
          name: dist-${{ github.sha }}

      - name: Run E2E tests (sharded)
        run: npx playwright test --shard=${{ matrix.shardIndex }}/${{ matrix.shardTotal }}

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: playwright-report-${{ matrix.shardIndex }}
          path: playwright-report/

  # ============================================================
  # STAGE 7: Security Gate
  # ============================================================
  security-gate:
    name: Security Gate
    runs-on: ubuntu-latest
    needs: [sast, dependency-scan, container-build]
    if: always()
    timeout-minutes: 5
    steps:
      - name: Evaluate Security Posture
        run: |
          echo "## Security Scan Summary" >> $GITHUB_STEP_SUMMARY

          if [ "${{ needs.sast.result }}" == "failure" ] || \
             [ "${{ needs.container-build.result }}" == "failure" ]; then
            echo "Security gate FAILED - Critical vulnerabilities found" >> $GITHUB_STEP_SUMMARY
            exit 1
          else
            echo "Security gate PASSED - No critical issues detected" >> $GITHUB_STEP_SUMMARY
          fi

  # ============================================================
  # STAGE 8: Deployment
  # ============================================================
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: [container-build, test, integration-test, security-gate]
    if: github.ref == 'refs/heads/develop'
    environment:
      name: staging
      url: https://staging.example.com
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
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: us-east-1

      - name: Deploy
        run: |
          # Replace with your actual deployment command
          aws s3 sync dist/ s3://${{ secrets.STAGING_BUCKET }}

      - name: Smoke tests
        run: |
          sleep 10
          curl -f https://staging.example.com/health || exit 1

  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: [e2e, security-gate]
    if: github.ref == 'refs/heads/main'
    environment:
      name: production
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
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: us-east-1

      - name: Deploy
        run: |
          # Replace with your actual deployment command
          aws s3 sync dist/ s3://${{ secrets.PRODUCTION_BUCKET }}

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
```

---

## Step 4: Key Optimizations to Cut Your 25-Minute Build

Here is a breakdown of the specific optimizations that will bring your pipeline down from 25 minutes, along with estimated time savings:

### Quick Wins (immediate, high impact)

| Optimization | Expected Savings | Details |
|---|---|---|
| **Dependency caching** | 2-5 min | Using `actions/setup-node@v4` with `cache: 'npm'` avoids re-downloading node_modules every run. This alone can save 50-90% of install time. |
| **`npm ci` instead of `npm install`** | 30-60 sec | `npm ci` is faster and deterministic; it skips the resolution step and installs directly from the lockfile. |
| **Concurrency cancellation** | Eliminates wasted runs | The `concurrency` block with `cancel-in-progress: true` kills old runs when a new push arrives, freeing runners immediately. |
| **Path filters** | Skips unnecessary runs | The `paths-ignore` on `**.md` and `docs/**` means documentation-only changes do not trigger the full pipeline at all. |
| **Job timeouts** | Prevents hung builds | Every job has an explicit `timeout-minutes` so a stuck process cannot burn runner minutes for hours. |

### Medium Impact (architecture changes)

| Optimization | Expected Savings | Details |
|---|---|---|
| **Test sharding (4 shards)** | 50-75% of test time | Splitting unit tests across 4 parallel shards means a 12-minute test suite runs in ~3 minutes wall-clock time. Uses Jest's built-in `--shard` flag. |
| **E2E sharding (3 shards)** | 50-66% of E2E time | Playwright tests split across 3 shards with `--shard` flag. |
| **Parallel security scans** | ~5 min saved | SAST, SCA, and lint run in parallel after secret-scan, not sequentially. |
| **Monorepo-aware builds with Turborepo** | 30-70% of build time | `npx turbo run build --filter=[HEAD^1]` only rebuilds packages that changed since the last commit. |
| **E2E only on main branch** | Saves 10-15 min on PRs | E2E tests are slow; run them only when merging to main, not on every PR push. |

### Advanced (for further gains)

| Optimization | Expected Savings | Details |
|---|---|---|
| **Turborepo remote caching** | 30-50% of build time | Enable Turborepo remote cache (Vercel or self-hosted) so build results are shared across CI runs and developer machines. |
| **Docker layer caching** | 2-5 min | `cache-from: type=gha` and `cache-to: type=gha,mode=max` reuses Docker layers from previous builds. |
| **TypeScript incremental builds** | 1-2 min | Add `"incremental": true` to `tsconfig.json` and cache the `.tsbuildinfo` file. |
| **Selective test execution** | Variable | Use `jest --findRelatedTests` with the list of changed files to skip tests for unaffected code on PRs. |

### Expected Timeline After Optimization

| Scenario | Before | After |
|---|---|---|
| PR (no E2E) | ~25 min | ~8-12 min |
| Main branch (full pipeline) | ~25 min | ~15-18 min |
| Docs-only change | ~25 min | Skipped entirely |

---

## Step 5: Security Scanning Architecture

The pipeline integrates a full DevSecOps security stack following the "shift-left" principle -- security checks run early and in parallel so they do not add to your critical path:

### Security Scans Included

| Scan Type | Tool | When | Impact on Build Time |
|---|---|---|---|
| **Secret Scanning** | TruffleHog + Gitleaks | Every commit, first gate | ~1 min (blocks everything) |
| **SAST** | CodeQL + Semgrep | Every commit, parallel | ~5-15 min (runs parallel with other jobs) |
| **SCA** | npm audit + Dependency Review | Every commit, parallel | ~2-5 min (runs parallel) |
| **Container Scanning** | Trivy | After Docker build | ~5 min (after build) |
| **SBOM Generation** | Syft | After Docker build | ~1 min (in container-build job) |
| **DAST** | (recommended separately) | Scheduled, weekly | Not in PR pipeline -- see below |

### DAST Recommendation

DAST scans (OWASP ZAP) are too slow for PR pipelines (15-60 minutes). Set them up as a separate scheduled workflow:

```yaml
# .github/workflows/dast-scan.yml
name: DAST Security Scan

on:
  schedule:
    - cron: '0 2 * * 1'  # Weekly, Monday 2 AM
  workflow_dispatch:

jobs:
  dast:
    runs-on: ubuntu-latest
    timeout-minutes: 60
    steps:
      - name: OWASP ZAP Baseline Scan
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
          retention-days: 30
```

### Security Gate Pattern

The `security-gate` job acts as a quality gate that evaluates results from all security scans. If SAST or container scanning fails, the pipeline blocks deployment. This prevents vulnerable code from reaching production while still allowing developers to see all other results (tests, lint, etc.).

---

## Step 6: Monorepo-Specific Configuration

### Turborepo Setup

If you are not already using a monorepo build tool, add Turborepo:

```bash
npm install turbo --save-dev
```

Create `turbo.json` in your repository root:

```json
{
  "$schema": "https://turbo.build/schema.json",
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**", ".next/**"]
    },
    "test": {
      "dependsOn": ["build"]
    },
    "lint": {},
    "test:unit": {},
    "test:integration": {
      "dependsOn": ["build"]
    }
  }
}
```

### Path-Based Triggers for Individual Packages

For larger monorepos, consider separate workflows per package that only trigger on relevant path changes:

```yaml
# .github/workflows/api-ci.yml
on:
  push:
    paths:
      - 'packages/api/**'
      - 'packages/shared/**'
      - 'package-lock.json'
```

### Alternative: Nx

If you prefer Nx over Turborepo:

```yaml
- name: Build affected packages
  run: npx nx affected --target=build --base=origin/main

- name: Test affected packages
  run: npx nx affected --target=test --base=origin/main
```

---

## Step 7: Additional Configurations

### Dependabot for Automated Dependency Updates

Create `.github/dependabot.yml`:

```yaml
version: 2
updates:
  - package-ecosystem: npm
    directory: "/"
    schedule:
      interval: weekly
    open-pull-requests-limit: 5
    groups:
      dev-dependencies:
        dependency-type: development
      production-dependencies:
        dependency-type: production

  - package-ecosystem: github-actions
    directory: "/"
    schedule:
      interval: weekly
```

### Branch Protection Rules

Configure in GitHub Settings -> Branches -> Branch protection rules for `main`:

- Require pull request reviews (at least 1 reviewer)
- Require status checks to pass before merging:
  - `lint`
  - `test`
  - `integration-test`
  - `sast`
  - `dependency-scan`
  - `security-gate`
- Require branches to be up to date before merging
- Restrict who can push to matching branches

### OIDC Authentication for AWS

Instead of storing long-lived AWS credentials as GitHub Secrets, set up OIDC:

1. Create an IAM OIDC identity provider in AWS for `token.actions.githubusercontent.com`
2. Create an IAM role with a trust policy scoped to your repository:

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
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
        "token.actions.githubusercontent.com:sub": "repo:your-org/your-repo:ref:refs/heads/main"
      }
    }
  }]
}
```

3. Store the role ARN as `AWS_ROLE_ARN` in GitHub Secrets
4. The workflow already uses `id-token: write` permission and `aws-actions/configure-aws-credentials@v4`

---

## Step 8: Monitoring and Continuous Improvement

### Track These Metrics

After implementing the pipeline, monitor:

- **Build duration**: Target under 12 minutes for PRs, under 18 minutes for main
- **Cache hit rate**: Target above 80% (check in GitHub Actions logs)
- **Failure rate**: Target under 5%
- **Flaky test rate**: Identify and fix tests that intermittently fail

### CI Health Check

Periodically run the CI health checker to identify failure patterns:

```bash
python3 scripts/ci_health.py --platform github --repo your-org/your-repo --limit 20
```

### Monthly Review Checklist

- Review build duration trends -- are they creeping up?
- Analyze failure patterns -- are the same tests failing?
- Update dependencies (Dependabot PRs)
- Review security scan results in the GitHub Security tab
- Check if new packages need to be added to the Turborepo pipeline

---

## Summary of Changes

1. **Pipeline structure**: Fast-fail architecture with secret scan as the first gate, then lint/SAST/SCA in parallel, then sharded tests + build in parallel, then E2E (main only) and deployment.

2. **Build optimization**: Dependency caching via `actions/setup-node` with `cache: 'npm'`, Turborepo incremental builds with `--filter=[HEAD^1]`, test sharding across 4 parallel runners, E2E sharding across 3 runners, path filters to skip unnecessary runs, and concurrency cancellation.

3. **Security scanning**: Complete DevSecOps pipeline with secret scanning (TruffleHog + Gitleaks), SAST (CodeQL + Semgrep), SCA (npm audit + Dependency Review), container scanning (Trivy), SBOM generation (Syft), and a security gate that blocks deployment on critical findings. DAST (OWASP ZAP) recommended as a separate weekly scheduled scan.

4. **Deployment**: Multi-environment setup (staging on develop, production on main) with OIDC authentication to AWS, health checks, and GitHub environment protection rules requiring manual approval for production.

5. **Expected result**: PR pipeline drops from 25 minutes to approximately 8-12 minutes. Full main-branch pipeline with E2E completes in approximately 15-18 minutes.
