---
name: production-release-documentation
description: Use when users ask to generate production release notes/documentation from a PR number or commit SHA, validate UAT deployment eligibility, determine exact release boundaries, collect rollout/rollback metadata, and produce stakeholder-ready deployment documents. Always use this skill for requests mentioning release scope, UAT-to-PROD validation, deployment summary tables, rollback details, or production release communication.
---

Generate enterprise-ready production deployment documentation from a target PR or commit SHA, with strict release-boundary logic and GitHub-backed evidence.

## When to Use

- Generate production release documentation from a PR or commit
- Validate UAT deployment eligibility before PROD release
- Determine release scope from last PROD to target PR/commit
- Build deployment/rollback summaries for stakeholders

## Required Inputs

| Field | Required | Description |
|---|---|---|
| `target` | Yes | PR number or commit SHA |
| `repo` | No | `owner/repo` (default: infer from `git remote get-url origin`) |
| `environment_labels` | No | Default: `uat`, `prod`, `production` |
| `release_tag_pattern` | No | Default: `v*` |

## Data Collection Workflow

### 1. Normalize Target

**PR target:**
```bash
gh pr view <PR> --json number,title,url,author,mergeCommit,mergedAt,baseRefName,headRefName,state
```
Set `target_commit` = `mergeCommit.oid`.

**Commit SHA target:**
```bash
gh api repos/<owner>/<repo>/commits/<sha>
gh api repos/<owner>/<repo>/commits/<sha>/pulls -H "Accept: application/vnd.github+json"
```

### 2. Identify UAT & PROD Deployments

```bash
gh api repos/<owner>/<repo>/deployments -f environment=uat -f per_page=100
gh api repos/<owner>/<repo>/deployments -f environment=production -f per_page=100
# For each: fetch statuses, keep only terminal "success" state
gh api repos/<owner>/<repo>/deployments/<id>/statuses
```

Capture: latest successful UAT and PROD deployments with refs, SHAs, timestamps, creator, URLs.

### 3. Validate Deployment Eligibility

`target_commit` must be reachable from latest successful UAT SHA:

```bash
git merge-base --is-ancestor <target_commit> <uat_deployed_sha>
```

**If check fails → stop.** Return: `Provided PR/Commit is not deployed till UAT` + latest UAT release number/link.

### 4. Determine Release Boundary

Range: `last_prod_success_sha..target_commit` (exclusive start, inclusive end). Never include commits beyond `target_commit`.

```bash
git log --reverse --pretty=format:"%H%x09%s" <prod_sha>..<target_commit>
```

Map each commit to its PR (deduplicate by PR number):
```bash
gh api repos/<owner>/<repo>/commits/<sha>/pulls -H "Accept: application/vnd.github+json"
```

### 5. Collect Metadata

Gather: release number, build/run link, deployment window, environment, release type (standard/hotfix/emergency), release owner, approvals.

```bash
gh release list --limit 50
gh run list --limit 50 --json databaseId,name,headSha,status,conclusion,createdAt,url,workflowName
```

### 6. Build Rollback Details

Rollback target = last successful PROD deployment before this rollout.

Rollback runbook: 1) Re-run last good workflow → 2) Re-deploy immutable artifacts → 3) Smoke test → 4) Announce completion.

## Required Output Tables

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

### B. Included PRs/Commits

| PR/Commit | Title | Author | Type | Status | UAT Release | Link |
|---|---|---|---|---|---|---|

### C. Deployment Scope Analysis

| Service | Change Type | Impact Level | Breaking | DB Change | Config Change |
|---|---|---|---|---|---|

### D. Rollback Details

| Rollback Release | Build Number | Deployment Date | Validation Status |
|---|---|---|---|

### E. Validation Checklist

| Item | Status |
|---|---|
| UAT Validation | |
| Smoke Test | |
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

- **Type:** infer from title prefix (`feat`, `fix`, `chore`, etc.), else `other`
- **Impact:** `High` (breaking/db/config), `Medium` (service logic), `Low` (docs/chore)
- **Breaking:** `Yes` if `!` or `BREAKING CHANGE` in title/body/commit
- **DB Change:** `Yes` if files match `migrations/`, `db/`, `schema`
- **Config Change:** `Yes` if touching config/workflow/env files

## Safety Rules

- Never include PRs/commits after `target_commit`
- Always anchor to `last_prod_success_sha..target_commit`
- Mark missing metadata as `TBD` — never invent values
- Prefer GitHub API evidence over assumptions
