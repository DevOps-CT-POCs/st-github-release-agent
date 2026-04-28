# Skill: Changelog Generation

Parse Conventional Commits into a structured, human-readable `CHANGELOG.md` following the Keep a Changelog format.

---

## When to Use This Skill

- After `semantic-versioning` has determined the new version
- User asks to "generate a changelog," "update release notes," or "what changed?"
- As part of the standard release flow

---

## Inputs

| Input | Required | Description |
|-------|----------|-------------|
| New version | Yes | The version string (e.g., `1.3.0`) from semantic-versioning |
| Previous tag | Yes | The Git tag to compare from (e.g., `v1.2.0`) |
| Repository URL | Yes | GitHub repository URL for compare links |
| Include contributors | No | Whether to list contributors (default: `true`) |

---

## Process

### Step 1: Collect Commits Between Tags

```bash
# Get structured commit data
git log v1.2.0..HEAD --pretty=format:"%H|%an|%ae|%s" --no-merges
```

### Step 2: Categorize Commits

Map each commit to a changelog section based on its Conventional Commit type:

| Commit Type | Changelog Section | Emoji |
|------------|-------------------|-------|
| `feat` | Features | 🚀 |
| `fix` | Bug Fixes | 🐛 |
| `BREAKING CHANGE` / `!` | Breaking Changes | ⚠️ |
| `docs` | Documentation | 📝 |
| `perf` | Performance | ⚡ |
| `chore`, `ci`, `build`, `refactor` | Maintenance | 🔧 |
| `test` | Tests | 🧪 |
| `style` | Styles | 💄 |
| `revert` | Reverts | ⏪ |

**Ordering:** Breaking Changes → Features → Bug Fixes → Performance → Documentation → Maintenance → Tests → Styles → Reverts

### Step 3: Extract Contributor Information

```bash
# Get unique contributors for the release
git log v1.2.0..HEAD --pretty=format:"%an <%ae>" --no-merges | sort -u
```

Map contributors to their GitHub usernames if possible:

```bash
# Get GitHub username from commit metadata
git log v1.2.0..HEAD --pretty=format:"%an|%(trailers:key=Co-authored-by,valueonly)" --no-merges
```

### Step 4: Generate Changelog Entry

Produce a changelog entry in this format:

```markdown
## [1.3.0](https://github.com/owner/repo/compare/v1.2.0...v1.3.0) (2025-01-15)

### ⚠️ Breaking Changes

- **api:** redesign authentication endpoints ([abc1234](https://github.com/owner/repo/commit/abc1234))

### 🚀 Features

- **auth:** add OAuth2 SSO support ([def5678](https://github.com/owner/repo/commit/def5678))
- **dashboard:** implement real-time notifications ([ghi9012](https://github.com/owner/repo/commit/ghi9012))

### 🐛 Bug Fixes

- **scheduler:** resolve timezone handling issue ([jkl3456](https://github.com/owner/repo/commit/jkl3456))
- resolve null pointer in auth handler ([mno7890](https://github.com/owner/repo/commit/mno7890))

### 📝 Documentation

- update API reference for v1.3.0 ([pqr1234](https://github.com/owner/repo/commit/pqr1234))

### 🔧 Maintenance

- update dependencies ([stu5678](https://github.com/owner/repo/commit/stu5678))

### Contributors

- @username1
- @username2
- @username3
```

### Step 5: Update CHANGELOG.md

Insert the new entry at the top of `CHANGELOG.md`, preserving existing content:

```markdown
# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/),
and this project adheres to [Semantic Versioning](https://semver.org/).

## [Unreleased]

<new entry inserted here>

<previous entries below>
```

#### Update Rules

1. **Clear the `[Unreleased]` section** — move its contents into the new version entry
2. **Preserve all previous entries** — never modify or delete existing entries
3. **Update the comparison links** at the bottom of the file:

```markdown
[Unreleased]: https://github.com/owner/repo/compare/v1.3.0...HEAD
[1.3.0]: https://github.com/owner/repo/compare/v1.2.0...v1.3.0
[1.2.0]: https://github.com/owner/repo/compare/v1.1.0...v1.2.0
```

### Step 6: Create a New CHANGELOG.md (if none exists)

If no `CHANGELOG.md` exists, create one from scratch with all available history:

```bash
# Get all tags in chronological order
git tag --sort=version:refname | grep '^v[0-9]'
```

Generate entries for each version pair, starting from the earliest.

---

## Output

Provide:

1. The full changelog entry (to be included in the GitHub Release)
2. Confirmation that `CHANGELOG.md` was updated
3. Summary statistics:

```markdown
## Changelog Summary

| Metric | Count |
|--------|-------|
| Total commits | 12 |
| Features | 3 |
| Bug Fixes | 5 |
| Breaking Changes | 1 |
| Contributors | 4 |
```

---

## Edge Cases

1. **No CHANGELOG.md exists** → Create one with the header and first entry
2. **No categorizable commits** → Create an entry with "No notable changes" and list raw commits
3. **Scope formatting** → Bold the scope: `**scope:** description`
4. **Multiple breaking changes** → List each one separately under Breaking Changes
5. **Revert commits** → Cross-reference the original commit being reverted
6. **Co-authored commits** → Credit all co-authors as contributors
7. **Very long descriptions** → Truncate at 120 characters with `...` and link to the full commit
