# 🚀 Release Agent

An AI-powered release engineering agent that automates the entire GitHub release lifecycle — semantic versioning, changelog generation, artifact signing, SBOM creation, multi-environment deployments, and emergency hotfix management.

Drop this agent into your repository and let it handle your releases.

---

## ✨ Features

| Feature | Description |
|---------|-------------|
| **Semantic Versioning** | Automatically determines the next version from Conventional Commits |
| **Changelog Generation** | Creates structured changelogs following Keep a Changelog format |
| **GitHub Actions Workflows** | Scaffolds production-ready CI/CD release pipelines |
| **Artifact Signing** | Sigstore/Cosign keyless signing with OIDC — no keys to manage |
| **SBOM Generation** | Software Bill of Materials via Syft or CycloneDX |
| **SLSA Provenance** | Supply chain attestation (Level 2 or 3) |
| **Hotfix Management** | Emergency patch workflow with expedited CI and post-mortem templates |
| **Multi-Environment Deploy** | Staging → Production with manual approval gates |
| **Stack Agnostic** | Works with Node.js, Go, Python, Rust, Java, and more |

---

## 📦 Quick Start

### 1. Clone into your project

```bash
# Clone the agent into your existing repo
git clone https://github.com/your-org/release-agent.git /tmp/release-agent

# Copy the agent files
cp -r /tmp/release-agent/AGENTS.md ./
cp -r /tmp/release-agent/.agents ./
cp -r /tmp/release-agent/.github ./

# Clean up
rm -rf /tmp/release-agent
```

Or copy the files manually:

```
Your Project/
├── AGENTS.md                          ← Copy this
├── .agents/
│   └── skills/                        ← Copy this entire directory
│       ├── semantic-versioning/
│       ├── changelog-generation/
│       ├── github-actions-release/
│       ├── artifact-signing-sbom/
│       └── hotfix-release/
└── .github/
    └── release.yml                    ← Copy this
```

### 2. Start using the agent

Open your project in your IDE and ask the agent:

> "Create a release for my project"

The agent will:
1. Analyze your commit history since the last tag
2. Calculate the correct semantic version
3. Generate/update `CHANGELOG.md`
4. Create an annotated Git tag
5. Scaffold a GitHub Actions release workflow (if needed)

---

## 🔌 IDE & Tool Integration

### Antigravity

The agent works natively with Antigravity. Skills are auto-discovered from `.agents/skills/`.

**Project-level** (recommended):
```
your-project/.agents/skills/    ← Agent skills here
your-project/AGENTS.md          ← Agent identity here
```

**User-level** (shared across projects):
```
~/.agents/skills/    ← Copy skills here for global access
```

Open your project in Antigravity and start prompting — the agent context is loaded automatically.

---

### VS Code (GitHub Copilot)

Copilot reads `AGENTS.md` at the repo root automatically. For skill integration:

**Option A: Copilot Instructions**

Copy key skill content into `.github/copilot-instructions.md`:

```markdown
# Release Agent Instructions

When asked about releases, versioning, or deployment, follow these guidelines:
- Use Conventional Commits for version determination
- Follow Semantic Versioning 2.0.0
- Generate changelogs in Keep a Changelog format
<!-- Paste relevant skill content here -->
```

**Option B: Custom Agent Files** (Copilot Chat Agents)

Create `.github/.copilot/agents/release.md` with the agent definition.

---

### OpenCode (CLI)

[OpenCode](https://github.com/opencode-ai/opencode) reads `AGENTS.md` natively.

```bash
# Navigate to your project
cd your-project

# Start OpenCode — it auto-discovers AGENTS.md
opencode

# Or initialize for first-time setup
opencode /init
```

Then ask:
```
> Create a new release
> What's the next version?
> Set up artifact signing
```

---

### Cursor / Windsurf

These editors support `.cursorrules` or similar config files. Copy the content from `AGENTS.md` into your editor's agent/rules configuration:

```
your-project/.cursorrules        ← Paste AGENTS.md content
your-project/.windsurfrules      ← Paste AGENTS.md content
```

---

## 🛠️ Skills Reference

| Skill | Path | Trigger Phrases |
|-------|------|----------------|
| **Semantic Versioning** | `.agents/skills/semantic-versioning/SKILL.md` | "bump version", "next version", "create release" |
| **Changelog Generation** | `.agents/skills/changelog-generation/SKILL.md` | "generate changelog", "what changed", "release notes" |
| **GitHub Actions Release** | `.agents/skills/github-actions-release/SKILL.md` | "set up CI/CD", "create workflow", "automate releases" |
| **Artifact Signing & SBOM** | `.agents/skills/artifact-signing-sbom/SKILL.md` | "sign artifacts", "add SBOM", "supply chain security" |
| **Hotfix Release** | `.agents/skills/hotfix-release/SKILL.md` | "hotfix", "emergency patch", "critical bug in production" |

---

## ⚙️ Customization

### Adapting for Your Tech Stack

The agent defaults to Node.js examples but adapts to any stack. Tell the agent your tech stack:

> "I'm using Go — set up releases for my Go project"

It will adjust:
- Version file detection (`go.mod` vs `package.json` vs `Cargo.toml`)
- Build commands (`go build` vs `npm run build` vs `cargo build --release`)
- SBOM generation tool (CycloneDX-gomod vs Syft)
- Artifact types (binaries vs packages vs containers)

### Adding Custom Skills

Create a new skill in `.agents/skills/your-skill-name/SKILL.md` following this template:

```markdown
---
name: your-skill-name
description: Use when users ask about <trigger phrases>. Describe what this skill does.
---

Description of what this skill does.

## When to Use

Use this skill when the request is about:
- Trigger conditions

Do not use this skill for:
- Other skills that handle adjacent concerns

## Workflow

### 1. Step name
Step-by-step instructions for the agent.

## Answer Shape
What the skill produces.

## Edge Cases
1. Edge case handling

## Common Mistakes
- Mistakes to avoid
```

Then reference it in `AGENTS.md` under the Skills table and decision tree.

---

## 🔒 Security Model

| Principle | Implementation |
|-----------|---------------|
| **No Long-Lived Secrets** | OIDC keyless signing and cloud authentication |
| **Least Privilege** | Minimal `permissions:` blocks in all workflows |
| **Immutable Releases** | Tags are never deleted; artifacts are never modified |
| **Supply Chain Security** | Sigstore signing, SBOM, SLSA provenance attestation |
| **Action Pinning** | GitHub Actions pinned to full SHA hashes |
| **Branch Protection** | No force-push to main; PRs required for all changes |
| **Environment Gates** | Manual approval + wait timers for production deploys |
| **Audit Trail** | All releases tagged, changelogged, and attested |

---

## 📁 Repository Structure

```
.
├── AGENTS.md                                    # Agent identity & orchestration
├── README.md                                    # This file
├── .agents/
│   └── skills/
│       ├── github-actions-docs/
│       │   ├── SKILL.md                         # GitHub Actions documentation
│       │   └── references/
│       │       └── topic-map.md                 # Docs topic index
│       ├── semantic-versioning/
│       │   └── SKILL.md                         # Version calculation & tagging
│       ├── changelog-generation/
│       │   └── SKILL.md                         # CHANGELOG.md generation
│       ├── github-actions-release/
│       │   ├── SKILL.md                         # CI/CD workflow scaffolding
│       │   └── references/
│       │       └── workflow-templates.md         # Reusable YAML snippets
│       ├── artifact-signing-sbom/
│       │   └── SKILL.md                         # Sigstore + SBOM + SLSA
│       └── hotfix-release/
│           └── SKILL.md                         # Emergency hotfix workflow
└── .github/
    └── release.yml                              # GitHub auto-release-notes config
```

---

## 📝 License

MIT — Use freely in your projects.
