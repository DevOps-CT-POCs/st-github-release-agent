---
name: hotfix-release
description: Use when users report a critical production bug, need an emergency patch, or require a hotfix release. Handles the full hotfix lifecycle — branching from a release tag, cherry-picking fixes, expedited CI, production deployment with approval, back-merging to main, and post-mortem generation.
---

Handle emergency production patches with an expedited release workflow — branch from tag, cherry-pick fixes, fast-track CI, and deploy with approval.

## When to Use

Use this skill when the request is about:

- A critical bug in production that needs an immediate fix
- An emergency patch, hotfix, or production fix
- A security vulnerability requiring an out-of-band release
- Cherry-picking specific fixes for an expedited release

Do not use this skill for:

- Regular releases or version bumps. Use `semantic-versioning` and `changelog-generation`.
- Setting up CI/CD pipelines. Use `github-actions-release`.
- Signing artifacts. Use `artifact-signing-sbom`.

## Workflow

### 1. Gather Inputs

| Input | Required | Description |
|-------|----------|-------------|
| Issue description | Yes | What's broken and its severity (critical/high) |
| Target release tag | No | Which release to patch (default: latest) |
| Fix commits/PRs | No | Specific commits or PRs to cherry-pick |

### 2. Identify the Release to Patch

```bash
LATEST_TAG=$(git describe --tags --abbrev=0 --match 'v[0-9]*.[0-9]*.[0-9]*' 2>/dev/null)
echo "Patching from: ${LATEST_TAG}"
```

### 3. Create the Hotfix Branch

Branch naming: `hotfix/v{major}.{minor}.{patch+1}`. Always branch from the release tag, not from `main`.

### 4. Apply the Fix

Either cherry-pick existing commits from main or write the fix directly. All commits must follow Conventional Commits format with `fix:` type.

### 5. Version Bump (Patch Only)

Hotfixes are **always PATCH bumps**. Never bump MINOR or MAJOR from a hotfix branch.

### 6. Create Hotfix Release Tag

Create an annotated tag with severity, issue description, and fix details.

### 7. Fast-Track CI Pipeline

The hotfix workflow (`hotfix.yml`) skips staging and deploys directly to production with approval. It runs only critical/smoke tests.

### 8. Back-merge to Main

After deployment, merge the hotfix branch back to `main`. If conflicts exist, create a PR instead.

### 9. Post-Mortem Template

Generate a post-mortem document covering: summary, timeline, root cause, fix applied, action items, and lessons learned.

## Answer Shape

Provide:

1. Hotfix branch name and creation commands
2. Cherry-pick or fix application instructions
3. Version bump and tag commands
4. Back-merge instructions
5. Post-mortem template (filled in with available details)

## Edge Cases

1. **Multiple tags on the same commit** → Use the highest semver tag
2. **Hotfix conflicts with main** → Create a PR for the back-merge
3. **Hotfix on an older release** → Support patching `v1.x` while `v2.x` is current
4. **No test suite** → Warn that manual verification is required; still proceed
5. **Dependent hotfixes** → If a second hotfix is needed before the first is merged back, stack on the existing hotfix branch

## Common Mistakes

- Bumping MINOR or MAJOR from a hotfix branch (always PATCH only)
- Forgetting to back-merge the hotfix into `main`, causing the fix to be lost
- Branching from `main` instead of from the release tag
- Skipping the post-mortem, losing organizational learning
