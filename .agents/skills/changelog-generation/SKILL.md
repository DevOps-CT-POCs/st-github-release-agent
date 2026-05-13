---
name: changelog-generation
description: Use when users ask to generate a changelog, update release notes, or understand what changed between versions. Parses Conventional Commits into a structured, human-readable CHANGELOG.md following the Keep a Changelog format, with contributor attribution and commit links.
---

Parse Conventional Commits into a structured `CHANGELOG.md` following the Keep a Changelog format.

## When to Use

- Generating a changelog or release notes from commit history
- Updating or creating `CHANGELOG.md` for a new release
- Understanding what changed between two versions/tags

**Prerequisite:** Run `semantic-versioning` first to determine the version number.

## Workflow

### 1. Gather Inputs

| Input | Required | Description |
|-------|----------|-------------|
| New version | Yes | Version string from `semantic-versioning` |
| Previous tag | Yes | Git tag to compare from |
| Repository URL | Yes | GitHub URL for commit/compare links |
| Include contributors | No | Default: `true` |

### 2. Collect & Categorize Commits

```bash
git log <prev_tag>..HEAD --pretty=format:"%H|%an|%ae|%s" --no-merges
```

**Section ordering** (include only non-empty sections):

| Type | Section | Emoji |
|------|---------|-------|
| `BREAKING CHANGE` / `!` | Breaking Changes | ⚠️ |
| `feat` | Features | 🚀 |
| `fix` | Bug Fixes | 🐛 |
| `perf` | Performance | ⚡ |
| `docs` | Documentation | 📝 |
| `chore`, `ci`, `build`, `refactor` | Maintenance | 🔧 |
| `test` | Tests | 🧪 |
| `style` | Styles | 💄 |
| `revert` | Reverts | ⏪ |

### 3. Extract Contributors

```bash
git log <prev_tag>..HEAD --pretty=format:"%an <%ae>" --no-merges | sort -u
```

Map to GitHub usernames when possible.

### 4. Generate Entry

Format: `## [VERSION](compare_url) (DATE)` with categorized commits as `- **scope:** description ([hash](commit_url))`, followed by `### Contributors` section.

### 5. Update CHANGELOG.md

- Insert new entry after `## [Unreleased]` header, clearing Unreleased contents into the new version
- Preserve all previous entries — never modify or delete
- Update comparison links at file bottom:
  ```
  [Unreleased]: https://github.com/owner/repo/compare/vNEW...HEAD
  [NEW]: https://github.com/owner/repo/compare/vPREV...vNEW
  ```
- If no `CHANGELOG.md` exists, create one with the standard header and first entry

## Answer Shape

Full changelog entry, confirmation of file update, and summary table with: total commits, features, bug fixes, breaking changes, and contributor count.

## Edge Cases

- **No categorizable commits** → entry with "No notable changes" + raw commit list
- **Scope formatting** → bold scope: `**scope:** description`
- **Revert commits** → cross-reference original commit
- **Co-authored commits** → credit all co-authors
- **Long descriptions** → truncate at 120 chars with link to full commit
