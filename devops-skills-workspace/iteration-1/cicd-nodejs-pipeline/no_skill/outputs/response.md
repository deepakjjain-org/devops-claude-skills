# CI/CD Pipeline for Node.js Monorepo on GitHub Actions

## Overview

Below is a complete CI/CD pipeline designed for a Node.js monorepo on GitHub Actions. It addresses your 25-minute build time with aggressive caching, parallelization, and change-detection strategies, while integrating security scanning throughout the pipeline.

---

## 1. Build Optimization Strategy (25 min --> ~8-10 min target)

The main levers for reducing a 25-minute monorepo build:

| Technique | Typical Savings | How |
|---|---|---|
| **Dependency caching** | 2-5 min | Cache `node_modules` and package manager store |
| **Affected-only builds** | 5-10 min | Only build/test packages that changed (using Nx, Turborepo, or custom change detection) |
| **Parallel jobs** | 3-5 min | Split lint, test, build, security into parallel matrix jobs |
| **Larger runners** | 1-3 min | Use `ubuntu-latest-xl` or `large` runners for CPU-bound tasks |
| **Incremental TypeScript** | 1-2 min | Use `tsBuildInfoFile` and cache `.tsbuildinfo` |

---

## 2. Complete Pipeline Configuration

### `.github/workflows/ci.yml` -- Main CI Pipeline

```yaml
name: CI Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

# Cancel redundant runs on the same branch
concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

env:
  NODE_VERSION: '20'
  # Set to your package manager: npm, yarn, or pnpm
  PACKAGE_MANAGER: 'pnpm'
  PNPM_VERSION: '9'

permissions:
  contents: read
  security-events: write
  pull-requests: write

jobs:
  # ============================================================
  # Phase 1: Setup & Change Detection
  # ============================================================
  detect-changes:
    name: Detect Changed Packages
    runs-on: ubuntu-latest
    outputs:
      packages: ${{ steps.changes.outputs.packages }}
      has_changes: ${{ steps.changes.outputs.has_changes }}
      all_packages: ${{ steps.all-packages.outputs.list }}
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Detect affected packages
        id: changes
        run: |
          if [ "${{ github.event_name }}" = "push" ] && [ "${{ github.ref }}" = "refs/heads/main" ]; then
            # On main, build everything
            echo "has_changes=true" >> "$GITHUB_OUTPUT"
            echo "packages=$(ls -d packages/*/package.json 2>/dev/null | xargs -I {} dirname {} | xargs -I {} basename {} | jq -R -s -c 'split("\n") | map(select(. != ""))')" >> "$GITHUB_OUTPUT"
          else
            BASE_SHA="${{ github.event.pull_request.base.sha || github.event.before }}"
            HEAD_SHA="${{ github.sha }}"

            # Find which top-level packages have changes
            CHANGED_PACKAGES=$(git diff --name-only "$BASE_SHA" "$HEAD_SHA" | \
              grep '^packages/' | \
              cut -d'/' -f2 | \
              sort -u | \
              jq -R -s -c 'split("\n") | map(select(. != ""))')

            if [ "$CHANGED_PACKAGES" = "[]" ] || [ -z "$CHANGED_PACKAGES" ]; then
              echo "has_changes=false" >> "$GITHUB_OUTPUT"
              echo "packages=[]" >> "$GITHUB_OUTPUT"
            else
              echo "has_changes=true" >> "$GITHUB_OUTPUT"
              echo "packages=$CHANGED_PACKAGES" >> "$GITHUB_OUTPUT"
            fi
          fi

      - name: List all packages
        id: all-packages
        run: |
          echo "list=$(ls -d packages/*/package.json 2>/dev/null | xargs -I {} dirname {} | xargs -I {} basename {} | jq -R -s -c 'split("\n") | map(select(. != ""))')" >> "$GITHUB_OUTPUT"

  # ============================================================
  # Phase 2: Install Dependencies (shared across jobs via cache)
  # ============================================================
  install:
    name: Install Dependencies
    runs-on: ubuntu-latest
    needs: detect-changes
    if: needs.detect-changes.outputs.has_changes == 'true'
    steps:
      - uses: actions/checkout@v4

      - uses: pnpm/action-setup@v4
        with:
          version: ${{ env.PNPM_VERSION }}

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'pnpm'

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      # Cache the entire node_modules for subsequent jobs
      - name: Cache node_modules
        uses: actions/cache/save@v4
        with:
          path: |
            node_modules
            packages/*/node_modules
          key: node-modules-${{ runner.os }}-${{ hashFiles('pnpm-lock.yaml') }}

  # ============================================================
  # Phase 3: Parallel Quality Gates
  # ============================================================

  # --- Lint ---
  lint:
    name: Lint
    runs-on: ubuntu-latest
    needs: [install, detect-changes]
    steps:
      - uses: actions/checkout@v4

      - uses: pnpm/action-setup@v4
        with:
          version: ${{ env.PNPM_VERSION }}

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}

      - name: Restore node_modules
        uses: actions/cache/restore@v4
        with:
          path: |
            node_modules
            packages/*/node_modules
          key: node-modules-${{ runner.os }}-${{ hashFiles('pnpm-lock.yaml') }}

      - name: Lint affected packages
        run: pnpm --filter "./packages/{${{ join(fromJSON(needs.detect-changes.outputs.packages), ',') }}}" lint

  # --- Type Check ---
  typecheck:
    name: Type Check
    runs-on: ubuntu-latest
    needs: [install, detect-changes]
    steps:
      - uses: actions/checkout@v4

      - uses: pnpm/action-setup@v4
        with:
          version: ${{ env.PNPM_VERSION }}

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}

      - name: Restore node_modules
        uses: actions/cache/restore@v4
        with:
          path: |
            node_modules
            packages/*/node_modules
          key: node-modules-${{ runner.os }}-${{ hashFiles('pnpm-lock.yaml') }}

      # Cache TypeScript build info for incremental compilation
      - name: Cache TypeScript build info
        uses: actions/cache@v4
        with:
          path: |
            packages/*/.tsbuildinfo
            packages/*/tsconfig.tsbuildinfo
          key: ts-buildinfo-${{ runner.os }}-${{ github.sha }}
          restore-keys: |
            ts-buildinfo-${{ runner.os }}-

      - name: Type check
        run: pnpm --filter "./packages/{${{ join(fromJSON(needs.detect-changes.outputs.packages), ',') }}}" typecheck

  # --- Unit Tests ---
  test:
    name: Test (${{ matrix.package }})
    runs-on: ubuntu-latest
    needs: [install, detect-changes]
    strategy:
      fail-fast: false
      matrix:
        package: ${{ fromJSON(needs.detect-changes.outputs.packages) }}
    steps:
      - uses: actions/checkout@v4

      - uses: pnpm/action-setup@v4
        with:
          version: ${{ env.PNPM_VERSION }}

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}

      - name: Restore node_modules
        uses: actions/cache/restore@v4
        with:
          path: |
            node_modules
            packages/*/node_modules
          key: node-modules-${{ runner.os }}-${{ hashFiles('pnpm-lock.yaml') }}

      - name: Run tests for ${{ matrix.package }}
        run: pnpm --filter "${{ matrix.package }}" test -- --coverage

      - name: Upload coverage
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: coverage-${{ matrix.package }}
          path: packages/${{ matrix.package }}/coverage/
          retention-days: 7

  # --- Build ---
  build:
    name: Build (${{ matrix.package }})
    runs-on: ubuntu-latest
    needs: [install, detect-changes]
    strategy:
      fail-fast: false
      matrix:
        package: ${{ fromJSON(needs.detect-changes.outputs.packages) }}
    steps:
      - uses: actions/checkout@v4

      - uses: pnpm/action-setup@v4
        with:
          version: ${{ env.PNPM_VERSION }}

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}

      - name: Restore node_modules
        uses: actions/cache/restore@v4
        with:
          path: |
            node_modules
            packages/*/node_modules
          key: node-modules-${{ runner.os }}-${{ hashFiles('pnpm-lock.yaml') }}

      - name: Build ${{ matrix.package }}
        run: pnpm --filter "${{ matrix.package }}" build

      - name: Upload build artifacts
        uses: actions/upload-artifact@v4
        with:
          name: build-${{ matrix.package }}
          path: packages/${{ matrix.package }}/dist/
          retention-days: 1

  # ============================================================
  # Phase 4: Security Scanning (runs in parallel with tests)
  # ============================================================

  # --- Dependency Vulnerability Scanning ---
  dependency-scan:
    name: Dependency Audit
    runs-on: ubuntu-latest
    needs: install
    steps:
      - uses: actions/checkout@v4

      - uses: pnpm/action-setup@v4
        with:
          version: ${{ env.PNPM_VERSION }}

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}

      - name: Restore node_modules
        uses: actions/cache/restore@v4
        with:
          path: |
            node_modules
            packages/*/node_modules
          key: node-modules-${{ runner.os }}-${{ hashFiles('pnpm-lock.yaml') }}

      # pnpm audit for known vulnerabilities
      - name: Audit dependencies
        run: pnpm audit --audit-level=high
        continue-on-error: true

      # Trivy for deeper SCA scanning
      - name: Run Trivy vulnerability scanner (filesystem)
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          format: 'sarif'
          output: 'trivy-fs-results.sarif'
          severity: 'CRITICAL,HIGH'

      - name: Upload Trivy scan results to GitHub Security
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: 'trivy-fs-results.sarif'

  # --- Static Application Security Testing (SAST) ---
  sast:
    name: SAST Scan
    runs-on: ubuntu-latest
    needs: install
    steps:
      - uses: actions/checkout@v4

      - name: Initialize CodeQL
        uses: github/codeql-action/init@v3
        with:
          languages: javascript-typescript
          queries: security-extended

      - name: Perform CodeQL analysis
        uses: github/codeql-action/analyze@v3
        with:
          category: '/language:javascript-typescript'

  # --- Secret Scanning ---
  secret-scan:
    name: Secret Scanning
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Scan for secrets with Gitleaks
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  # --- License Compliance ---
  license-check:
    name: License Compliance
    runs-on: ubuntu-latest
    needs: install
    steps:
      - uses: actions/checkout@v4

      - uses: pnpm/action-setup@v4
        with:
          version: ${{ env.PNPM_VERSION }}

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}

      - name: Restore node_modules
        uses: actions/cache/restore@v4
        with:
          path: |
            node_modules
            packages/*/node_modules
          key: node-modules-${{ runner.os }}-${{ hashFiles('pnpm-lock.yaml') }}

      - name: Check licenses
        run: |
          npx license-checker --production --failOn "GPL-3.0;AGPL-3.0" --summary

  # ============================================================
  # Phase 5: Integration / E2E Tests (after build)
  # ============================================================
  e2e:
    name: E2E Tests
    runs-on: ubuntu-latest
    needs: [build, detect-changes]
    if: needs.detect-changes.outputs.has_changes == 'true'
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: testdb
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      redis:
        image: redis:7
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
      - uses: actions/checkout@v4

      - uses: pnpm/action-setup@v4
        with:
          version: ${{ env.PNPM_VERSION }}

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}

      - name: Restore node_modules
        uses: actions/cache/restore@v4
        with:
          path: |
            node_modules
            packages/*/node_modules
          key: node-modules-${{ runner.os }}-${{ hashFiles('pnpm-lock.yaml') }}

      - name: Download all build artifacts
        uses: actions/download-artifact@v4
        with:
          pattern: build-*
          merge-multiple: false

      - name: Run E2E tests
        run: pnpm run e2e
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/testdb
          REDIS_URL: redis://localhost:6379
          NODE_ENV: test

  # ============================================================
  # Phase 6: Quality Gate (all checks must pass)
  # ============================================================
  quality-gate:
    name: Quality Gate
    runs-on: ubuntu-latest
    if: always()
    needs:
      - lint
      - typecheck
      - test
      - build
      - dependency-scan
      - sast
      - secret-scan
      - license-check
    steps:
      - name: Check all job statuses
        run: |
          echo "Lint: ${{ needs.lint.result }}"
          echo "Typecheck: ${{ needs.typecheck.result }}"
          echo "Test: ${{ needs.test.result }}"
          echo "Build: ${{ needs.build.result }}"
          echo "Dependency Scan: ${{ needs.dependency-scan.result }}"
          echo "SAST: ${{ needs.sast.result }}"
          echo "Secret Scan: ${{ needs.secret-scan.result }}"
          echo "License Check: ${{ needs.license-check.result }}"

          # Fail if any required job failed
          if [[ "${{ needs.lint.result }}" == "failure" ]] || \
             [[ "${{ needs.typecheck.result }}" == "failure" ]] || \
             [[ "${{ needs.test.result }}" == "failure" ]] || \
             [[ "${{ needs.build.result }}" == "failure" ]] || \
             [[ "${{ needs.secret-scan.result }}" == "failure" ]]; then
            echo "::error::One or more required jobs failed"
            exit 1
          fi

  # ============================================================
  # Phase 7: Deploy (only on main branch push)
  # ============================================================
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: [quality-gate, e2e]
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    environment:
      name: staging
      url: https://staging.example.com
    steps:
      - uses: actions/checkout@v4

      - name: Download build artifacts
        uses: actions/download-artifact@v4
        with:
          pattern: build-*
          merge-multiple: false

      - name: Deploy to staging
        run: |
          echo "Deploy to staging environment"
          # Replace with your actual deployment commands:
          # - AWS: aws ecs update-service / aws lambda update-function-code
          # - Docker: docker build + push to ECR, then trigger ECS/K8s rollout
          # - Serverless: npx serverless deploy --stage staging
          # - Vercel/Netlify: triggered automatically or via CLI

  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: deploy-staging
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    environment:
      name: production
      url: https://app.example.com
    steps:
      - uses: actions/checkout@v4

      - name: Download build artifacts
        uses: actions/download-artifact@v4
        with:
          pattern: build-*
          merge-multiple: false

      - name: Deploy to production
        run: |
          echo "Deploy to production environment"
          # Same deployment mechanism as staging, targeting production
```

---

### `.github/workflows/container-scan.yml` -- Container Image Security (if you build Docker images)

```yaml
name: Container Security Scan

on:
  push:
    branches: [main]
    paths:
      - '**/Dockerfile'
      - '**/docker-compose*.yml'
  pull_request:
    paths:
      - '**/Dockerfile'
      - '**/docker-compose*.yml'

permissions:
  contents: read
  security-events: write

jobs:
  scan-dockerfiles:
    name: Scan Dockerfiles
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run Hadolint (Dockerfile linter)
        uses: hadolint/hadolint-action@v3.1.0
        with:
          recursive: true
          format: sarif
          output-file: hadolint-results.sarif

      - name: Upload Hadolint results
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: hadolint-results.sarif

  build-and-scan-images:
    name: Build & Scan Images
    runs-on: ubuntu-latest
    strategy:
      matrix:
        package: [api, web, worker]  # Adjust to your actual services
    steps:
      - uses: actions/checkout@v4

      - name: Build image
        run: |
          docker build \
            -t ${{ matrix.package }}:scan \
            -f packages/${{ matrix.package }}/Dockerfile \
            .

      - name: Run Trivy on image
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: '${{ matrix.package }}:scan'
          format: 'sarif'
          output: 'trivy-image-${{ matrix.package }}.sarif'
          severity: 'CRITICAL,HIGH'

      - name: Upload Trivy results
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: 'trivy-image-${{ matrix.package }}.sarif'
```

---

## 3. Pipeline Architecture Diagram

```
PR / Push
  |
  v
[detect-changes] -----> determines affected packages
  |
  v
[install] -----> frozen lockfile + cache save
  |
  +----> [lint]              \
  +----> [typecheck]          |--- All run in parallel
  +----> [test (matrix)]      |    (one job per affected package for tests)
  +----> [build (matrix)]     |
  +----> [dependency-scan]    |
  +----> [sast / CodeQL]      |
  +----> [secret-scan]        |
  +----> [license-check]     /
  |
  v
[quality-gate] -----> aggregates all results
  |
  v
[e2e] -----> integration tests with service containers
  |
  v (main branch only)
[deploy-staging] --> [deploy-production]
```

---

## 4. Key Speed Optimizations Explained

### 4.1 Change Detection
The `detect-changes` job uses `git diff` to identify which packages under `packages/` actually changed. Only those packages go through lint, test, and build. For a monorepo with 10 packages where only 2 changed, this cuts work by ~80%.

**If you use Nx or Turborepo**, replace the custom change detection with their built-in affected commands:

```yaml
# Nx alternative
- run: npx nx affected --target=build --base=origin/main --head=HEAD

# Turborepo alternative
- run: npx turbo run build --filter="...[origin/main]"
```

### 4.2 Dependency Caching
The pipeline uses a two-layer cache strategy:
1. **pnpm store cache** (via `actions/setup-node` with `cache: 'pnpm'`) -- caches the global package store
2. **node_modules cache** (via `actions/cache`) -- caches the installed node_modules so subsequent jobs skip `pnpm install` entirely

### 4.3 Matrix Strategy for Tests
Each affected package gets its own parallel test job via `strategy.matrix`. If you have 5 packages, you get 5 parallel test runners instead of sequential execution.

### 4.4 Concurrency Control
The `concurrency` block at the top cancels in-progress runs for the same branch when a new push arrives. This prevents wasted minutes on outdated commits.

---

## 5. Security Scanning Summary

| Layer | Tool | What It Catches | When |
|---|---|---|---|
| **Dependencies (SCA)** | `pnpm audit` + Trivy | Known CVEs in npm packages | Every PR |
| **Static Analysis (SAST)** | CodeQL | SQL injection, XSS, insecure patterns | Every PR |
| **Secrets** | Gitleaks | API keys, tokens, passwords in code | Every PR |
| **Licenses** | license-checker | GPL/AGPL contamination | Every PR |
| **Container Images** | Trivy + Hadolint | OS-level CVEs, Dockerfile misconfigs | Dockerfile changes |

All SARIF results are uploaded to GitHub's Security tab, giving you a unified vulnerability dashboard.

---

## 6. Additional Configuration Files

### `.github/dependabot.yml` -- Automated Dependency Updates

```yaml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
    open-pull-requests-limit: 10
    groups:
      # Group minor/patch updates to reduce PR noise
      production-dependencies:
        patterns:
          - "*"
        update-types:
          - "minor"
          - "patch"
    labels:
      - "dependencies"
    reviewers:
      - "your-team"

  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
    labels:
      - "ci"
```

### `.github/CODEOWNERS` -- Require Reviews for CI Changes

```
# CI/CD pipeline changes require platform team review
.github/workflows/ @your-org/platform-team
Dockerfile @your-org/platform-team
docker-compose*.yml @your-org/platform-team
```

### `.gitleaksignore` -- False Positive Suppression (if needed)

```
# Add fingerprints of known false positives from gitleaks
# Example: a test fixture that looks like a key but isn't
# abc123def456...
```

---

## 7. Branch Protection Rules

Configure these in GitHub Settings > Branches > Branch protection rules for `main`:

- **Require status checks to pass**: Select `quality-gate` and `e2e`
- **Require branches to be up to date before merging**: Yes
- **Require pull request reviews**: At least 1 reviewer
- **Require conversation resolution**: Yes
- **Require signed commits**: Recommended
- **Do not allow bypassing the above settings**: Yes

---

## 8. Monitoring Pipeline Performance

Add a workflow to track pipeline metrics over time:

```yaml
# In your ci.yml, add to the quality-gate job:
      - name: Report pipeline duration
        if: always()
        run: |
          echo "## Pipeline Summary" >> "$GITHUB_STEP_SUMMARY"
          echo "| Job | Status |" >> "$GITHUB_STEP_SUMMARY"
          echo "|---|---|" >> "$GITHUB_STEP_SUMMARY"
          echo "| Lint | ${{ needs.lint.result }} |" >> "$GITHUB_STEP_SUMMARY"
          echo "| Typecheck | ${{ needs.typecheck.result }} |" >> "$GITHUB_STEP_SUMMARY"
          echo "| Tests | ${{ needs.test.result }} |" >> "$GITHUB_STEP_SUMMARY"
          echo "| Build | ${{ needs.build.result }} |" >> "$GITHUB_STEP_SUMMARY"
          echo "| Security | ${{ needs.sast.result }} |" >> "$GITHUB_STEP_SUMMARY"
```

---

## 9. Immediate Quick Wins Checklist

If you want to start with the highest-impact changes before adopting the full pipeline above:

1. **Add `concurrency` with `cancel-in-progress: true`** -- stops wasting minutes on stale runs
2. **Cache dependencies** -- typically saves 2-5 minutes per run
3. **Add change detection** -- biggest single win for monorepos
4. **Split test/lint/build into parallel jobs** -- exploit GitHub's free parallelism
5. **Add `pnpm audit`** -- zero-config security win
6. **Add Gitleaks** -- prevents credential leaks with minimal setup

These six changes alone should bring your 25-minute build down to 8-12 minutes while adding security coverage you likely do not have today.

---

## 10. Cost Considerations

- **GitHub Actions free tier**: 2,000 minutes/month (private repos), unlimited for public repos
- **Matrix jobs multiply minutes**: 5 parallel test jobs x 3 min each = 15 runner-minutes (but only 3 wall-clock minutes)
- **Larger runners**: 2x or 4x cost but proportional speed gains for CPU-bound builds
- **Caching**: Free, saves significant time -- always enable it
- **CodeQL**: Free for public repos, included in GitHub Advanced Security for private repos

If cost is a concern, keep matrix parallelism low and rely more on change detection to skip unnecessary work.
