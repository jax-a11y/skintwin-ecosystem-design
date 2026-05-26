# SkinTwin Ecosystem CI/CD Standards

This document defines the centralized CI/CD standards for all repositories in the SkinTwin-AI ecosystem. These standards ensure consistent quality gates, security scanning, and release processes across all components.

## 1. Overview

The SkinTwin ecosystem CI/CD infrastructure is built on **reusable GitHub Actions workflows** that can be called from any repository. This approach ensures:

- **Consistency**: All repositories follow the same quality standards
- **Maintainability**: Updates to CI logic propagate automatically
- **Security**: Centralized security scanning and compliance checks
- **Observability**: Unified reporting and metrics across the ecosystem

## 2. Repository Classification

### 2.1 TypeScript/Node.js Repositories

| Repository | Type | CI Workflow | E2E Required |
|------------|------|-------------|--------------|
| `skintwin-asi` | AI/Cognitive | `ci-node.yml` | Yes (API) |
| `skintwin-customer-portal` | Application | `ci-node.yml` | Yes (Full) |
| `regima-training-lms` | Application | `ci-node.yml` | Yes (Full) |
| `business-directory-template` | Frontend | `ci-node.yml` | Yes (Smoke) |
| `bus-listing` | Frontend | `ci-node.yml` | Yes (Smoke) |

### 2.2 Python Repositories

| Repository | Type | CI Workflow | E2E Required |
|------------|------|-------------|--------------|
| `multiskin` | AI/ML | `ci-python.yml` | Yes (API) |
| `skintwin-integrations` | Integration | `ci-python.yml` | Yes (Full) |
| `org-skin` | DevOps | `ci-python.yml` | No |

## 3. Reusable Workflows

### 3.1 Available Workflows

| Workflow | Purpose | Location |
|----------|---------|----------|
| `ci-node.yml` | Node/TypeScript build, test, lint | `.github/workflows/` |
| `ci-python.yml` | Python build, test, lint | `.github/workflows/` |
| `ci-container.yml` | Container build and security scan | `.github/workflows/` |
| `ci-e2e.yml` | End-to-end test execution | `.github/workflows/` |
| `ci-release.yml` | Release validation and publishing | `.github/workflows/` |
| `ci-contracts.yml` | Cross-repo contract validation | `.github/workflows/` |
| `ci-security.yml` | Security scanning (CodeQL, deps, secrets) | `.github/workflows/` |

### 3.2 Calling Reusable Workflows

Example for a TypeScript repository:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  ci:
    uses: skintwin-ai/skintwin-ecosystem-design/.github/workflows/ci-node.yml@main
    with:
      node-version: "20"
      package-manager: "pnpm"
      coverage-threshold: 80
    secrets:
      CODECOV_TOKEN: ${{ secrets.CODECOV_TOKEN }}

  security:
    uses: skintwin-ai/skintwin-ecosystem-design/.github/workflows/ci-security.yml@main
    with:
      language: typescript
```

Example for a Python repository:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  ci:
    uses: skintwin-ai/skintwin-ecosystem-design/.github/workflows/ci-python.yml@main
    with:
      python-version: "3.11"
      enable-jax: true  # For AI/ML repos
      coverage-threshold: 80
    secrets:
      CODECOV_TOKEN: ${{ secrets.CODECOV_TOKEN }}

  security:
    uses: skintwin-ai/skintwin-ecosystem-design/.github/workflows/ci-security.yml@main
    with:
      language: python
```

## 4. Required Checks

### 4.1 PR Checks (Blocking)

All PRs must pass these checks before merging:

| Check | Node Repos | Python Repos |
|-------|------------|--------------|
| Type checking | ✅ | ✅ (mypy) |
| Linting | ✅ (ESLint) | ✅ (Ruff) |
| Formatting | ✅ (Prettier) | ✅ (Ruff) |
| Unit tests | ✅ | ✅ |
| Coverage threshold | ✅ (80%) | ✅ (80%) |
| Build | ✅ | ✅ |
| Security scan | ✅ | ✅ |
| Smoke E2E | ✅ (if applicable) | ✅ (if applicable) |

### 4.2 Main Branch Checks

Additional checks run on pushes to `main`:

- Full E2E regression suite
- Cross-repo contract validation
- SBOM generation and upload
- Container image build (if applicable)

### 4.3 Nightly Checks

Scheduled runs for comprehensive testing:

- Exhaustive E2E test suite
- Full security audit
- Dependency vulnerability scan
- License compliance check
- Cross-repo compatibility matrix

## 5. Trigger Strategy

### 5.1 Event-Based Triggers

```yaml
on:
  # PR: Fast checks + smoke E2E
  pull_request:
    branches: [main, develop]

  # Push to main: Full build + regression E2E
  push:
    branches: [main]

  # Tags: Release validation
  push:
    tags: ["v*"]

  # Nightly: Exhaustive testing
  schedule:
    - cron: "0 2 * * *"  # 2 AM UTC daily

  # Manual: Targeted reruns
  workflow_dispatch:
    inputs:
      environment:
        description: "Target environment"
        type: choice
        options: [preview, staging, production]
```

### 5.2 Concurrency Controls

All workflows implement concurrency controls to cancel superseded runs:

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

## 6. Caching Strategy

### 6.1 Node.js Caching

```yaml
- uses: actions/setup-node@v4
  with:
    node-version: "20"
    cache: "pnpm"
    cache-dependency-path: pnpm-lock.yaml
```

### 6.2 Python Caching

```yaml
- uses: actions/setup-python@v5
  with:
    python-version: "3.11"
    cache: "pip"
    cache-dependency-path: |
      requirements*.txt
      pyproject.toml
```

### 6.3 Docker Layer Caching

```yaml
- uses: docker/build-push-action@v6
  with:
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

## 7. Artifact Retention

| Artifact Type | Retention | Purpose |
|---------------|-----------|---------|
| Test results | 14 days | Debugging, flake analysis |
| Coverage reports | 14 days | Trend analysis |
| Build artifacts | 7 days | Deployment, debugging |
| SBOM | 90 days | Compliance, auditing |
| Security scans | 30 days | Compliance, remediation |
| E2E screenshots | 14 days | Failure analysis |

## 8. Security Gates

### 8.1 Required Security Checks

| Check | Tool | Blocking |
|-------|------|----------|
| Static analysis | CodeQL | Yes |
| Dependency vulnerabilities | Trivy | Yes (HIGH+) |
| Secret scanning | Gitleaks, TruffleHog | Yes |
| SAST | Semgrep | Yes |
| License compliance | license-checker/pip-licenses | No |
| Container scanning | Trivy | Yes (for images) |

### 8.2 OIDC Authentication

For deployments and cloud resource access, use OIDC-based authentication:

```yaml
permissions:
  id-token: write
  contents: read

steps:
  - name: Configure AWS credentials
    uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::ACCOUNT:role/GitHubActions
      aws-region: us-east-1
```

## 9. Branch Protection

### 9.1 Required Settings

Apply these branch protection rules to `main`:

- ✅ Require pull request before merging
- ✅ Require approvals (minimum: 1)
- ✅ Dismiss stale approvals on new commits
- ✅ Require status checks to pass
- ✅ Require branches to be up to date
- ✅ Require signed commits (recommended)
- ✅ Include administrators
- ✅ Restrict who can push (team leads only)

### 9.2 Required Status Checks

Configure these as required status checks:

- `ci-summary` (from Node or Python CI)
- `Security Summary` (from security workflow)
- `E2E Summary (smoke)` (for applicable repos)

## 10. Workflow Versioning

### 10.1 Version Pinning

Always pin actions to specific versions:

```yaml
# ✅ Good - pinned to major version
uses: actions/checkout@v4

# ✅ Better - pinned to specific SHA
uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11

# ❌ Bad - floating tag
uses: actions/checkout@main
```

### 10.2 Reusable Workflow Versioning

Reference reusable workflows by tag or commit:

```yaml
# For production stability
uses: skintwin-ai/skintwin-ecosystem-design/.github/workflows/ci-node.yml@v1.0.0

# For development/testing
uses: skintwin-ai/skintwin-ecosystem-design/.github/workflows/ci-node.yml@main
```

## 11. Monitoring and Observability

### 11.1 Workflow Summaries

All workflows generate GitHub Step Summaries with:

- Check status (pass/fail)
- Test counts and coverage
- Security findings summary
- Artifact links

### 11.2 Metrics to Track

| Metric | Target | Alert Threshold |
|--------|--------|-----------------|
| CI duration | < 10 min | > 15 min |
| E2E duration | < 20 min | > 30 min |
| Flaky test rate | < 1% | > 5% |
| Coverage | > 80% | < 70% |
| Security vulns (high) | 0 | > 0 |

### 11.3 CODEOWNERS Integration

Route failures to appropriate teams:

```
# .github/CODEOWNERS
* @skintwin-ai/platform-team
/src/ai/ @skintwin-ai/ai-team
/src/integrations/ @skintwin-ai/integrations-team
/.github/ @skintwin-ai/devops-team
```

## 12. Rollout Plan

### Phase 1: Foundation (Week 1-2)
- [ ] Deploy reusable workflows to this repository
- [ ] Add baseline CI to all active repositories
- [ ] Configure branch protection rules

### Phase 2: Smoke E2E (Week 3-4)
- [ ] Implement smoke E2E tests for critical paths
- [ ] Add E2E to PR gating for application repos
- [ ] Set up preview environments

### Phase 3: Contracts (Week 5-6)
- [ ] Define API contracts between services
- [ ] Implement contract validation workflows
- [ ] Add nightly compatibility matrix

### Phase 4: Hardening (Week 7-8)
- [ ] Tune security gates
- [ ] Optimize CI performance
- [ ] Lock branch protections

### Phase 5: Governance (Ongoing)
- [ ] Monthly quality review meetings
- [ ] Flaky test triage and burn-down
- [ ] Compatibility drift monitoring

## 13. Troubleshooting

### Common Issues

1. **Cache miss**: Check `cache-dependency-path` matches your lockfile location
2. **Workflow not triggering**: Verify branch patterns and event types
3. **Permission denied**: Check workflow permissions and OIDC configuration
4. **Timeout**: Increase `timeout-minutes` or optimize test parallelization

### Getting Help

- Open an issue in this repository for CI/CD infrastructure questions
- Tag `@skintwin-ai/devops-team` for urgent issues
- Check the [GitHub Actions documentation](https://docs.github.com/en/actions)
