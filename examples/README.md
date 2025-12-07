# Workflow Examples

This directory contains example workflows demonstrating different use cases and patterns.

## Examples

### 1. Container-Only Repository (`container-repo.yaml`)

**Use case:** Building and releasing Docker images without application deployment (like `gha-runner-images`).

**Features:**
- Automatic Git Flow tagging
- Automated release on merge to main
- Image signing and scanning
- Changelog generation
- Semver tagging (X, X.Y, X.Y.Z, latest)

**Perfect for:**
- Base images
- Builder images
- Tool images
- Runner images

### 2. Full-Stack Node.js Application (`fullstack-nodejs.yaml`)

**Use case:** Complete CI/CD for Node.js applications with GitOps deployment.

**Features:**
- Lint, test, coverage
- Docker build and push
- ArgoCD deployment
- Automatic environment detection
- Manual production deployment

**Perfect for:**
- REST APIs
- GraphQL servers
- Web applications
- Microservices

### 3. Go Microservice (`golang-microservice.yaml`)

**Use case:** Go backend service with comprehensive testing.

**Features:**
- golangci-lint
- Unit tests with race detection
- Coverage reporting
- Security scanning with gosec
- GitOps deployment

**Perfect for:**
- Backend services
- gRPC services
- CLI tools
- System services

### 4. Python ML Service (`python-ml-service.yaml`)

**Use case:** Python application with type checking and ML model testing.

**Features:**
- Linting with ruff
- Type checking with mypy
- Code formatting with black
- Security scanning with bandit
- pytest with coverage

**Perfect for:**
- ML inference services
- Data processing services
- API backends
- Batch jobs

### 5. Custom Modular Pipeline (`custom-modular.yaml`)

**Use case:** Fine-grained control using individual building blocks.

**Features:**
- Uses composite actions directly
- Custom CI steps
- Custom deployment (non-ArgoCD)
- Multi-platform builds
- Manual release orchestration

**Perfect for:**
- Custom deployment platforms (Cloud Run, Lambda, ECS)
- Non-Kubernetes deployments
- Hybrid CI/CD workflows
- Special requirements

## Quick Start

1. **Choose an example** that matches your use case
2. **Copy** the workflow to `.github/workflows/pipeline.yaml` in your repository
3. **Customize** environment variables (`APP_NAME`, `PROJECT`, etc.)
4. **Configure secrets** (see below)
5. **Push** to trigger the pipeline

## Required Configuration

### Secrets (Organization or Repository Level)

| Secret | Required For | How to Get |
|--------|--------------|------------|
| `GITHUB_TOKEN` | All workflows | Automatic, no setup needed |
| `ARGOCD_AUTH_TOKEN` | ArgoCD deployments | `argocd account generate-token` |
| `ARGOCD_SERVER` | ArgoCD deployments | Your ArgoCD server URL (e.g., `argocd.example.com`) |

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
jobs:
  pipeline:
    uses: mrops-br/gha-workflows/.github/workflows/_pipeline-nodejs.yaml@main
    with:
      app-name: my-app
      project: devportal
      gitops-repo: org/gitops-devportal
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
