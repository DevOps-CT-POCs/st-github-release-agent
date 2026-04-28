# Skill: Artifact Signing & SBOM

Configure Sigstore/Cosign keyless signing, generate Software Bill of Materials (SBOM), and add SLSA provenance attestation to release artifacts.

---

## When to Use This Skill

- User asks to "sign artifacts," "add SBOM," "secure the supply chain," or "add provenance"
- When hardening an existing release pipeline for supply chain security
- As part of a full release pipeline setup

---

## Inputs

| Input | Required | Description |
|-------|----------|-------------|
| Artifact type | Yes | `container-image`, `binary`, `archive`, or `npm-package` |
| Registry URL | Conditional | Required for container images (e.g., `ghcr.io/owner/repo`) |
| SBOM format | No | `spdx-json` (default) or `cyclonedx-json` |
| SLSA level | No | Target SLSA level: 1, 2, or 3 (default: 2) |

---

## Process

### Step 1: Sigstore/Cosign Keyless Signing

Cosign keyless signing uses **OIDC federation** — no private keys to manage. GitHub Actions provides the identity token automatically.

#### For Container Images

```yaml
- name: Sign container image
  uses: sigstore/cosign-installer@v3.8.0

- name: Sign the image
  env:
    DIGEST: ${{ steps.build.outputs.digest }}
    REGISTRY: ghcr.io
    IMAGE: ${{ github.repository }}
  run: |
    cosign sign --yes "${REGISTRY}/${IMAGE}@${DIGEST}"
```

#### For Binary Artifacts

```yaml
- name: Sign binary artifacts
  run: |
    for artifact in dist/*; do
      cosign sign-blob --yes \
        --output-signature "${artifact}.sig" \
        --output-certificate "${artifact}.pem" \
        "${artifact}"
    done
```

#### Verification Commands (for consumers)

```bash
# Verify container image
cosign verify \
  --certificate-identity-regexp="https://github.com/OWNER/REPO" \
  --certificate-oidc-issuer="https://token.actions.githubusercontent.com" \
  ghcr.io/owner/repo:v1.2.3

# Verify binary artifact
cosign verify-blob \
  --signature artifact.sig \
  --certificate artifact.pem \
  --certificate-identity-regexp="https://github.com/OWNER/REPO" \
  --certificate-oidc-issuer="https://token.actions.githubusercontent.com" \
  artifact.tar.gz
```

### Step 2: SBOM Generation

#### Using Syft (recommended)

```yaml
- name: Generate SBOM
  uses: anchore/sbom-action@v0.18.0
  with:
    image: ghcr.io/${{ github.repository }}:${{ github.sha }}
    format: spdx-json
    output-file: sbom.spdx.json
```

#### Using CycloneDX (alternative)

```yaml
# Node.js
- run: npx @cyclonedx/cyclonedx-npm --output-format json --output-file sbom.cdx.json

# Python
- run: |
    pip install cyclonedx-bom
    cyclonedx-py environment --output-format json -o sbom.cdx.json

# Go
- run: |
    go install github.com/CycloneDX/cyclonedx-gomod/cmd/cyclonedx-gomod@latest
    cyclonedx-gomod app -json -output sbom.cdx.json
```

#### Attest SBOM to Container Image

```yaml
- name: Attest SBOM to image
  run: |
    cosign attest --yes \
      --type spdxjson \
      --predicate sbom.spdx.json \
      "${REGISTRY}/${IMAGE}@${DIGEST}"
```

### Step 3: SLSA Provenance Attestation

#### GitHub Native Attestation (SLSA Level 2)

```yaml
permissions:
  id-token: write
  contents: read
  attestations: write

- name: Generate artifact attestation
  uses: actions/attest-build-provenance@v2
  with:
    subject-path: dist/my-artifact.tar.gz
```

#### SLSA GitHub Generator (SLSA Level 3)

```yaml
provenance:
  needs: build
  permissions:
    actions: read
    id-token: write
    contents: write
  uses: slsa-framework/slsa-github-generator/.github/workflows/generator_generic_slsa3.yml@v2.1.0
  with:
    base64-subjects: ${{ needs.build.outputs.hashes }}
    upload-assets: true
```

#### Verification (for consumers)

```bash
# GitHub native attestation
gh attestation verify artifact.tar.gz --owner OWNER --repo REPO

# SLSA provenance
slsa-verifier verify-artifact artifact.tar.gz \
  --provenance-path artifact.intoto.jsonl \
  --source-uri github.com/owner/repo \
  --source-tag v1.2.3
```

---

## Required Workflow Permissions

```yaml
permissions:
  contents: write
  packages: write
  id-token: write
  attestations: write
```

---

## Output

Provide modified workflow YAML with signing, SBOM, and attestation steps, verification commands for consumers, and a security posture summary.

---

## Edge Cases

1. **Private repos** → Cosign keyless still works; OIDC issuer is the same
2. **Self-hosted runners** → Ensure network access to `fulcio.sigstore.dev`, `rekor.sigstore.dev`
3. **Air-gapped environments** → Fall back to GPG signing with imported keys
4. **Multi-arch images** → Sign the manifest list digest, not individual platform digests
5. **npm packages** → Use `npm publish --provenance` for built-in SLSA attestation
