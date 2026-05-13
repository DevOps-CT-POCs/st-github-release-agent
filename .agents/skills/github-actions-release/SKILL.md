---
name: github-actions-release
description: Use when users ask to set up a release workflow, create a CI/CD pipeline, automate releases, or scaffold GitHub Actions workflows for building, testing, and publishing releases with multi-environment deployments, approval gates, matrix builds, and OIDC authentication.
---

Scaffold and maintain GitHub Actions release workflows with multi-environment deployments, approval gates, matrix builds, and OIDC authentication.

## When to Use

- Setting up or modifying a release workflow / CI/CD pipeline
- Scaffolding `.github/workflows/release.yml`
- Configuring deployment environments with approval gates
- Adding Release Please or automated version management

## Workflow

### 1. Detect Project Type

| File | Type | Build Command | Artifact |
|------|------|---------------|----------|
| `package.json` | Node.js | `npm ci && npm run build` | `dist/` |
| `go.mod` | Go | `go build -o bin/` | `bin/` |
| `Cargo.toml` | Rust | `cargo build --release` | `target/release/` |
| `pyproject.toml` | Python | `python -m build` | `dist/*.whl` |
| `Dockerfile` | Container | `docker build` | Image |
| `Makefile` | Generic | `make release` | Varies |

### 2. Generate Workflow YAML

Create `.github/workflows/release.yml` with jobs: prepare (version + changelog) → build (optional matrix) → publish (GitHub Release) → deploy-staging → deploy-production (manual approval).

**Required configuration:**
- Trigger: tag push `v[0-9]+.[0-9]+.[0-9]+*` and `workflow_dispatch`
- `permissions:` block: `contents: write`, `packages: write`, `id-token: write`, `attestations: write`
- `concurrency:` with `cancel-in-progress: false`
- Pin actions to specific versions; generate SHA256 checksums

### 3. Configure Environments

Instruct user to set up `staging` and `production` environments in GitHub Settings with required reviewers and branch restrictions.

### 4. Optionally Add Release Please

Scaffold `.github/workflows/release-please.yml` for automated version management.

## Answer Shape

Complete workflow YAML + environment setup instructions + required secrets/variables + next steps.

## Common Mistakes

- Mutable action tags instead of pinned SHAs
- Missing `permissions:` block (defaults to overly broad)
- `cancel-in-progress: true` on release workflows (should be `false`)
- Missing `fetch-depth: 0` when full Git history is needed

## Bundled Reference

See `references/workflow-templates.md` for reusable snippets: Docker builds, multi-platform matrices, Slack notifications, rollback jobs, OIDC cloud auth.
