---
name: gh-cli
description: Use when users need to interact with GitHub directly from the command line using the `gh` CLI — creating releases, managing PRs and issues, checking workflow runs, uploading release assets, managing secrets and variables, querying the GitHub API, or any GitHub operation that can be performed via `gh` without opening a browser.
---

Provide guidance on using the GitHub CLI (`gh`) for repository operations, release management, CI/CD interaction, and GitHub API access.

## When to Use

- Creating, editing, listing, or downloading GitHub Releases
- Managing PRs, issues, workflow runs, secrets, variables
- Direct GitHub API calls (`gh api`)
- Repo operations, authentication, search, attestation verification

## Prerequisites

```bash
gh --version        # Check installation
gh auth status      # Check auth
gh auth login       # Interactive login
echo "$GH_TOKEN" | gh auth login --with-token  # Non-interactive (CI)
```

## Command Reference

### Releases (`gh release`) — Primary for release engineering

```bash
gh release list --limit 10
gh release view v1.2.3
gh release create v1.2.3 --title "v1.2.3" --notes "Release notes"
gh release create v1.2.3 --generate-notes              # Auto-generate from commits
gh release create v1.2.3 --notes-file RELEASE_NOTES.md  # From file
gh release create v1.2.3 --draft --generate-notes       # Draft release
gh release create v1.2.3-rc.1 --prerelease --generate-notes
gh release upload v1.2.3 ./dist/*.tar.gz ./dist/*.zip
gh release download v1.2.3 --dir ./downloads
gh release edit v1.2.3 --draft=false --prerelease=false
gh release delete v1.2.3 --yes  # Does NOT delete the Git tag
```

### Workflow Runs (`gh run`, `gh workflow`)

```bash
gh run list --limit 10
gh run view 12345
gh run watch 12345              # Live updates
gh run rerun 12345 [--failed]   # Re-run all or only failed jobs
gh run download 12345
gh workflow list
gh workflow run release.yml --ref main [-f version=1.2.3 -f dry-run=true]
```

### Pull Requests (`gh pr`)

```bash
gh pr create --title "feat: ..." --body "..." --base main
gh pr create --fill                    # Auto-fill from commits
gh pr list / gh pr view 42
gh pr checks 42                        # CI status
gh pr merge 42 --squash --delete-branch
gh pr review 42 --approve
gh pr ready 42                         # Mark draft as ready
```

### Issues (`gh issue`)

```bash
gh issue create --title "Bug: ..." --body "..." --label bug
gh issue list --label "bug" --state open
gh issue close 42 --reason completed
```

### Secrets & Variables

```bash
gh secret set NPM_TOKEN --body "$NPM_TOKEN"
gh secret set SIGNING_KEY < ./key.pem
gh secret set AWS_KEY --env production --body "$KEY"
gh secret list / gh secret delete NPM_TOKEN
gh variable set DEPLOY_ENV --body "staging"
gh variable list / gh variable get DEPLOY_ENV
```

### API (`gh api`)

```bash
gh api repos/{owner}/{repo}/releases/latest
gh api repos/{owner}/{repo}/releases -f tag_name="v1.2.3" -f name="v1.2.3" -f body="Notes"
gh api repos/{owner}/{repo}/releases --jq '.[0].tag_name'   # JQ filtering
gh api repos/{owner}/{repo}/issues --paginate --jq '.[].title'
gh api graphql -f query='{ repository(owner:"o",name:"r") { releases(last:5) { nodes { tagName } } } }'
```

### Attestations

```bash
gh attestation verify ./artifact.tar.gz --repo owner/repo
gh attestation download ./artifact.tar.gz --repo owner/repo
```

## Usage Tips

- `--json <fields> --jq '<expr>'` for machine-readable output
- `-R owner/repo` to target a specific repo without being in its directory
- `--yes` / `-y` to skip confirmation prompts in scripts

## Integration with Other Skills

| Skill | `gh` CLI Usage |
|-------|----------------|
| `semantic-versioning` | `gh release create` to publish |
| `changelog-generation` | `gh release create --notes-file` |
| `github-actions-release` | `gh workflow run` to trigger, `gh run watch` to monitor |
| `artifact-signing-sbom` | `gh release upload` for signed artifacts, `gh attestation verify` |
| `hotfix-release` | `gh pr create/merge` for hotfix PRs |

## Environment Variables

| Variable | Purpose |
|----------|---------|
| `GH_TOKEN` / `GITHUB_TOKEN` | Authentication token |
| `GH_HOST` | GitHub Enterprise hostname |
| `GH_REPO` | Default `owner/repo` |

## Common Mistakes

- `gh release delete` does NOT delete the Git tag — use `git push --delete origin <tag>` separately
- Use `--json` with `--jq` (not without) to get filtered output
- Don't confuse `gh run` (runs) with `gh workflow` (definitions)
- Quote JSON values in `gh api -f` to avoid shell interpolation
