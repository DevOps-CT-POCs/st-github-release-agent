---
name: artifact-signing-sbom
description: Use when users ask to sign artifacts, add SBOM, secure the supply chain, add provenance, or harden a release pipeline with Sigstore/Cosign keyless signing, Software Bill of Materials generation, and SLSA provenance attestation.
---

Configure Sigstore/Cosign keyless signing, generate Software Bill of Materials (SBOM), and add SLSA provenance attestation to release artifacts.

## When to Use

Use this skill when the request is about:

- Signing release artifacts (container images, binaries, archives)
- Generating an SBOM (Software Bill of Materials)
- Adding SLSA provenance attestation to a release pipeline
- Hardening an existing release workflow for supply chain security
- Verifying signed artifacts or attestations

Do not use this skill for:

- Scaffolding the release workflow itself. Use `github-actions-release`.
- Version calculation or tagging. Use `semantic-versioning`.
- General GitHub Actions syntax. Use `github-actions-docs`.

## Workflow

### 1. Gather Inputs

| Input | Required | Description |
|-------|----------|-------------|
| Artifact type | Yes | `container-image`, `binary`, `archive`, or `npm-package` |
| Registry URL | Conditional | Required for container images (e.g., `ghcr.io/owner/repo`) |
| SBOM format | No | `spdx-json` (default) or `cyclonedx-json` |
| SLSA level | No | Target SLSA level: 1, 2, or 3 (default: 2) |

### 2. Sigstore/Cosign Keyless Signing

Cosign keyless signing uses OIDC federation — no private keys to manage. GitHub Actions provides the identity token automatically.

For container images:
```yaml
- uses: sigstore/cosign-installer@v3.8.0
- run: cosign sign --yes "${REGISTRY}/${IMAGE}@${DIGEST}"
```

For binary artifacts:
```yaml
- run: |
    for artifact in dist/*; do
      cosign sign-blob --yes \
        --output-signature "${artifact}.sig" \
        --output-certificate "${artifact}.pem" \
        "${artifact}"
    done
```

### 3. SBOM Generation

Use Syft (recommended) or CycloneDX for language-specific SBOMs. Attest SBOMs to container images using `cosign attest`.

### 4. SLSA Provenance Attestation

- **Level 2**: Use `actions/attest-build-provenance@v2`
- **Level 3**: Use `slsa-framework/slsa-github-generator`

### 5. Required Permissions

```yaml
permissions:
  contents: write
  packages: write
  id-token: write
  attestations: write
```

## Answer Shape

Provide modified workflow YAML with signing, SBOM, and attestation steps, verification commands for consumers, and a security posture summary.

## Edge Cases

1. **Private repos** → Cosign keyless still works; OIDC issuer is the same
2. **Self-hosted runners** → Ensure network access to `fulcio.sigstore.dev`, `rekor.sigstore.dev`
3. **Air-gapped environments** → Fall back to GPG signing with imported keys
4. **Multi-arch images** → Sign the manifest list digest, not individual platform digests
5. **npm packages** → Use `npm publish --provenance` for built-in SLSA attestation

## Common Mistakes

- Signing individual platform digests instead of the manifest list for multi-arch images
- Forgetting `id-token: write` permission required for keyless signing
- Using long-lived secrets when OIDC keyless authentication is available
- Not providing verification commands for consumers of the signed artifacts
