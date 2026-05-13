---
name: artifact-signing-sbom
description: Use when users ask to sign artifacts, add SBOM, secure the supply chain, add provenance, or harden a release pipeline with Sigstore/Cosign keyless signing, Software Bill of Materials generation, and SLSA provenance attestation.
---

Configure Sigstore/Cosign keyless signing, SBOM generation, and SLSA provenance attestation for release artifacts.

## When to Use

- Signing release artifacts (containers, binaries, archives)
- Generating SBOM (Software Bill of Materials)
- Adding SLSA provenance attestation
- Hardening an existing release workflow for supply chain security

## Workflow

### 1. Gather Inputs

| Input | Required | Description |
|-------|----------|-------------|
| Artifact type | Yes | `container-image`, `binary`, `archive`, or `npm-package` |
| Registry URL | Conditional | Required for containers (e.g., `ghcr.io/owner/repo`) |
| SBOM format | No | `spdx-json` (default) or `cyclonedx-json` |
| SLSA level | No | 1, 2, or 3 (default: 2) |

### 2. Cosign Keyless Signing

Uses OIDC federation — no private keys. GitHub Actions provides the identity token.

**Containers:**
```yaml
- uses: sigstore/cosign-installer@v3.8.0
- run: cosign sign --yes "${REGISTRY}/${IMAGE}@${DIGEST}"
```

**Binaries:**
```yaml
- run: |
    for artifact in dist/*; do
      cosign sign-blob --yes --output-signature "${artifact}.sig" --output-certificate "${artifact}.pem" "${artifact}"
    done
```

### 3. SBOM Generation

Use Syft (recommended) or CycloneDX. Attest SBOMs to container images with `cosign attest`.

### 4. SLSA Provenance

- **Level 2:** `actions/attest-build-provenance@v2`
- **Level 3:** `slsa-framework/slsa-github-generator`

### 5. Required Permissions

```yaml
permissions:
  contents: write
  packages: write
  id-token: write      # Required for keyless signing
  attestations: write
```

## Answer Shape

Modified workflow YAML with signing/SBOM/attestation steps + verification commands for consumers + security posture summary.

## Edge Cases

- **Self-hosted runners** → ensure network access to `fulcio.sigstore.dev`, `rekor.sigstore.dev`
- **Air-gapped** → fall back to GPG signing with imported keys
- **Multi-arch images** → sign the manifest list digest, not individual platform digests
- **npm packages** → use `npm publish --provenance` for built-in SLSA attestation
