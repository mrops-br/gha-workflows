# Contributing to gha-workflows

Thank you for contributing to the centralized GitHub Actions workflows! This guide will help you make effective contributions.

## Development Workflow

We follow **Git Flow** for this repository:

```
main      ──●────────────────●──────●────>  (Production releases)
            │                 │      │
develop   ──●────●────●───────●──────●────>  (Development)
               │    │         │
feature/*      └────┴─────────┘              (Feature branches)
```

### Branch Strategy

- `main` - Production-ready releases only
- `develop` - Integration branch for ongoing development
- `feature/*` - New features (branch from `develop`)
- `release/*` - Release preparation (branch from `develop`)
- `hotfix/*` - Production fixes (branch from `main`)

## Making Changes

### 1. Create a Feature Branch

```bash
git checkout develop
git pull origin develop
git checkout -b feature/your-feature-name
```

### 2. Make Your Changes

#### Adding a New Workflow

1. Create the workflow file in `.github/workflows/`
2. Prefix reusable workflows with `_` (e.g., `_new-workflow.yaml`)
3. Add comprehensive documentation to README.md
4. Create an example in `examples/`

#### Adding a New Action

1. Create a directory under `.github/actions/action-name/`
2. Add `action.yaml` with proper metadata
3. Document inputs, outputs, and usage
4. Add examples to README.md

#### Modifying Existing Workflows/Actions

1. Update the workflow/action file
2. Update documentation
3. Update examples if needed
4. Test thoroughly before committing

### 3. Test Your Changes

```bash
# Validate YAML syntax
yamllint .github/workflows/*.yaml
yamllint .github/actions/*/action.yaml

# Run validation workflow locally (if possible)
act -W .github/workflows/validate.yaml
```

### 4. Commit Your Changes

Follow [Conventional Commits](https://www.conventionalcommits.org/):

```bash
git add .
git commit -m "feat(workflows): add new deployment workflow"
```

**Commit Types:**
- `feat`: New feature
- `fix`: Bug fix
- `docs`: Documentation changes
- `refactor`: Code refactoring
- `test`: Adding tests
- `chore`: Maintenance tasks
- `ci`: CI/CD changes

**Examples:**
```bash
feat(actions): add semantic-version action
fix(deploy): correct ArgoCD sync timeout
docs(readme): update build-docker workflow examples
refactor(ci): simplify validation steps
```

### 5. Push and Create PR

```bash
git push origin feature/your-feature-name
```

Then create a Pull Request:
- **Target**: `develop` branch
- **Title**: Clear, descriptive title
- **Description**: Explain what and why
- **Testing**: Describe how you tested

## Code Review Process

1. **Automated Checks**: Must pass all validation checks
2. **Code Review**: Requires approval from `@mrops-br/platform-team-leads`
3. **Testing**: Reviewers will test in their environments
4. **Merge**: Squash and merge to `develop`

## Release Process

### Creating a Release

1. **Start Release Branch**
   ```bash
   git checkout develop
   git pull origin develop
   git checkout -b release/1.2.0
   ```

2. **Update Version References**
   - Update README examples if needed
   - Review CHANGELOG (auto-generated on merge)

3. **Open PR to `main`**
   ```bash
   git push origin release/1.2.0
   ```
   - Create PR from `release/1.2.0` to `main`
   - Title: "Release v1.2.0"

4. **After Merge**
   - Tag created automatically: `v1.2.0`
   - CHANGELOG updated automatically
   - GitHub Release created automatically
   - Changes merged back to `develop` automatically

## Best Practices

### Workflow Design

1. **Keep Workflows Focused**
   - One workflow = one purpose
   - Use inputs for flexibility
   - Provide sensible defaults

2. **Make Workflows Reusable**
   - Use `workflow_call` trigger
   - Accept inputs for configuration
   - Output important values

3. **Handle Errors Gracefully**
   - Use `continue-on-error` when appropriate
   - Provide clear error messages
   - Generate helpful summaries

### Action Design

1. **Clear Inputs/Outputs**
   - Document all inputs
   - Specify required vs optional
   - Define output values

2. **Idempotent Operations**
   - Actions should be safe to run multiple times
   - Check before making changes
   - Report what changed

3. **Security**
   - Never log secrets
   - Use proper authentication
   - Validate inputs

### Documentation

1. **README.md**
   - Update for any public-facing changes
   - Include usage examples
   - Document all inputs/outputs

2. **Examples**
   - Keep examples up-to-date
   - Cover common use cases
   - Include comments

3. **Inline Comments**
   - Explain complex logic
   - Document workarounds
   - Reference issues/PRs

## Testing

### Manual Testing

1. **Create Test Repository**
   ```bash
   # Create a test app repository
   # Add workflow using your changes
   # Test all scenarios
   ```

2. **Test Scenarios**
   - Push to `develop`
   - PR to `develop`
   - PR to `main` (from `release/*`)
   - Manual workflow_dispatch
   - Error conditions

### Automated Testing

Our validation workflow checks:
- YAML syntax
- Action metadata completeness
- Workflow structure
- Documentation coverage
- Security issues

## Common Pitfalls

### 1. Breaking Changes

⚠️ **Avoid breaking existing workflows!**

If you must make breaking changes:
1. Deprecate old behavior first
2. Document migration path
3. Update all examples
4. Consider major version bump

### 2. Secrets Management

❌ **Don't:**
- Log secrets (even partially)
- Echo secret values
- Store secrets in code

✅ **Do:**
- Use `secrets` context
- Mask sensitive output
- Document required secrets

### 3. Runner Compatibility

Ensure your changes work with:
- GitHub-hosted runners
- Self-hosted runners
- Different operating systems (if applicable)

## Getting Help

- **Questions**: Open a Discussion on GitHub
- **Bugs**: Open an Issue
- **Security**: Email security@mrops-br.com
- **Slack**: #platform-team channel

## Code of Conduct

- Be respectful and professional
- Provide constructive feedback
- Help others learn and grow
- Follow organizational guidelines

## License

By contributing, you agree that your contributions will be licensed under the MIT License.
