---
name: semantic-versioning
description: Use when users ask to bump the version, create a release, determine the next version, validate version consistency, or manage pre-release identifiers. Calculates the correct semantic version bump from Git commit history using Conventional Commits, creates annotated tags, and updates manifest files.
---

Calculate the correct semantic version bump from Git commit history, create annotated tags, and manage pre-release identifiers.

## When to Use

Use this skill when the request is about:

- Bumping the version, creating a release, or determining "what's the next version?"
- Calculating a version bump from commit history before generating a changelog (this skill runs first)
- Validating version consistency across manifest files (`package.json`, `Cargo.toml`, `pyproject.toml`, etc.)
- Managing pre-release channels (`alpha`, `beta`, `rc`)
- Creating annotated Git tags for releases

Do not use this skill for:

- Generating changelogs or release notes. Use `changelog-generation`.
- Scaffolding CI/CD release workflows. Use `github-actions-release`.
- Emergency hotfix patches. Use `hotfix-release`.

## Workflow

### 1. Gather Inputs

| Input | Required | Description |
|-------|----------|-------------|
| Repository path | Yes | Path to the Git repository |
| Pre-release channel | No | One of: `alpha`, `beta`, `rc` (default: stable release) |
| Force bump type | No | Override automatic detection with `major`, `minor`, or `patch` |

### 2. Identify the Latest Release Tag

```bash
# Find the latest semver tag on the current branch
git describe --tags --abbrev=0 --match 'v[0-9]*.[0-9]*.[0-9]*' 2>/dev/null || echo "v0.0.0"
```

If no tag exists, start from `v0.0.0`.

### 3. Collect Commits Since Last Tag

```bash
# Get all commits since the last tag
git log $(git describe --tags --abbrev=0 --match 'v*' 2>/dev/null || echo "")..HEAD \
  --pretty=format:"%H %s" --no-merges
```

### 4. Parse Conventional Commits

Analyze each commit message to determine the bump type:

```
Priority (highest to lowest):
  1. MAJOR — Any commit with `!` after type/scope OR `BREAKING CHANGE:` in footer
  2. MINOR — Any commit starting with `feat:`
  3. PATCH — Any commit starting with `fix:`
  4. NONE  — `chore:`, `docs:`, `ci:`, `test:`, `style:`, `refactor:`, `build:`, `perf:`
```

**The highest-priority bump type wins.** If there are no version-bumping commits, report "No version bump needed."

#### Parsing Rules

```
Pattern: ^(?<type>[a-z]+)(?:\((?<scope>[^)]+)\))?(?<breaking>!)?: (?<description>.+)$

Examples:
  "feat: add user auth"           → type=feat, breaking=false → MINOR
  "feat(api)!: redesign endpoints" → type=feat, breaking=true  → MAJOR
  "fix: resolve race condition"    → type=fix,  breaking=false → PATCH
  "chore: update deps"            → type=chore, breaking=false → NONE
```

Also check commit body/footer for `BREAKING CHANGE:` or `BREAKING-CHANGE:`:

```
feat: new auth system

BREAKING CHANGE: The `login()` method now requires an options object
```

### 5. Calculate the New Version

```
Current: vMAJOR.MINOR.PATCH

If bump == MAJOR → v(MAJOR+1).0.0
If bump == MINOR → vMAJOR.(MINOR+1).0
If bump == PATCH → vMAJOR.MINOR.(PATCH+1)
```

#### Pre-release Versions

If a pre-release channel is specified:

```
# First pre-release for this version
v1.2.0-alpha.1

# Subsequent pre-releases (same version, same channel)
v1.2.0-alpha.2

# Promoting to next channel
v1.2.0-beta.1

# Promoting to stable
v1.2.0
```

### 6. Validate Version Consistency

Check and update version in relevant manifest files:

| File | Field | Language |
|------|-------|----------|
| `package.json` | `"version"` | Node.js |
| `Cargo.toml` | `version` in `[package]` | Rust |
| `pyproject.toml` | `version` in `[project]` or `[tool.poetry]` | Python |
| `go.mod` | Module path suffix (e.g., `/v2`) | Go (major only) |
| `build.gradle` / `build.gradle.kts` | `version` property | Java/Kotlin |
| `version.txt` | Plain text | Any |

```bash
# Example: Update package.json
npm version <new_version> --no-git-tag-version

# Example: Update pyproject.toml
sed -i 's/^version = ".*"/version = "'$NEW_VERSION'"/' pyproject.toml

# Example: Update Cargo.toml
sed -i 's/^version = ".*"/version = "'$NEW_VERSION'"/' Cargo.toml
```

### 7. Create Annotated Git Tag

```bash
# Create the annotated tag
git tag -a "v${NEW_VERSION}" -m "Release v${NEW_VERSION}

$(cat <<'EOF'
## Changes in this release

<insert changelog summary here>
EOF
)"

# Push the tag
git push origin "v${NEW_VERSION}"
```

## Answer Shape

Provide a structured summary:

```markdown
## Version Bump Summary

| Field | Value |
|-------|-------|
| Previous version | v1.2.3 |
| Bump type | MINOR |
| New version | v1.3.0 |
| Commits analyzed | 12 |
| Breaking changes | 0 |
| Features | 3 |
| Fixes | 5 |
| Other | 4 |

### Commits included:
- `feat: add OAuth2 SSO support` → MINOR
- `fix: resolve null pointer in auth handler` → PATCH
- `fix: correct timezone handling in scheduler` → PATCH
- ...
```

## Edge Cases

1. **No commits since last tag** → Report "No version bump needed"
2. **No existing tags** → Start from `v0.0.0`, apply bump → `v0.1.0` or `v1.0.0`
3. **Pre-release already exists** → Increment pre-release counter (`-alpha.1` → `-alpha.2`)
4. **Merge commits** → Skip merge commits (`--no-merges`), analyze only direct commits
5. **Non-conventional commits** → Warn the user, treat as PATCH by default or skip
6. **Monorepo** → If detected, ask user which package to version (future enhancement)

## Common Mistakes

- Forgetting to check commit footers for `BREAKING CHANGE:` (not just the `!` suffix)
- Skipping manifest file updates, causing version drift between tags and `package.json`/`Cargo.toml`
- Using lightweight tags instead of annotated tags
- Not running with `--no-merges`, which double-counts squashed commits
