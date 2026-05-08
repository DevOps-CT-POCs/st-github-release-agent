---
name: github-actions-release
description: Use when users ask to set up a release workflow, create a CI/CD pipeline, automate releases, or scaffold GitHub Actions workflows for building, testing, and publishing releases with multi-environment deployments, approval gates, matrix builds, and OIDC authentication.
---

Scaffold and maintain GitHub Actions workflows for automated releases, including multi-environment deployments with approval gates, matrix builds, and OIDC authentication.

## When to Use

Use this skill when the request is about:

- Setting up a release workflow or CI/CD pipeline for automated releases
- Scaffolding `.github/workflows/release.yml` for a new or existing project
- Modifying an existing release workflow to add new stages or features
- Configuring deployment environments with approval gates
- Adding Release Please or similar automated version management

Do not use this skill for:

- Determining the version number. Use `semantic-versioning`.
- Generating changelogs. Use `changelog-generation`.
- Signing artifacts or generating SBOMs. Use `artifact-signing-sbom`.
- General GitHub Actions syntax or documentation questions. Use `github-actions-docs`.
- Emergency hotfix releases. Use `hotfix-release`.

## Workflow

### 1. Detect Project Type

Inspect the repository to determine the build system:

| File Present | Project Type | Build Command | Artifact |
|-------------|-------------|---------------|----------|
| `package.json` | Node.js | `npm ci && npm run build` | `dist/` |
| `go.mod` | Go | `go build -o bin/` | `bin/` |
| `Cargo.toml` | Rust | `cargo build --release` | `target/release/` |
| `pyproject.toml` | Python | `python -m build` | `dist/*.whl` |
| `Dockerfile` | Container | `docker build` | Container image |
| `Makefile` | Generic | `make release` | Varies |

### 2. Generate the workflow YAML

Create `.github/workflows/release.yml` with jobs for: prepare (extract version + changelog), build (with optional matrix), publish (create GitHub Release), deploy-staging, and deploy-production (with manual approval).

Key requirements:
- Trigger on tag push `v[0-9]+.[0-9]+.[0-9]+*` and `workflow_dispatch`
- Add `permissions:` block with least privilege (`contents: write`, `packages: write`, `id-token: write`, `attestations: write`)
- Use `concurrency` with `cancel-in-progress: false`
- Pin actions to specific versions
- Generate SHA256 checksums for artifacts

### 3. Configure GitHub Environments

Instruct the user to set up `staging` and `production` environments in GitHub Settings with required reviewers and branch restrictions.

### 4. Optionally add Release Please

Scaffold `.github/workflows/release-please.yml` for automated version management.

## Answer Shape

1. The complete workflow YAML file
2. Instructions for configuring GitHub environments
3. Any required secrets or variables
4. Next steps (e.g., "add signing with the artifact-signing-sbom skill")

## Common Mistakes

- Using mutable action tags instead of pinning to full SHA hashes for third-party actions
- Missing the `permissions:` block, defaulting to overly broad access
- Setting `cancel-in-progress: true` on release workflows
- Forgetting `fetch-depth: 0` when the workflow needs full Git history

## Bundled Reference

See `references/workflow-templates.md` for reusable YAML snippets for Docker builds, multi-platform matrices, Slack notifications, rollback jobs, and OIDC cloud authentication.
