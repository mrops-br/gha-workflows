# GitHub Actions Reusable Workflows

Centralized, tech-stack-specific CI/CD workflows for maximum code reuse and minimal maintenance.

## Design Philosophy

**Problem:** Generic workflows require callers to pass commands as inputs (e.g., `lint-command: 'npm run lint'`), making caller workflows verbose and defeating the purpose of reusability.

**Solution:** Tech-stack-specific workflow templates that encapsulate all language-specific logic. Callers only need to specify their tech stack and basic configuration.

## Architecture

### Two-Layer Approach

```
┌─────────────────────────────────────────────────────────┐
│  Application Repository                                  │
│  .github/workflows/pipeline.yaml (~25 lines)            │
│                                                          │
│  - Calls tech-stack pipeline (e.g., _pipeline-golang)   │
│  - Passes app-name, project, gitops-repo                │
│  - No command strings needed!                           │
└──────────────────┬──────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────┐
│  gha-workflows Repository (This Repo)                    │
│                                                          │
│  Pipeline Orchestrators (Full CI/CD):                   │
│  ├─ _pipeline-golang.yaml                               │
│  ├─ _pipeline-nodejs.yaml                               │
│  └─ _pipeline-python.yaml                               │
│                                                          │
│  CI Templates (Lint + Test + Scan):                     │
│  ├─ _ci-golang.yaml                                     │
│  ├─ _ci-nodejs.yaml                                     │
│  └─ _ci-python.yaml                                     │
│                                                          │
│  Shared Building Blocks:                                │
│  ├─ _build-docker.yaml                                  │
│  └─ _deploy-argocd.yaml                                 │
└─────────────────────────────────────────────────────────┘
```

## Workflow Templates

### Pipeline Orchestrators (Recommended)

These workflows handle the complete CI/CD flow: CI → Build → Deploy

| Workflow | Tech Stack | What It Does |
|----------|-----------|--------------|
| `_pipeline-golang.yaml` | Go | Lint (golangci-lint) → Test → Build → Deploy |
| `_pipeline-nodejs.yaml` | Node.js | Lint (eslint) → Test (jest) → Build → Deploy |
| `_pipeline-python.yaml` | Python | Lint (ruff, black) → Type Check (mypy) → Test (pytest) → Build → Deploy |

**Features:**
- ✅ Automatic environment detection (develop → dev, release/* → stg, main → prd)
- ✅ Git Flow support (develop, release/*, hotfix/*, main)
- ✅ Image tagging strategy built-in
- ✅ Security scanning with Trivy
- ✅ Code coverage reporting
- ✅ ArgoCD deployment

### CI Templates (For Custom Pipelines)

If you need more control, use CI templates directly:

| Workflow | Tech Stack | What It Does |
|----------|-----------|--------------|
| `_ci-golang.yaml` | Go | Lint + Test + Coverage + Security Scan |
| `_ci-nodejs.yaml` | Node.js | Lint + Test + Coverage + Security Scan |
| `_ci-python.yaml` | Python | Lint + Type Check + Test + Coverage + Security Scan |

### Building Blocks (Low-Level)

| Workflow | Purpose |
|----------|---------|
| `_build-docker.yaml` | Build, tag, push, sign, and scan Docker images |
| `_deploy-argocd.yaml` | Update GitOps repo and sync ArgoCD |

## Usage Examples

### Go Application

```yaml
# .github/workflows/pipeline.yaml
name: Pipeline

on:
  push:
    branches: [main, develop, 'release/**', 'hotfix/**']
  pull_request:
    branches: [main, develop]
  workflow_dispatch:
    inputs:
      environment:
        type: choice
        options: [dev, stg, prd]

env:
  APP_NAME: my-go-app
  PROJECT: platform

jobs:
  pipeline:
    uses: mrops-br/gha-workflows/.github/workflows/_pipeline-golang.yaml@main
    with:
      app-name: ${{ env.APP_NAME }}
      project: ${{ env.PROJECT }}
      gitops-repo: mrops-br/gitops-${{ env.PROJECT }}
      go-version: '1.23'
      deploy-environment: ${{ inputs.environment }}
    secrets: inherit
```

**That's it!** ~25 lines vs ~80+ lines with the old approach.

### Node.js Application

```yaml
# .github/workflows/pipeline.yaml
name: Pipeline

on:
  push:
    branches: [main, develop, 'release/**', 'hotfix/**']
  pull_request:
    branches: [main, develop]

env:
  APP_NAME: my-nodejs-app
  PROJECT: devportal

jobs:
  pipeline:
    uses: mrops-br/gha-workflows/.github/workflows/_pipeline-nodejs.yaml@main
    with:
      app-name: ${{ env.APP_NAME }}
      project: ${{ env.PROJECT }}
      gitops-repo: mrops-br/gitops-${{ env.PROJECT }}
      node-version: '20'
      package-manager: 'npm'  # or 'yarn', 'pnpm'
    secrets: inherit
```

### Python Application

```yaml
# .github/workflows/pipeline.yaml
name: Pipeline

on:
  push:
    branches: [main, develop, 'release/**', 'hotfix/**']
  pull_request:
    branches: [main, develop]

env:
  APP_NAME: my-python-app
  PROJECT: data-platform

jobs:
  pipeline:
    uses: mrops-br/gha-workflows/.github/workflows/_pipeline-python.yaml@main
    with:
      app-name: ${{ env.APP_NAME }}
      project: ${{ env.PROJECT }}
      gitops-repo: mrops-br/gitops-${{ env.PROJECT }}
      python-version: '3.12'
      package-manager: 'pip'  # or 'poetry', 'pipenv'
    secrets: inherit
```

## Configuration Options

### Common Inputs (All Pipelines)

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `app-name` | ✅ | - | Application name (used for image naming) |
| `project` | ✅ | - | Project name (e.g., devportal) |
| `gitops-repo` | ✅ | - | GitOps repository (e.g., org/gitops-devportal) |
| `working-directory` | ❌ | `.` | Working directory for commands |
| `dockerfile` | ❌ | `Dockerfile` | Path to Dockerfile |
| `registry` | ❌ | `ghcr.io` | Container registry |
| `enable-lint` | ❌ | `true` | Enable linting |
| `enable-test` | ❌ | `true` | Enable testing |
| `enable-coverage` | ❌ | `true` | Enable coverage reporting |
| `deploy-environment` | ❌ | auto-detect | Override environment (dev, stg, prd) |

### Go-Specific Inputs

| Input | Default | Description |
|-------|---------|-------------|
| `go-version` | `1.23` | Go version to use |

### Node.js-Specific Inputs

| Input | Default | Description |
|-------|---------|-------------|
| `node-version` | `20` | Node.js version to use |
| `package-manager` | `npm` | Package manager (npm, yarn, pnpm) |

### Python-Specific Inputs

| Input | Default | Description |
|-------|---------|-------------|
| `python-version` | `3.12` | Python version to use |
| `package-manager` | `pip` | Package manager (pip, poetry, pipenv) |
| `enable-type-check` | `true` | Enable type checking with mypy |

## Environment Detection (Git Flow)

Pipelines automatically detect the target environment based on the branch:

| Branch Pattern | Environment | Image Tag | Auto-Deploy |
|---------------|-------------|-----------|-------------|
| `develop` | `dev` | `dev-latest` | ✅ Yes |
| `release/*` | `stg` | `X.Y.Z-rc` | ✅ Yes |
| `hotfix/*` | `stg` | `X.Y.Z-hotfix-rc` | ✅ Yes |
| `main` | `prd` | `latest`, `vX.Y.Z` | ❌ Manual only |
| Pull Request | - | `pr-<number>` | ❌ No deploy |

Override with `workflow_dispatch` input:
```yaml
workflow_dispatch:
  inputs:
    environment:
      type: choice
      options: [dev, stg, prd]
```

## Required Secrets

Set these at the **organization level** or per repository:

| Secret | Description |
|--------|-------------|
| `ARGOCD_AUTH_TOKEN` | ArgoCD API authentication token |
| `ARGOCD_SERVER` | ArgoCD server URL (e.g., argocd.yourdomain.com) |
| `GITHUB_TOKEN` | Automatic, no setup needed |

## Image Tagging Strategy

Images are automatically tagged based on context:

```
# Develop branch
ghcr.io/org/app:dev-latest
ghcr.io/org/app:dev-a1b2c3d

# Release branch (release/1.2.0)
ghcr.io/org/app:1.2.0-rc
ghcr.io/org/app:a1b2c3d

# Hotfix branch (hotfix/1.2.1)
ghcr.io/org/app:1.2.1-hotfix-rc
ghcr.io/org/app:a1b2c3d

# Main branch (after release)
ghcr.io/org/app:latest
ghcr.io/org/app:v1.2.0
ghcr.io/org/app:a1b2c3d

# Pull Request
ghcr.io/org/app:pr-123
ghcr.io/org/app:a1b2c3d
```

## Tech Stack Requirements

### Go Projects Must Have:
- `go.mod` and `go.sum`
- `.golangci.yml` (optional, uses defaults)
- Tests in `*_test.go` files
- `Dockerfile`

### Node.js Projects Must Have:
- `package.json` and lock file (`package-lock.json`, `yarn.lock`, or `pnpm-lock.yaml`)
- `npm run lint` script
- `npm test` or `npm run test` script
- `Dockerfile`

### Python Projects Must Have:
- `requirements.txt`, `pyproject.toml` (poetry), or `Pipfile` (pipenv)
- Source code in standard layout
- `pytest` tests (optional but recommended)
- `Dockerfile`

## Advanced: Custom Pipelines

If the orchestrator workflows don't fit your needs, compose your own:

```yaml
jobs:
  ci:
    uses: mrops-br/gha-workflows/.github/workflows/_ci-golang.yaml@main
    with:
      go-version: '1.23'
      enable-lint: true
      enable-test: true

  build:
    needs: ci
    uses: mrops-br/gha-workflows/.github/workflows/_build-docker.yaml@main
    with:
      app-name: my-app
      push: ${{ github.event_name != 'pull_request' }}
    secrets: inherit

  deploy:
    needs: build
    if: github.ref == 'refs/heads/develop'
    uses: mrops-br/gha-workflows/.github/workflows/_deploy-argocd.yaml@main
    with:
      app-name: my-app
      project: my-project
      environment: dev
      gitops-repo: org/gitops-my-project
      image-tag: dev-latest
    secrets: inherit
```

## Migration Guide

### From Old Generic Workflow

**Before (80+ lines):**
```yaml
jobs:
  ci:
    uses: ./_ci.yaml@main
    with:
      runner: '[shared-runners]'
      lint-command: 'npm run lint'
      test-command: 'npm test'
      setup-command: 'npm ci'

  build:
    needs: ci
    uses: ./_build-docker.yaml@main
    with:
      app-name: my-app
      push: true

  deploy-dev:
    needs: build
    if: github.ref == 'refs/heads/develop'
    uses: ./_deploy-argocd.yaml@main
    with:
      app-name: my-app
      environment: dev
      image-tag: dev-latest
```

**After (25 lines):**
```yaml
jobs:
  pipeline:
    uses: mrops-br/gha-workflows/.github/workflows/_pipeline-nodejs.yaml@main
    with:
      app-name: my-app
      project: devportal
      gitops-repo: org/gitops-devportal
    secrets: inherit
```

## Examples

See the `examples/` directory for complete working examples:
- `examples/go-app.yaml` - Go application
- `examples/nodejs-app.yaml` - Node.js application
- `examples/python-app.yaml` - Python application

## Contributing

When adding support for a new tech stack:

1. Create `_ci-<stack>.yaml` with tech-specific CI logic
2. Create `_pipeline-<stack>.yaml` orchestrator
3. Add example in `examples/<stack>-app.yaml`
4. Update this README

## Support

For issues or questions, contact the Platform Engineering team or open an issue in this repository.
