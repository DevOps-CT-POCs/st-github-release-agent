# Skill: Hotfix Release

Handle emergency production patches with an expedited release workflow — branch from tag, cherry-pick fixes, fast-track CI, and deploy with approval.

---

## When to Use This Skill

- User reports a critical bug in production that needs an immediate fix
- User asks for a "hotfix," "emergency patch," or "production fix"
- A security vulnerability requires an out-of-band release

---

## Inputs

| Input | Required | Description |
|-------|----------|-------------|
| Issue description | Yes | What's broken and its severity (critical/high) |
| Target release tag | No | Which release to patch (default: latest) |
| Fix commits/PRs | No | Specific commits or PRs to cherry-pick |

---

## Process

### Step 1: Identify the Release to Patch

```bash
# Find the latest release tag
LATEST_TAG=$(git describe --tags --abbrev=0 --match 'v[0-9]*.[0-9]*.[0-9]*' 2>/dev/null)
echo "Patching from: ${LATEST_TAG}"

# Verify the tag exists and is on the expected branch
git log --oneline -1 "${LATEST_TAG}"
```

### Step 2: Create the Hotfix Branch

```bash
# Branch naming convention: hotfix/v{major}.{minor}.{patch+1}
CURRENT_VERSION="${LATEST_TAG#v}"  # e.g., 1.2.3
IFS='.' read -r MAJOR MINOR PATCH <<< "$CURRENT_VERSION"
NEW_PATCH=$((PATCH + 1))
HOTFIX_VERSION="${MAJOR}.${MINOR}.${NEW_PATCH}"
HOTFIX_BRANCH="hotfix/v${HOTFIX_VERSION}"

# Create the branch from the release tag
git checkout -b "${HOTFIX_BRANCH}" "${LATEST_TAG}"
git push -u origin "${HOTFIX_BRANCH}"
```

### Step 3: Apply the Fix

#### Option A: Cherry-pick existing commits

```bash
# Cherry-pick specific fix commits from main
git cherry-pick --no-commit <commit-sha-1> <commit-sha-2>

# Resolve any conflicts, then:
git commit -m "fix: <description of the fix>

Cherry-picked from <commit-sha> on main.
Fixes #<issue-number>"
```

#### Option B: Write the fix directly

Apply the fix on the hotfix branch. Ensure the commit follows Conventional Commits:

```bash
git add .
git commit -m "fix(<scope>): <description>

<detailed explanation of the fix>

Fixes #<issue-number>
Severity: critical"
```

### Step 4: Expedited Version Bump (Patch Only)

Hotfixes are **always PATCH bumps**. Never bump MINOR or MAJOR from a hotfix branch.

```bash
# Update version in manifest files
# Node.js:
npm version patch --no-git-tag-version

# Or manually update version.txt:
echo "${HOTFIX_VERSION}" > version.txt

# Commit the version bump
git add .
git commit -m "chore(release): bump version to v${HOTFIX_VERSION}"
```

### Step 5: Create Hotfix Release Tag

```bash
git tag -a "v${HOTFIX_VERSION}" -m "Hotfix release v${HOTFIX_VERSION}

## 🚨 Hotfix

- **Issue**: <brief description>
- **Severity**: Critical
- **Fix**: <what was changed>
- **Cherry-picked from**: <commit SHA(s)>
"
git push origin "v${HOTFIX_VERSION}"
```

### Step 6: Fast-Track CI Pipeline

The hotfix workflow skips staging and deploys directly to production with approval:

```yaml
# .github/workflows/hotfix.yml
name: Hotfix Release

on:
  push:
    branches: ['hotfix/**']
    tags: ['v[0-9]+.[0-9]+.[0-9]+']

permissions:
  contents: write
  packages: write
  id-token: write

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run critical tests only
        run: |
          # Run only the test suites relevant to the fix
          npm test -- --grep "critical|smoke|integration"

  build:
    needs: validate
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build
        run: npm ci && npm run build

  deploy-production:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: production
      # Requires manual approval — but reviewers are alerted immediately
    steps:
      - name: Deploy hotfix to production
        run: |
          echo "🚨 Deploying hotfix ${GITHUB_REF_NAME} to production"

      - name: Verify fix
        run: |
          echo "Verifying the fix is applied..."
          # Add verification commands
```

### Step 7: Back-merge to Main

After the hotfix is deployed successfully, merge the changes back to `main`:

```bash
# Switch to main and merge the hotfix
git checkout main
git pull origin main
git merge --no-ff "${HOTFIX_BRANCH}" -m "chore: merge hotfix/v${HOTFIX_VERSION} into main"
git push origin main
```

If there are conflicts, create a PR instead:

```bash
gh pr create \
  --base main \
  --head "${HOTFIX_BRANCH}" \
  --title "chore: merge hotfix v${HOTFIX_VERSION} back to main" \
  --body "Back-merging hotfix changes to main branch.

## Hotfix Details
- **Version**: v${HOTFIX_VERSION}
- **Issue**: #<issue-number>
- **Fix**: <description>"
```

### Step 8: Post-Mortem Template

Generate a post-mortem document for the team:

```markdown
# Post-Mortem: Hotfix v{VERSION}

## Summary
- **Date**: YYYY-MM-DD
- **Severity**: Critical / High
- **Duration**: Time from issue detection to fix deployment
- **Impact**: Description of user/system impact

## Timeline
| Time (UTC) | Event |
|-----------|-------|
| HH:MM | Issue detected by [monitoring/user report] |
| HH:MM | Hotfix branch created |
| HH:MM | Fix implemented and tested |
| HH:MM | Deployed to production |
| HH:MM | Fix verified, incident resolved |

## Root Cause
<What caused the issue>

## Fix Applied
<What was changed and why>

## Action Items
- [ ] Add regression test for this scenario
- [ ] Update monitoring/alerting to catch this earlier
- [ ] Review similar code paths for the same pattern

## Lessons Learned
<What we learned and how to prevent recurrence>
```

---

## Output

Provide:

1. Hotfix branch name and creation commands
2. Cherry-pick or fix application instructions
3. Version bump and tag commands
4. Back-merge instructions
5. Post-mortem template (filled in with available details)

---

## Edge Cases

1. **Multiple tags on the same commit** → Use the highest semver tag
2. **Hotfix conflicts with main** → Create a PR for the back-merge
3. **Hotfix on an older release** → Support patching `v1.x` while `v2.x` is current
4. **No test suite** → Warn that manual verification is required; still proceed
5. **Dependent hotfixes** → If a second hotfix is needed before the first is merged back, stack on the existing hotfix branch
