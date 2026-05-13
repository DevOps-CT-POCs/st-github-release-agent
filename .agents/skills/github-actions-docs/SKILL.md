---
name: github-actions-docs
description: Use when users ask how to write, explain, customize, migrate, secure, or troubleshoot GitHub Actions workflows, workflow syntax, triggers, matrices, runners, reusable workflows, artifacts, caching, secrets, OIDC, deployments, custom actions, or Actions Runner Controller, especially when they need official GitHub documentation, exact links, or docs-grounded YAML guidance.
---

Ground GitHub Actions answers in official documentation. Return the closest authoritative page instead of generic CI/CD advice.

## When to Use

- GitHub Actions concepts, syntax, triggers, jobs, matrices, concurrency, variables, contexts, expressions
- Runners (GitHub-hosted, larger, self-hosted, Actions Runner Controller)
- Artifacts, caches, reusable workflows, workflow templates, custom actions
- Secrets, `GITHUB_TOKEN`, OIDC, artifact attestations, secure workflow patterns
- Environments, deployment protection rules, deployment history
- Migration from Jenkins, CircleCI, GitLab CI/CD, Travis CI, Azure Pipelines
- Troubleshooting workflow behavior needing docs references

## Workflow

### 1. Classify the request

Bucket: Getting started · Workflow syntax · Runners · Security/supply chain · Deployments · Custom actions · Monitoring/troubleshooting · Migration.

Load `references/topic-map.md` as a compact index of entry points.

### 2. Search `docs.github.com` first

- Source of truth: `https://docs.github.com/en/actions`
- Search with user's exact terms + focused Actions phrase
- Compare 2-3 candidate pages; pick the most direct answer

### 3. Read the best page before answering

Use the topic map to narrow, then verify against the live doc. If a page appears moved or incomplete, say so and return the nearest authoritative pages.

### 4. Answer with docs-grounded guidance

- Direct answer in plain language with exact docs links
- YAML examples only when asked or when the docs page necessitates them
- Make inferences explicit: "According to GitHub docs, ..." / "Inference: ..."

## Answer Shape

1. Direct answer → 2. Relevant doc links → 3. Example YAML (if needed) → 4. Inference callout (if connecting multiple docs)

Keep citations close to claims they support.

## Search Tips

- Concepts → overview pages first; Syntax → reference pages; Security → OIDC/secrets/GITHUB_TOKEN docs
- Deployments → environments docs before cloud-specific; Migration → hub page then platform-specific
- Beginners → tutorials/quickstarts over raw references

## Bundled Reference

Read `references/topic-map.md` as a compact routing index. It is intentionally selective — always verify against live GitHub docs.
