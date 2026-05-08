---
name: production-release-documentation
description: Use when users ask to generate production release notes/documentation from a PR number or commit SHA, validate UAT deployment eligibility, determine exact release boundaries, collect rollout/rollback metadata, and produce stakeholder-ready deployment documents. Always use this skill for requests mentioning release scope, UAT-to-PROD validation, deployment summary tables, rollback details, or production release communication.
---

Generate enterprise-ready production deployment documentation from a target PR number or commit SHA, with strict release-boundary logic and GitHub-backed evidence.

## When to Use

Use this skill when the user asks to:

- Generate production release documentation from a PR or commit
- Validate whether a change is deployed to UAT before PROD release
- Produce release scope from last PROD deployment up to a target PR/commit
- Build deployment/rollback summaries for stakeholder communication
- Include workflow/build/run links, approvals, risk table, and timeline

Do not use this skill for:

- Calculating semantic version bumps from commit messages alone (`semantic-versioning`)
- Writing generic changelogs not tied to deployment boundaries (`changelog-generation`)
- Building release workflows from scratch (`github-actions-release`)

## Required Inputs

| Field | Required | Description |
|---|---|---|
| `target` | Yes | PR number (e.g. `1234`) or commit SHA |
| `repo` | No | `owner/repo` if not inferable from local git remotes |
| `environment_labels` | No | Names used in deployments (default: `uat`, `prod`, `production`) |
| `release_tag_pattern` | No | Tag convention (default: `v*`) |

If `repo` is missing, resolve from `git remote get-url origin`.

## Data Collection Workflow

### 1) Normalize target (PR or commit)

If target is numeric, treat as PR:

```bash
gh pr view <PR_NUMBER> --json number,title,url,author,mergeCommit,mergedAt,baseRefName,headRefName,state
```

Set `target_commit` to `mergeCommit.oid` (or HEAD commit of PR if mergeCommit absent for special merge methods).

If target is commit SHA:

```bash
gh api repos/<owner>/<repo>/commits/<sha>
```

Resolve associated PR (if any):

```bash
gh api repos/<owner>/<repo>/commits/<sha>/pulls -H "Accept: application/vnd.github+json"
```

### 2) Identify latest deployments for UAT and PROD

Fetch deployments by environment (newest first):

```bash
gh api repos/<owner>/<repo>/deployments -f environment=uat -f per_page=100
gh api repos/<owner>/<repo>/deployments -f environment=production -f per_page=100
```

For each deployment, fetch statuses and keep only successful terminal state (`success`):

```bash
gh api repos/<owner>/<repo>/deployments/<deployment_id>/statuses
```

Capture:

- UAT latest successful deployment (`uat_deployment`)
- PROD latest successful deployment (`prod_deployment`)
- Their refs, SHAs, timestamps, creator, and URLs

### 3) Validate deployment eligibility

Eligibility rule: `target_commit` must be reachable from latest successful UAT deployed SHA/ref.

Use git ancestry check:

```bash
git merge-base --is-ancestor <target_commit> <uat_deployed_sha>
```

If check fails, stop and return exactly:

`Provided PR/Commit is not deployed till UAT`

Also include latest UAT release number/tag and link.

### 4) Determine release boundary

Boundary start: last successful PROD deployed commit (`prod_deployed_sha`).
Boundary end: `target_commit`.

Collect commits in range (inclusive end, exclusive start):

```bash
git log --reverse --pretty=format:"%H%x09%s" <prod_deployed_sha>..<target_commit>
```

Guardrail: do not include any commit beyond `target_commit`.

For each commit in range, map PR if present:

```bash
gh api repos/<owner>/<repo>/commits/<sha>/pulls -H "Accept: application/vnd.github+json"
```

Deduplicate by PR number when multiple commits belong to same PR. Keep direct commits with no PR as commit rows.

### 5) Collect release metadata

Gather:

- Upcoming production release number (infer from release tags or naming convention)
- Release link (planned URL if not created yet)
- Build/run number and workflow link from the pipeline that will deploy `target_commit`
- Deployment window/date-time (planned or latest schedule input)
- Environment (`production`)
- Release type (`standard`, `hotfix`, `emergency`) from input/context
- Release owner
- Approval details (from PR reviews, environment protection approvals, or supplied data)

Useful commands:

```bash
gh release list --limit 50
gh run list --limit 50 --json databaseId,name,headSha,status,conclusion,createdAt,updatedAt,url,workflowName
gh pr view <PR_NUMBER> --json reviews,author,title,url
gh api repos/<owner>/<repo>/actions/runs?per_page=100
```

### 6) Build rollback details

Rollback target is latest successful PROD release/deployment before target rollout.

Include:

- Last PROD release number
- Rollback release/build link
- Rollback build number
- Short rollback runbook summary

Rollback summary template:

1. Re-run last known good production workflow for rollback tag/SHA.
2. Re-deploy immutable artifacts for that release.
3. Run smoke tests and validate critical paths.
4. Announce rollback completion and incident status.

## Output Contract

Always produce:

1. Executive summary (2-5 bullets)
2. Validation result (pass/fail)
3. Full deployment document using all required tables
4. GitHub links for traceability

If validation fails, return only:

- Failure message
- Latest UAT release number/link
- Optional one-line next action

## Required Table Sections

### A. Release Summary

| Field | Value |
|---|---|
| Upcoming Release | |
| Build Number | |
| Environment | |
| Deployment Window | |
| Release Owner | |
| Last PROD Release | |
| Rollback Release | |
| UAT Validation Status | |
| Change Count | |

### B. Included PRs / Commits

| PR/Commit | Title | Author | Type | Status | UAT Release | Link |
|---|---|---|---|---|---|---|

### C. Deployment Scope Analysis

| Service | Change Type | Impact Level | Breaking Change | DB Change | Config Change |
|---|---|---|---|---|---|

### D. Rollback Details

| Rollback Release | Build Number | Deployment Date | Validation Status |
|---|---|---|---|

### E. Validation Checklist

| Validation Item | Status |
|---|---|
| UAT Validation Completed | |
| Smoke Test Completed | |
| Security Validation | |
| DB Migration Verified | |
| Rollback Tested | |

### F. Risks and Known Issues

| Type | Description | Mitigation |
|---|---|---|

### G. Deployment Timeline

| Stage | Status | Time | Owner |
|---|---|---|---|

## Classification Rules

- `Type` in Included PRs/Commits: infer from title prefixes (`feat`, `fix`, `chore`, etc.), else `other`
- `Impact Level`: `High` for breaking/db/config changes, `Medium` for service logic changes, `Low` for docs/chore-only
- `Breaking Change`: mark `Yes` if PR title/body/commit includes `!` or `BREAKING CHANGE`
- `DB Change`: mark `Yes` if files include migration patterns (`migrations/`, `db/`, `schema`)
- `Config Change`: mark `Yes` if touching config files/workflow/env definitions

## Safety and Boundary Rules

- Never include PRs/commits merged after `target_commit`
- Always anchor scope to `last_prod_success_sha..target_commit`
- If deployment metadata is missing, explicitly mark fields as `TBD` instead of inventing values
- Prefer GitHub API/gh evidence; avoid assumptions

## Minimal Execution Checklist

1. Resolve target to commit SHA
2. Resolve latest successful UAT and PROD deployments
3. Validate target is in UAT lineage
4. Build exact commit/PR range from last PROD to target
5. Collect build/workflow/approval/owner metadata
6. Compile rollback data
7. Output executive summary + required tables

## Example Failure Response

`Provided PR/Commit is not deployed till UAT`

| Field | Value |
|---|---|
| Latest UAT Release | v2.14.0 |
| Latest UAT Release Link | https://github.com/<owner>/<repo>/releases/tag/v2.14.0 |

Next action: deploy target PR/commit to UAT, then regenerate release documentation.
