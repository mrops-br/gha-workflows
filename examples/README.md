# Workflow Examples

**7 focused examples** covering all major use cases. Simple, no duplicates.

> **Note**: All examples use a **single workflow file** with both `push` and `pull_request` triggers. No need for separate PR check workflows - the pipelines automatically handle different scenarios based on the Git Flow context.

## Quick Reference

| Example | Language | Use Case |
|---------|----------|----------|
| `golang-microservice.yaml` | Go | Backend services, APIs, microservices |
| `nodejs-microservice.yaml` | Node.js | Web apps, APIs, full-stack |
| `python-ml-service.yaml` | Python | ML services, data processing, APIs |
| `container-repo.yaml` | Any | Build container images only (no app deployment) |
| `simple-app.yaml` | Any | Quick-start template (choose your language) |
| `custom-modular.yaml` | Any | Advanced custom pipelines |
| `cleanup-images.yaml` | Utility | Clean up old container images |

## Examples

### Language-Specific Pipelines

#### 1. Go Microservice (`golang-microservice.yaml`)

**Features:**
- golangci-lint, go test, race detection, coverage
- gosec security scanning
- Docker build, sign, scan
- GitOps deployment
- Automatic releases

**Perfect for:** Backend services, gRPC, CLI tools

#### 2. Node.js Microservice (`nodejs-microservice.yaml`)

**Features:**
- ESLint, Jest, coverage
- Docker build, sign, scan
- GitOps deployment
- Multiple package managers (npm, yarn, pnpm)

**Perfect for:** Web apps, REST APIs, GraphQL servers

#### 3. Python ML Service (`python-ml-service.yaml`)

**Features:**
- ruff, black, mypy, pytest
- bandit security scanning
- Docker build, sign, scan
- GitOps deployment

**Perfect for:** ML inference, data processing, API backends

### Special Use Cases

#### 4. Container-Only (`container-repo.yaml`)

**Features:**
- Build and release container images only
- Automatic semver tagging
- Image signing and scanning
- Changelog generation

**Perfect for:** Base images, builder images, runner images

#### 5. Simple Template (`simple-app.yaml`)

**Features:**
- All language pipelines in one file (disabled by default)
- Enable the one you need
- Quick starting point

**Perfect for:** New projects, choosing a language

#### 6. Custom Modular (`custom-modular.yaml`)

**Features:**
- Use individual composite actions
- Full control over each step
- Custom deployment logic
- Advanced use cases

**Perfect for:** Cloud Run, Lambda, ECS, non-Kubernetes deployments

### Utility Workflows

#### 7. Cleanup Images (`cleanup-images.yaml`)

**Features:**
- Protection-based cleanup (specify what to keep, delete the rest)
- Git Flow-aware protection patterns
- Separate protections for semver, RC, hotfix RC, and dev-latest
- Custom regex patterns for additional protection
- Dry-run mode for testing
- Scheduled execution (weekly by default)
- Manual trigger support

**How it works:**
- Keeps at least N versions (default: 15)
- Protects specific version patterns (semver, dev-latest, etc.)
- Deletes old versions that don't match protection patterns
- Separately cleans up untagged images

**Default protections:**
- ✅ Semver tags (latest, 1, 1.2, 1.2.3)
- ✅ dev-latest tag
- ❌ RC tags (can be cleaned)
- ❌ Hotfix RC tags (can be cleaned)
- ❌ dev-<sha> tags (can be cleaned)
- ❌ pr-<number> tags (can be cleaned)
- ❌ SHA-only tags (can be cleaned)

**Important:** This workflow uses protection patterns, not age-based deletion. The `actions/delete-package-versions` action doesn't support "delete images older than X days".

**Perfect for:**
- All repositories with container images
- Reducing registry storage costs
- Maintaining clean image inventory
- Compliance with retention policies

## Quick Start

1. **Choose an example** that matches your use case
2. **Copy** the workflow to `.github/workflows/pipeline.yaml` in your repository (single file handles all events)
3. **Customize** environment variables:
   - `APP_NAME`: Application name for GitOps paths (e.g., `my-app`)
   - `IMAGE_NAME`: Full container image path (e.g., `myorg/my-app`)
   - `PROJECT_NAME`: Project name (e.g., `platform`, `devportal`)
4. **Configure secrets** (see Required Configuration below)
5. **Push or create a PR** to trigger the pipeline

**Note:** `APP_NAME` and `IMAGE_NAME` can differ if needed (e.g., multi-org setups or custom registries).

## How It Works

The single workflow file automatically handles different scenarios:

| Trigger | Behavior |
|---------|----------|
| **PR to develop** | Build only (validation, no push, no deploy) |
| **Push to develop** | Build, push (`dev-latest`, `dev-<sha>`), deploy to DEV |
| **PR to main** (from release/hotfix) | Build, push (`X.Y.Z-rc`), deploy to STAGING |
| **Merge to main** | Retag rc→final version, create release, deploy to PRODUCTION |
| **Manual trigger** | Deploy to chosen environment (dev/stg/prd) |

No separate workflows needed - the pipeline detects the Git Flow context automatically!

## Required Configuration

### Secrets (Organization or Repository Level)

| Secret | Required For | How to Get |
|--------|--------------|------------|
| `GITHUB_TOKEN` | All workflows | Automatic, no setup needed |
| `ARGOCD_AUTH_TOKEN` | ArgoCD deployments | `argocd account generate-token` |
| `ARGOCD_SERVER` | ArgoCD deployments | Your ArgoCD server URL (e.g., `argocd.example.com`) |
| `CICD_USERNAME` | Runner authentication | GitHub username or organization name |
| `CICD_TOKEN` | Runner authentication | GitHub PAT with `read:packages` scope |

**Note:** The `CICD_USERNAME` and `CICD_TOKEN` are used to authenticate when pulling the `runner-base` container image from GHCR on self-hosted runners.

**Setup Example:**
```bash
# Using GitHub CLI
gh secret set ARGOCD_AUTH_TOKEN --org YOUR_ORG --body "your-argocd-token"
gh secret set ARGOCD_SERVER --org YOUR_ORG --body "argocd.example.com"
gh secret set CICD_USERNAME --org YOUR_ORG --body "your-github-username"
gh secret set CICD_TOKEN --org YOUR_ORG --body "ghp_your_personal_access_token"
```

### Variables (Optional)

| Variable | Purpose | Default |
|----------|---------|---------|
| `REGISTRY` | Container registry | `ghcr.io` |

## Git Flow Setup

These workflows expect Git Flow branching:

```bash
# Install git-flow (optional but recommended)
brew install git-flow  # macOS
apt-get install git-flow  # Ubuntu

# Initialize Git Flow in your repo
git flow init

# Use default branch names:
# Production: main
# Development: develop
# Feature prefix: feature/
# Release prefix: release/
# Hotfix prefix: hotfix/
```

## Typical Workflow

### Feature Development

```bash
git flow feature start my-feature
# Make changes
git add .
git commit -m "feat: implement my feature"
git flow feature finish my-feature
# Creates PR to develop (or push develop)
```

### Release

```bash
# From develop branch
git flow release start 1.2.0

# Update version, changelog, etc.
git add .
git commit -m "chore: prepare release 1.2.0"

git flow release finish 1.2.0
# Merges to main and develop, creates tag
```

### Hotfix

```bash
# From main branch
git flow hotfix start 1.2.1

# Fix the bug
git add .
git commit -m "fix: critical bug"

git flow hotfix finish 1.2.1
# Merges to main and develop, creates tag
```

## Testing Locally

Before pushing, you can validate workflows locally:

```bash
# Install act (GitHub Actions local runner)
brew install act  # macOS
# or download from https://github.com/nektos/act

# Run workflow locally
act -j pipeline
```

## Migration Guide

### From Generic Workflows

**Before:**
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - run: npm ci
      - run: npm test
      - run: docker build -t my-app .
      - run: docker push my-app
      # ... 50+ more lines
```

**After:**
```yaml
env:
  APP_NAME: my-app
  IMAGE_NAME: org/my-app
  PROJECT_NAME: devportal

jobs:
  pipeline:
    uses: mrops-br/gha-workflows/.github/workflows/_pipeline-nodejs.yaml@main
    with:
      app-name: ${{ env.APP_NAME }}
      image-name: ${{ env.IMAGE_NAME }}
      project-name: ${{ env.PROJECT_NAME }}
    secrets: inherit
```

### From Custom Docker Builds

Replace manual Docker commands with the modular approach:

**Before:**
```yaml
- name: Build
  run: docker build -t $IMAGE .
- name: Tag
  run: docker tag $IMAGE $IMAGE:latest
- name: Push
  run: docker push $IMAGE
```

**After:**
```yaml
- uses: mrops-br/gha-workflows/.github/actions/docker/build-push@main
  with:
    context: .
    image-name: my-app
    push: true
```

## Troubleshooting

See [GITFLOW-WORKFLOWS.md](../docs/GITFLOW-WORKFLOWS.md#troubleshooting) for common issues and solutions.

## Contributing

To add a new example:

1. Create a new YAML file in this directory
2. Add comprehensive comments explaining the use case
3. Include expected project structure
4. Update this README
5. Submit a PR

## Support

For questions or issues:
- Check [GITFLOW-WORKFLOWS.md](../docs/GITFLOW-WORKFLOWS.md)
- Review existing examples
- Open an issue in this repository
- Contact the Platform Engineering team
