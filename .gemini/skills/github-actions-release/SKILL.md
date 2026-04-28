# Skill: GitHub Actions Release Workflow

Scaffold and maintain GitHub Actions workflows for automated releases, including multi-environment deployments with approval gates, matrix builds, and OIDC authentication.

---

## When to Use This Skill

- User asks to "set up a release workflow," "create a CI/CD pipeline," or "automate releases"
- When scaffolding `.github/workflows/release.yml` for a new or existing project
- When modifying an existing release workflow to add new stages or features

---

## Inputs

| Input | Required | Description |
|-------|----------|-------------|
| Trigger type | No | `tag-push` (default), `release-please`, `manual`, or `pr-merge` |
| Build steps | No | Custom build commands (default: auto-detected from project type) |
| Environments | No | Deployment targets (default: `staging`, `production`) |
| Runner type | No | `github-hosted` (default) or `self-hosted` |
| Multi-platform | No | Build matrix for cross-platform artifacts (default: `false`) |

---

## Process

### Step 1: Detect Project Type

Inspect the repository to determine the build system:

| File Present | Project Type | Build Command | Artifact |
|-------------|-------------|---------------|----------|
| `package.json` | Node.js | `npm ci && npm run build` | `dist/` |
| `go.mod` | Go | `go build -o bin/` | `bin/` |
| `Cargo.toml` | Rust | `cargo build --release` | `target/release/` |
| `pyproject.toml` | Python | `python -m build` | `dist/*.whl` |
| `Dockerfile` | Container | `docker build` | Container image |
| `Makefile` | Generic | `make release` | Varies |

### Step 2: Generate Workflow File

Create `.github/workflows/release.yml` with the following structure:

```yaml
name: Release

on:
  push:
    tags: ['v[0-9]+.[0-9]+.[0-9]+*']
  workflow_dispatch:
    inputs:
      version:
        description: 'Version to release (e.g., v1.2.3)'
        required: true
      environment:
        description: 'Target environment'
        required: true
        default: 'staging'
        type: choice
        options: [staging, production]
      dry-run:
        description: 'Dry run (no actual release)'
        required: false
        default: false
        type: boolean

permissions:
  contents: write
  packages: write
  id-token: write
  attestations: write

concurrency:
  group: release-${{ github.ref }}
  cancel-in-progress: false

jobs:
  prepare:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.version.outputs.version }}
      changelog: ${{ steps.changelog.outputs.entry }}
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Extract version from tag
        id: version
        run: |
          if [ "${{ github.event_name }}" = "workflow_dispatch" ]; then
            echo "version=${{ inputs.version }}" >> "$GITHUB_OUTPUT"
          else
            echo "version=${GITHUB_REF_NAME}" >> "$GITHUB_OUTPUT"
          fi

      - name: Extract changelog entry
        id: changelog
        run: |
          VERSION="${{ steps.version.outputs.version }}"
          # Extract the section for this version from CHANGELOG.md
          ENTRY=$(sed -n "/^## \[${VERSION#v}\]/,/^## \[/{ /^## \[${VERSION#v}\]/d; /^## \[/d; p; }" CHANGELOG.md)
          echo "entry<<EOF" >> "$GITHUB_OUTPUT"
          echo "$ENTRY" >> "$GITHUB_OUTPUT"
          echo "EOF" >> "$GITHUB_OUTPUT"

  build:
    needs: prepare
    runs-on: ubuntu-latest
    strategy:
      matrix:
        # Customize per project:
        os: [ubuntu-latest]
        # For multi-platform: os: [ubuntu-latest, macos-latest, windows-latest]
    steps:
      - uses: actions/checkout@v4

      - name: Set up build environment
        # Language-specific setup (Node.js example):
        uses: actions/setup-node@v4
        with:
          node-version: 'lts/*'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test

      - name: Build
        run: npm run build

      - name: Generate checksums
        run: |
          cd dist
          sha256sum * > SHA256SUMS.txt

      - name: Upload build artifacts
        uses: actions/upload-artifact@v4
        with:
          name: release-artifacts-${{ matrix.os }}
          path: |
            dist/
            SHA256SUMS.txt

  publish:
    needs: [prepare, build]
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Download artifacts
        uses: actions/download-artifact@v4
        with:
          pattern: release-artifacts-*
          merge-multiple: true
          path: artifacts/

      - name: Create GitHub Release
        env:
          GH_TOKEN: ${{ github.token }}
        run: |
          gh release create "${{ needs.prepare.outputs.version }}" \
            --repo "${{ github.repository }}" \
            --title "Release ${{ needs.prepare.outputs.version }}" \
            --notes "${{ needs.prepare.outputs.changelog }}" \
            artifacts/*

  deploy-staging:
    needs: [prepare, publish]
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - name: Deploy to staging
        run: |
          echo "Deploying ${{ needs.prepare.outputs.version }} to staging..."
          # Add deployment commands here

      - name: Run smoke tests
        run: |
          echo "Running smoke tests against staging..."
          # Add smoke test commands here

  deploy-production:
    needs: [prepare, deploy-staging]
    runs-on: ubuntu-latest
    environment:
      name: production
      # Manual approval required (configure in GitHub repo settings)
    steps:
      - name: Deploy to production
        run: |
          echo "Deploying ${{ needs.prepare.outputs.version }} to production..."
          # Add deployment commands here

      - name: Verify deployment
        run: |
          echo "Verifying production deployment..."
          # Add health check commands here
```

### Step 3: Configure GitHub Environments

Instruct the user to configure environments in GitHub:

1. Go to **Settings → Environments**
2. Create `staging` and `production` environments
3. For `production`:
   - Add **required reviewers** (at least 1)
   - Set **wait timer** (recommended: 15 minutes)
   - Restrict to `main` branch
   - Optionally prevent self-reviews

### Step 4: Add Release-Please (optional)

If the user wants automated version management:

```yaml
# .github/workflows/release-please.yml
name: Release Please

on:
  push:
    branches: [main]

permissions:
  contents: write
  pull-requests: write

jobs:
  release-please:
    runs-on: ubuntu-latest
    steps:
      - uses: googleapis/release-please-action@v4
        with:
          release-type: node  # or: go, python, rust, simple
```

---

## Output

Provide:

1. The complete workflow YAML file
2. Instructions for configuring GitHub environments
3. Any required secrets or variables
4. Next steps (e.g., "add signing with the artifact-signing-sbom skill")

---

## References

See `references/workflow-templates.md` for reusable YAML snippets for common patterns.
