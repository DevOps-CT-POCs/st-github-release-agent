---
name: gh-cli
description: Use when users need to interact with GitHub directly from the command line using the `gh` CLI — creating releases, managing PRs and issues, checking workflow runs, uploading release assets, managing secrets and variables, querying the GitHub API, or any GitHub operation that can be performed via `gh` without opening a browser.
---

Provide accurate, up-to-date guidance on using the GitHub CLI (`gh`) for repository operations, release management, CI/CD interaction, and GitHub API access — all from the terminal.

## When to Use

Use this skill when the request is about:

- Creating, editing, listing, viewing, or downloading GitHub Releases (`gh release`)
- Managing pull requests — creating, reviewing, merging, checking CI status (`gh pr`)
- Managing issues — creating, listing, closing, labeling (`gh issue`)
- Monitoring or re-running GitHub Actions workflow runs (`gh run`, `gh workflow`)
- Managing repository secrets and variables (`gh secret`, `gh variable`)
- Making direct GitHub API calls (`gh api`)
- Repository operations — cloning, forking, creating, viewing (`gh repo`)
- Authentication and credential management (`gh auth`)
- Searching across GitHub — code, commits, issues, PRs, repos (`gh search`)
- Managing labels, rulesets, caches, attestations, or SSH/GPG keys
- Any task where the user wants to avoid the GitHub web UI

Do not use this skill for:

- Scaffolding or writing GitHub Actions workflow YAML files. Use `github-actions-release` or `github-actions-docs`.
- Calculating semantic versions from commit history. Use `semantic-versioning`.
- Generating changelogs from commits. Use `changelog-generation`.
- Configuring Sigstore/Cosign signing or SBOM generation. Use `artifact-signing-sbom`.
- Emergency hotfix branching and release workflows. Use `hotfix-release`.

## Prerequisites

The `gh` CLI must be installed and authenticated:

```bash
# Check installation
gh --version

# Check authentication status
gh auth status

# Login if needed (interactive)
gh auth login

# Login with a token (non-interactive, e.g. in CI)
echo "$GH_TOKEN" | gh auth login --with-token
```

## Command Reference

### Releases (`gh release`)

The most critical command group for release engineering.

```bash
# List releases
gh release list --limit 10

# View a specific release
gh release view v1.2.3

# Create a release from an existing tag
gh release create v1.2.3 --title "v1.2.3" --notes "Release notes here"

# Create a release with auto-generated notes from commits
gh release create v1.2.3 --generate-notes

# Create a release with notes from a file (e.g., CHANGELOG section)
gh release create v1.2.3 --notes-file RELEASE_NOTES.md

# Create a draft release
gh release create v1.2.3 --draft --generate-notes

# Create a pre-release
gh release create v1.2.3-rc.1 --prerelease --generate-notes

# Upload assets to an existing release
gh release upload v1.2.3 ./dist/*.tar.gz ./dist/*.zip

# Download release assets
gh release download v1.2.3 --dir ./downloads

# Edit an existing release
gh release edit v1.2.3 --draft=false --prerelease=false

# Delete a release (does NOT delete the Git tag)
gh release delete v1.2.3 --yes

# Verify a release asset signature
gh release verify-asset ./artifact.tar.gz --release v1.2.3
```

### Pull Requests (`gh pr`)

```bash
# Create a PR
gh pr create --title "feat: add OAuth2 support" --body "Description here" --base main

# Create a PR and auto-fill title/body from commits
gh pr create --fill

# List open PRs
gh pr list

# View PR details
gh pr view 42

# Check CI status on a PR
gh pr checks 42

# Merge a PR (squash)
gh pr merge 42 --squash --delete-branch

# Review a PR
gh pr review 42 --approve
gh pr review 42 --request-changes --body "Please fix X"

# Mark a draft PR as ready
gh pr ready 42
```

### Issues (`gh issue`)

```bash
# Create an issue
gh issue create --title "Bug: login fails" --body "Steps to reproduce..." --label bug

# List issues
gh issue list --label "bug" --state open

# Close an issue
gh issue close 42 --reason completed

# View issue details
gh issue view 42
```

### Workflow Runs (`gh run`, `gh workflow`)

```bash
# List recent workflow runs
gh run list --limit 10

# View a specific run
gh run view 12345

# Watch a run in progress (live updates)
gh run watch 12345

# Re-run a failed run
gh run rerun 12345

# Re-run only failed jobs
gh run rerun 12345 --failed

# Download run artifacts
gh run download 12345

# List workflows
gh workflow list

# Manually trigger a workflow
gh workflow run release.yml --ref main

# Trigger with inputs
gh workflow run release.yml --ref main -f version=1.2.3 -f dry-run=true

# Disable/enable a workflow
gh workflow disable release.yml
gh workflow enable release.yml
```

### Secrets & Variables (`gh secret`, `gh variable`)

```bash
# Set a repository secret
gh secret set NPM_TOKEN --body "$NPM_TOKEN"

# Set a secret from a file
gh secret set SIGNING_KEY < ./signing-key.pem

# Set an environment secret
gh secret set AWS_ACCESS_KEY --env production --body "$KEY"

# List secrets
gh secret list

# Delete a secret
gh secret delete NPM_TOKEN

# Set a repository variable
gh variable set DEPLOY_ENV --body "staging"

# List variables
gh variable list

# Get a variable value
gh variable get DEPLOY_ENV
```

### API (`gh api`)

For anything not covered by built-in commands:

```bash
# GET request
gh api repos/{owner}/{repo}/releases/latest

# POST request with JSON body
gh api repos/{owner}/{repo}/releases \
  -f tag_name="v1.2.3" \
  -f name="v1.2.3" \
  -f body="Release notes" \
  -F draft=false \
  -F prerelease=false

# Use JQ expressions to filter output
gh api repos/{owner}/{repo}/releases --jq '.[0].tag_name'

# Paginate through results
gh api repos/{owner}/{repo}/issues --paginate --jq '.[].title'

# GraphQL query
gh api graphql -f query='
  query {
    repository(owner: "cli", name: "cli") {
      releases(last: 5) {
        nodes { tagName publishedAt }
      }
    }
  }
'

# Use --hostname for GitHub Enterprise
gh api --hostname github.example.com repos/{owner}/{repo}
```

### Repository (`gh repo`)

```bash
# View repo info
gh repo view

# Clone a repo
gh repo clone owner/repo

# Create a new repo
gh repo create my-project --public --clone

# Fork a repo
gh repo fork owner/repo --clone

# Sync a fork with upstream
gh repo sync owner/fork

# Set the default repo for gh commands
gh repo set-default owner/repo
```

### Attestations (`gh attestation`)

```bash
# Verify an artifact's attestation
gh attestation verify ./artifact.tar.gz --repo owner/repo

# Download attestations
gh attestation download ./artifact.tar.gz --repo owner/repo
```

## Workflow

### 1. Identify the operation

Map the user's request to the appropriate `gh` command group:

| User Intent | Command Group |
|-------------|---------------|
| Create/manage releases | `gh release` |
| Create/merge PRs | `gh pr` |
| File/track issues | `gh issue` |
| Check CI/run workflows | `gh run` / `gh workflow` |
| Manage secrets | `gh secret` |
| Manage variables | `gh variable` |
| Direct API calls | `gh api` |
| Repo operations | `gh repo` |
| Verify artifacts | `gh attestation` |

### 2. Construct the command

- Use `--json` flag with `--jq` for machine-readable output when piping or scripting
- Use `--repo owner/repo` or `-R owner/repo` to target a specific repo without being in its directory
- Prefer `--yes` or `-y` to skip confirmation prompts in scripts
- Use `--help` on any command to see all available flags

### 3. Handle output formatting

```bash
# JSON output with specific fields
gh release list --json tagName,publishedAt,isDraft --jq '.[] | select(.isDraft == false)'

# Table output (default for humans)
gh pr list --state open

# Structured JSON for scripting
gh pr list --json number,title,author --jq '.[] | "\(.number)\t\(.title)\t\(.author.login)"'
```

## Answer Shape

1. **Exact command(s)** — ready to copy-paste
2. **Flags explained** — note any non-obvious flags
3. **Scripting tips** — if the user is automating, show `--json` + `--jq` patterns
4. **Alternative approaches** — mention `gh api` if the built-in command is limited

## Integration with Other Skills

The `gh` CLI is the execution layer that other skills build on:

| Skill | How `gh` CLI Helps |
|-------|--------------------|
| `semantic-versioning` | After calculating the version, use `gh release create` to publish |
| `changelog-generation` | Pipe generated changelog into `gh release create --notes-file` |
| `github-actions-release` | Use `gh workflow run` to trigger release workflows, `gh run watch` to monitor |
| `artifact-signing-sbom` | Use `gh release upload` to attach signed artifacts, `gh attestation verify` to validate |
| `hotfix-release` | Use `gh pr create` for hotfix PRs, `gh pr merge` for fast-tracking |

## Common Mistakes

- Forgetting `--repo owner/repo` when running outside the repository directory
- Using `gh release delete` and expecting it to also delete the Git tag (it does not — use `git push --delete origin v1.2.3` separately)
- Not quoting JSON values in `gh api -f` flags, causing shell interpolation issues
- Using `--json` without `--jq`, which dumps raw JSON instead of filtered output
- Confusing `gh run` (workflow runs) with `gh workflow` (workflow definitions)
- Setting secrets with `--body` containing shell-special characters without proper quoting

## Environment Variables

Key environment variables that affect `gh` behavior:

| Variable | Purpose |
|----------|---------|
| `GH_TOKEN` | Authentication token (overrides `gh auth login`) |
| `GH_HOST` | Default GitHub hostname (for GitHub Enterprise) |
| `GH_REPO` | Default repository in `owner/repo` format |
| `GH_NO_UPDATE_NOTIFIER` | Set to `1` to suppress update notices |
| `GITHUB_TOKEN` | Fallback token (also used by GitHub Actions) |
| `NO_COLOR` | Set to `1` to disable colored output |

## Official Documentation

- Manual: <https://cli.github.com/manual/>
- Source: <https://github.com/cli/cli>
- Release notes: <https://github.com/cli/cli/releases/latest>
