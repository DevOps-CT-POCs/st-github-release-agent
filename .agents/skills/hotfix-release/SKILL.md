---
name: hotfix-release
description: Use when users report a critical production bug, need an emergency patch, or require a hotfix release. Handles the full hotfix lifecycle — branching from a release tag, cherry-picking fixes, expedited CI, production deployment with approval, back-merging to main, and post-mortem generation.
---

Handle emergency production patches — branch from tag, cherry-pick fixes, fast-track CI, deploy with approval.

## When to Use

- Critical production bug needing immediate fix
- Emergency patch, hotfix, or security vulnerability
- Cherry-picking specific fixes for expedited release

## Workflow

### 1. Gather Inputs

| Input | Required | Description |
|-------|----------|-------------|
| Issue description | Yes | What's broken and severity (critical/high) |
| Target release tag | No | Which release to patch (default: latest) |
| Fix commits/PRs | No | Specific commits or PRs to cherry-pick |

### 2. Identify Release to Patch

```bash
LATEST_TAG=$(git describe --tags --abbrev=0 --match 'v[0-9]*.[0-9]*.[0-9]*' 2>/dev/null)
```

### 3. Create Hotfix Branch

Branch from the **release tag** (not `main`): `hotfix/v{major}.{minor}.{patch+1}`

### 4. Apply Fix

Cherry-pick existing commits or write fix directly. All commits must use `fix:` type.

### 5. Version Bump & Tag

**Always PATCH only** — never MINOR or MAJOR from a hotfix. Create annotated tag with severity, issue description, and fix details.

### 6. Fast-Track CI

Hotfix workflow skips staging → deploys directly to production with approval. Runs only critical/smoke tests.

### 7. Back-merge to Main

Merge hotfix branch back to `main`. If conflicts, create a PR instead of forcing.

### 8. Post-Mortem

Generate a post-mortem covering: summary, timeline, root cause, fix applied, action items, lessons learned.

## Answer Shape

Hotfix branch commands → fix instructions → version bump & tag → back-merge → post-mortem template.

## Edge Cases

- **Multiple tags on same commit** → use highest semver tag
- **Conflicts with main** → create PR for back-merge
- **Patching older release** → support `v1.x` while `v2.x` is current
- **No test suite** → warn about manual verification; proceed
- **Stacked hotfixes** → build on existing hotfix branch if first isn't merged back yet
