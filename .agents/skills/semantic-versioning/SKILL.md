---
name: semantic-versioning
description: Use when users ask to bump the version, create a release, determine the next version, validate version consistency, or manage pre-release identifiers. Calculates the correct semantic version bump from Git commit history using Conventional Commits, creates annotated tags, and updates manifest files.
---

Calculate the correct semantic version bump from Git commit history, create annotated tags, and manage pre-release identifiers.

## When to Use

- Bumping the version, creating a release, or determining the next version
- Calculating a version bump before generating a changelog (this skill runs first)
- Validating version consistency across manifest files
- Managing pre-release channels (`alpha`, `beta`, `rc`)

## Workflow

### 1. Gather Inputs

| Input | Required | Description |
|-------|----------|-------------|
| Repository path | Yes | Path to the Git repository |
| Pre-release channel | No | `alpha`, `beta`, or `rc` (default: stable) |
| Force bump type | No | Override with `major`, `minor`, or `patch` |

### 2. Find Latest Tag & Collect Commits

```bash
# Latest semver tag (fallback: v0.0.0)
git describe --tags --abbrev=0 --match 'v[0-9]*.[0-9]*.[0-9]*' 2>/dev/null || echo "v0.0.0"

# Commits since last tag
git log <last_tag>..HEAD --pretty=format:"%H %s" --no-merges
```

### 3. Parse & Classify Commits

```
Pattern: ^(?<type>[a-z]+)(?:\((?<scope>[^)]+)\))?(?<breaking>!)?: (?<description>.+)$

Priority: MAJOR (! suffix or BREAKING CHANGE:/BREAKING-CHANGE: in footer)
        → MINOR (feat:) → PATCH (fix:) → NONE (all others)
```

The highest-priority bump wins. No version-bumping commits → "No version bump needed."

### 4. Calculate New Version

```
MAJOR bump → v(MAJOR+1).0.0
MINOR bump → vMAJOR.(MINOR+1).0
PATCH bump → vMAJOR.MINOR.(PATCH+1)
```

Pre-release: `v1.2.0-alpha.1` → `v1.2.0-alpha.2` (increment counter). Promote: `-alpha` → `-beta.1` → `-rc.1` → stable.

### 5. Update Manifest Files

| File | Field |
|------|-------|
| `package.json` | `"version"` |
| `Cargo.toml` | `version` in `[package]` |
| `pyproject.toml` | `version` in `[project]` or `[tool.poetry]` |
| `go.mod` | Module path suffix (`/v2`) for major only |
| `build.gradle(.kts)` | `version` property |
| `version.txt` | Plain text |

### 6. Create Annotated Tag & Push

```bash
git tag -a "v${NEW_VERSION}" -m "Release v${NEW_VERSION}"
git push origin "v${NEW_VERSION}"
```

## Answer Shape

Summary table with: previous version, bump type, new version, commits analyzed, breaking changes count, features count, fixes count. Then list each commit with its classification.

## Edge Cases

- **No commits since last tag** → "No version bump needed"
- **No existing tags** → Start from `v0.0.0`
- **Pre-release exists** → Increment counter (`-alpha.1` → `-alpha.2`)
- **Merge commits** → Skip with `--no-merges`
- **Non-conventional commits** → Warn user; treat as PATCH or skip
