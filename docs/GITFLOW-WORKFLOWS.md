# Git Flow Workflows Guide

This document explains how to use the Git Flow-aware workflows and composite actions.

## Overview

Our workflows are designed to work seamlessly with Git Flow branching strategy:

- **develop** → Auto-deploy to `dev` environment
- **release/X.Y.Z** → Auto-deploy to `stg` environment
- **hotfix/X.Y.Z** → Auto-deploy to `stg` environment
- **main** → Manual deployment to `prd` environment (+ automated release process)
- **feature/** → Build only, no deployment
- **PR** → Validation only

## Architecture

### Composite Actions (Building Blocks)

Located in `.github/actions/`:

#### Git Flow Actions (`gitflow/`)

- **detect-version** - Extract version from branch names or merge commits
- **generate-tags** - Generate image tags based on Git Flow rules
- **detect-env** - Determine deployment environment from branch type

#### Docker Actions (`docker/`)

- **build-push** - Build and push images with automatic Git Flow tagging
- **sign** - Sign images with cosign (Sigstore)
- **scan** - Scan images for vulnerabilities with Trivy
- **retag** - Retag existing images (for releases)

#### Quality Actions (`quality/`)

- **lint-dockerfile** - Lint Dockerfiles with hadolint

### Reusable Workflows

Located in `.github/workflows/`:

#### Core Functions

- **_gitflow-release.yaml** - Automated release orchestrator
- **_docker-retag.yaml** - Standalone Docker image retagging
- **_build-docker.yaml** - Build and push Docker images
- **_deploy-argocd.yaml** - Deploy to Kubernetes via ArgoCD
- **_ci-generic.yaml** - Generic CI for any repository

#### Pipeline Orchestrators

- **_pipeline-nodejs.yaml** - Complete CI/CD for Node.js apps
- **_pipeline-golang.yaml** - Complete CI/CD for Go apps
- **_pipeline-python.yaml** - Complete CI/CD for Python apps
- **_pipeline-container.yaml** - CI/CD for container-only repos

## Git Flow Tagging Strategy

### Automatic Tags

| Branch | Event | Tags Generated | Example |
|--------|-------|----------------|---------|
| `develop` | Push | `dev-latest`, `dev-<sha>` | `dev-latest`, `dev-a1b2c3d` |
| `release/1.2.0` | Push/PR | `1.2.0-rc`, `<sha>` | `1.2.0-rc`, `a1b2c3d` |
| `hotfix/1.2.1` | Push/PR | `1.2.1-hotfix-rc`, `<sha>` | `1.2.1-hotfix-rc`, `a1b2c3d` |
| `main` | Merge | `1.2.0`, `1.2`, `1`, `latest` | `1.2.0`, `1.2`, `1`, `latest` |
| `feature/*` | PR | `pr-<number>`, `<sha>` | `pr-123`, `a1b2c3d` |

### Environment Mapping

| Branch Type | Environment | Auto-Deploy | Requires Approval |
|-------------|-------------|-------------|-------------------|
| `develop` | `dev` | ✅ Yes | ❌ No |
| `release/*` | `stg` | ✅ Yes | ❌ No |
| `hotfix/*` | `stg` | ✅ Yes | ❌ No |
| `main` | `prd` | ❌ Manual | ✅ Yes |
| `feature/*` | - | ❌ No | - |

## Usage Examples

### Example 1: Container-Only Repository (like gha-runner-images)

Perfect for building and releasing container images without application deployment.

```yaml
# .github/workflows/pipeline.yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop, 'release/**', 'hotfix/**']
  pull_request:
    branches: [main, develop]

jobs:
  pipeline:
    uses: mrops-br/gha-workflows/.github/workflows/_pipeline-container.yaml@main
    permissions:
      contents: write
      packages: write
      id-token: write
      security-events: write
    secrets: inherit
    with:
      image-name: ${{ github.repository_owner }}/my-image
      context: ./images/base
      dockerfile: ./images/base/Dockerfile
      enable-auto-release: true  # Auto-release on merge to main
      enable-changelog: true
      changelog-config: cliff.toml
```

**What happens:**
- **develop**: Builds and pushes `dev-latest`, `dev-<sha>`
- **release/1.2.0 PR to main**: Builds and pushes `1.2.0-rc`
- **main merge**: Automatically retags `1.2.0-rc` → `1.2.0`, `1.2`, `1`, `latest`
- Creates git tag `v1.2.0`
- Generates changelog
- Merges back to develop

### Example 2: Node.js Application with GitOps

Full application pipeline with CI/CD and ArgoCD deployment.

```yaml
# .github/workflows/pipeline.yaml
name: CI/CD Pipeline

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
  APP_NAME: my-nodejs-app
  PROJECT: devportal
  GITOPS_REPO: mrops-br/gitops-devportal

jobs:
  pipeline:
    uses: mrops-br/gha-workflows/.github/workflows/_pipeline-nodejs.yaml@main
    with:
      app-name: ${{ env.APP_NAME }}
      project: ${{ env.PROJECT }}
      gitops-repo: ${{ env.GITOPS_REPO }}
      node-version: '20'
      package-manager: 'npm'
      deploy-environment: ${{ inputs.environment }}
    secrets: inherit
```

**What happens:**
- **develop**: CI → Build → Push → Deploy to `dev`
- **release/1.2.0**: CI → Build → Push → Deploy to `stg`
- **main**: CI → Build → Push → Manual deploy to `prd`
- **PR**: CI → Build (validation only)

### Example 3: Custom Pipeline Using Building Blocks

Compose your own pipeline for specific needs.

```yaml
name: Custom Pipeline

on:
  push:
    branches: [main, develop]

jobs:
  # Detect Git Flow context
  detect-gitflow:
    runs-on: ubuntu-latest
    outputs:
      branch-type: ${{ steps.gitflow.outputs.branch-type }}
      version: ${{ steps.gitflow.outputs.version }}
    steps:
      - uses: actions/checkout@v6
      - id: gitflow
        uses: mrops-br/gha-workflows/.github/actions/gitflow/detect-version@main

  # Custom CI
  ci:
    uses: ./.github/workflows/_ci-generic.yaml
    with:
      enable-lint-dockerfile: true
      dockerfile: Dockerfile
      enable-custom-test: true
      custom-setup-command: 'npm ci'
      custom-test-command: 'npm test'

  # Build Docker image
  build:
    needs: [detect-gitflow, ci]
    uses: mrops-br/gha-workflows/.github/workflows/_build-docker.yaml@main
    with:
      app-name: my-app
      push: ${{ github.ref == 'refs/heads/develop' }}
    secrets: inherit

  # Custom deployment (not ArgoCD)
  deploy:
    needs: [detect-gitflow, build]
    if: needs.detect-gitflow.outputs.branch-type == 'develop'
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to custom platform
        run: |
          echo "Deploying to dev environment..."
          # Your custom deployment logic here
```

### Example 4: Manual Release Workflow

Trigger releases manually without merging to main.

```yaml
name: Manual Release

on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Version to release (e.g., 1.2.3)'
        required: true
      source-tag:
        description: 'Source tag (e.g., 1.2.3-rc)'
        required: true

jobs:
  release:
    uses: mrops-br/gha-workflows/.github/workflows/_gitflow-release.yaml@main
    permissions:
      contents: write
      packages: write
      id-token: write
    secrets: inherit
    with:
      auto-detect-version: false
      version: ${{ inputs.version }}
      source-tag: ${{ inputs.source-tag }}
      enable-docker-retag: true
      docker-registry: ghcr.io
      docker-image-name: ${{ github.repository_owner }}/my-app
      docker-semver-tags: true
      enable-changelog: true
      enable-git-tag: true
      enable-merge-back: true
```

## Composite Actions Usage

### Detect Git Flow Version

```yaml
steps:
  - uses: actions/checkout@v6

  - name: Detect version
    id: version
    uses: mrops-br/gha-workflows/.github/actions/gitflow/detect-version@main

  - name: Use outputs
    run: |
      echo "Branch type: ${{ steps.version.outputs.branch-type }}"
      echo "Version: ${{ steps.version.outputs.version }}"
      echo "Source tag: ${{ steps.version.outputs.source-tag }}"
```

### Generate Git Flow Tags

```yaml
steps:
  - name: Generate tags
    id: tags
    uses: mrops-br/gha-workflows/.github/actions/gitflow/generate-tags@main
    with:
      branch-type: develop
      version: ''
      sha: ${{ github.sha }}
      event-name: ${{ github.event_name }}

  - name: Use tags
    run: echo "Tags: ${{ steps.tags.outputs.tags }}"
```

### Build and Push Docker Image

```yaml
steps:
  - uses: actions/checkout@v6

  - name: Build and push
    id: build
    uses: mrops-br/gha-workflows/.github/actions/docker/build-push@main
    with:
      context: .
      dockerfile: Dockerfile
      registry: ghcr.io
      image-name: ${{ github.repository_owner }}/my-app
      push: true
      registry-username: ${{ github.actor }}
      registry-password: ${{ secrets.GITHUB_TOKEN }}
```

### Retag Docker Image

```yaml
steps:
  - uses: actions/checkout@v6

  - name: Retag for release
    uses: mrops-br/gha-workflows/.github/actions/docker/retag@main
    with:
      registry: ghcr.io
      image-name: org/my-app
      source-tag: 1.2.3-rc
      target-tags: |
        1.2.3
        1.2
        1
        latest
      registry-username: ${{ github.actor }}
      registry-password: ${{ secrets.GITHUB_TOKEN }}
```

## Required Secrets

Configure at organization or repository level:

| Secret | Purpose | Required For |
|--------|---------|--------------|
| `GITHUB_TOKEN` | Automatic (no setup) | Docker registry, GitHub API |
| `ARGOCD_AUTH_TOKEN` | ArgoCD API authentication | ArgoCD deployments |
| `ARGOCD_SERVER` | ArgoCD server URL | ArgoCD deployments |

## Migration from Old Workflows

### Before (Custom Logic in Every Repo)

```yaml
jobs:
  build:
    runs-on: [self-hosted]
    steps:
      - uses: actions/checkout@v6
      - name: Determine tag
        id: tag
        run: |
          if [ "${{ github.ref }}" = "refs/heads/develop" ]; then
            echo "tag=dev-latest" >> $GITHUB_OUTPUT
          elif [[ "${{ github.ref }}" =~ ^refs/heads/release/ ]]; then
            # Complex logic...
          fi
      - name: Build and push
        # More manual steps...
```

### After (Reusable Workflow)

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

**Benefits:**
- ✅ Reduced from ~100 lines to ~15 lines
- ✅ Consistent Git Flow handling across all repos
- ✅ Automatic version detection
- ✅ Built-in security (signing, scanning)
- ✅ Centralized updates

## Troubleshooting

### Version Not Detected

**Problem:** Release workflow says "Could not auto-detect version"

**Solution:** Ensure merge commit message contains the branch name:
- Good: `Merge pull request #123 from org/release/1.2.3`
- Good: `Merge branch 'release/1.2.3' into main`
- Bad: Custom commit message without branch reference

### Image Not Found During Retag

**Problem:** "Error: source image not found"

**Solution:** Ensure the RC image was built and pushed:
1. Check that PR to main from `release/1.2.3` built successfully
2. Verify image exists: `ghcr.io/org/app:1.2.3-rc`
3. Check GHCR permissions

### Deployment Not Triggered

**Problem:** Build succeeds but deployment doesn't run

**Solution:**
1. Check branch type is correct (develop, release/*, hotfix/*)
2. Verify ArgoCD secrets are configured
3. Check if it's a PR (PRs don't deploy by default)
4. Look at `determine-env` job outputs

## Best Practices

1. **Always use Git Flow branches** - The workflows expect standard Git Flow patterns
2. **Version format** - Use semantic versioning: `X.Y.Z` (e.g., `1.2.3`)
3. **RC images** - Always create release branches and build RC images before merging to main
4. **Test in develop** - Thoroughly test in develop before creating release branch
5. **Changelog** - Enable changelog generation for better release notes
6. **Secrets** - Configure secrets at organization level for consistency

## Reference

For complete API reference, see:
- [Composite Actions Reference](./COMPOSITE-ACTIONS.md)
- [Workflow Inputs Reference](./WORKFLOW-INPUTS.md)
- [Examples Directory](../examples/)
