# Workflow Templates Reference

Reusable YAML snippets for common GitHub Actions release patterns. Copy and adapt these into your workflows.

---

## Docker Build & Push (with GHCR)

```yaml
- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v3

- name: Login to GitHub Container Registry
  uses: docker/login-action@v3
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}

- name: Build and push
  id: build
  uses: docker/build-push-action@v6
  with:
    context: .
    push: true
    tags: |
      ghcr.io/${{ github.repository }}:${{ github.ref_name }}
      ghcr.io/${{ github.repository }}:latest
    cache-from: type=gha
    cache-to: type=gha,mode=max
    labels: |
      org.opencontainers.image.source=${{ github.server_url }}/${{ github.repository }}
      org.opencontainers.image.revision=${{ github.sha }}
      org.opencontainers.image.version=${{ github.ref_name }}
```

---

## Multi-Platform Matrix Build

```yaml
strategy:
  fail-fast: false
  matrix:
    include:
      - os: ubuntu-latest
        goos: linux
        goarch: amd64
        suffix: linux-amd64
      - os: ubuntu-latest
        goos: linux
        goarch: arm64
        suffix: linux-arm64
      - os: macos-latest
        goos: darwin
        goarch: amd64
        suffix: darwin-amd64
      - os: macos-latest
        goos: darwin
        goarch: arm64
        suffix: darwin-arm64
      - os: windows-latest
        goos: windows
        goarch: amd64
        suffix: windows-amd64.exe
```

---

## Slack Notification

```yaml
- name: Notify Slack
  if: always()
  uses: slackapi/slack-github-action@v2.0.0
  with:
    webhook: ${{ secrets.SLACK_WEBHOOK_URL }}
    webhook-type: incoming-webhook
    payload: |
      {
        "text": "${{ job.status == 'success' && '✅' || '❌' }} Release ${{ needs.prepare.outputs.version }} — ${{ job.status }}",
        "blocks": [
          {
            "type": "section",
            "text": {
              "type": "mrkdwn",
              "text": "${{ job.status == 'success' && '✅' || '❌' }} *Release ${{ needs.prepare.outputs.version }}*\nStatus: `${{ job.status }}`\nRepo: `${{ github.repository }}`\n<${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|View Run>"
            }
          }
        ]
      }
```

---

## Rollback Job

```yaml
rollback:
  needs: [prepare, deploy-production]
  if: failure() && needs.deploy-production.result == 'failure'
  runs-on: ubuntu-latest
  environment: production
  steps:
    - name: Get previous release tag
      id: previous
      run: |
        CURRENT="${{ needs.prepare.outputs.version }}"
        PREVIOUS=$(git tag --sort=-version:refname | grep -v "$CURRENT" | head -1)
        echo "tag=$PREVIOUS" >> "$GITHUB_OUTPUT"

    - name: Rollback deployment
      run: |
        echo "Rolling back to ${{ steps.previous.outputs.tag }}..."
        # Add rollback commands here

    - name: Notify team
      run: |
        echo "⚠️ Rollback triggered for ${{ needs.prepare.outputs.version }}"
        echo "Rolled back to ${{ steps.previous.outputs.tag }}"
```

---

## Concurrency Control

```yaml
concurrency:
  group: release-${{ github.ref }}
  cancel-in-progress: false  # Never cancel in-progress releases
```

---

## Artifact Checksums

```yaml
- name: Generate checksums
  run: |
    cd dist
    sha256sum * > SHA256SUMS.txt
    cat SHA256SUMS.txt

- name: Verify checksums
  run: |
    cd dist
    sha256sum --check SHA256SUMS.txt
```

---

## OIDC Cloud Authentication

### AWS

```yaml
- name: Configure AWS credentials (OIDC)
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
    aws-region: us-east-1
```

### Azure

```yaml
- name: Azure Login (OIDC)
  uses: azure/login@v2
  with:
    client-id: ${{ secrets.AZURE_CLIENT_ID }}
    tenant-id: ${{ secrets.AZURE_TENANT_ID }}
    subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

### GCP

```yaml
- name: Authenticate to Google Cloud (OIDC)
  uses: google-github-actions/auth@v2
  with:
    workload_identity_provider: projects/123/locations/global/workloadIdentityPools/pool/providers/github
    service_account: deploy@project.iam.gserviceaccount.com
```

---

## Environment Protection Rules Setup

```
GitHub UI: Settings → Environments → [environment name]

Production:
  ✅ Required reviewers: @release-approvers (team)
  ✅ Wait timer: 15 minutes
  ✅ Deployment branches: main only
  ✅ Prevent self-review: enabled

Staging:
  ✅ Deployment branches: main, release/*
  ❌ No required reviewers (auto-deploy)
```
