+++
author = "Adur"
title = "Supply chain security: SBOM and Sigstore"
date = "2025-02-10"
description = "A practical guide to software supply chain security using SBOM, Sigstore, and the SLSA framework."
featured = true
tags = [
    "sbom",
    "sigstore",
    "cosign",
    "supply-chain",
    "security",
    "slsa",
]
categories = [
    "Security",
    "devops",
]
series = ["Security"]
thumbnail = "images/seguridad.png"
toc = true
+++

Supply chain attacks are no longer theoretical. The 2020 SolarWinds breach showed how one compromised build pipeline could hit thousands of organizations, including government agencies. Then Log4Shell in 2021 proved that a vulnerability deep in a transitive dependency could threaten every Java app overnight. The message was clear: we need visibility into what's in our software and stronger integrity guarantees.

This guide covers the practical tools: SBOMs, Sigstore, and SLSA framework.

<!--more-->

## Why Supply Chain Security Matters

Traditional security focuses on your own code: static analysis, dependency scanning, penetration testing. Supply chain security extends that perimeter to everything your software depends on and every step in the process of building and delivering it.

The attack surface includes:

- **Source code repositories**: compromised developer accounts, malicious commits
- **Dependencies**: typosquatting, dependency confusion, compromised upstream packages
- **Build systems**: tampered CI/CD pipelines, injected build steps
- **Artifact registries**: replaced binaries, unsigned packages
- **Deployment pipelines**: modified manifests, man-in-the-middle attacks

A single weak link in any of these stages can compromise the entire chain.

## Software Bill of Materials (SBOM)

An SBOM is a formal, machine-readable inventory of all components in a piece of software. Think of it as an ingredients list for your application. It includes direct dependencies, transitive dependencies, their versions, licenses, and relationships.

### Why You Need One

- **Vulnerability response**: When a new CVE drops (like Log4Shell), you can instantly check if any of your applications are affected.
- **License compliance**: Know exactly what licenses you are shipping.
- **Regulatory requirements**: The US Executive Order 14028 and the EU Cyber Resilience Act both push toward mandatory SBOMs.
- **Transparency**: Customers and partners can verify what they are running.

### SBOM Formats

Two main formats dominate:

- **SPDX** (Software Package Data Exchange): An ISO standard (ISO/IEC 5962:2021), originally focused on license compliance, now comprehensive. Supports JSON, RDF, YAML, and tag-value formats.
- **CycloneDX**: An OWASP project, designed from the ground up for security use cases. Supports JSON and XML. Lighter weight and more opinionated.

Both are solid choices. CycloneDX tends to be easier to work with for security-focused workflows. SPDX has broader adoption in compliance-heavy industries.

### Generating SBOMs with Syft

[Syft](https://github.com/anchore/syft) from Anchore is one of the best tools for generating SBOMs. It supports container images, filesystems, and archives.

**Install syft:**

```bash
curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sh -s -- -b /usr/local/bin
```

**Generate an SBOM from a container image:**

```bash
# CycloneDX format (JSON)
syft packages registry.example.com/myapp:v1.2.3 -o cyclonedx-json > sbom.cdx.json

# SPDX format (JSON)
syft packages registry.example.com/myapp:v1.2.3 -o spdx-json > sbom.spdx.json
```

**Generate an SBOM from a local directory:**

```bash
syft packages dir:/path/to/project -o cyclonedx-json > sbom.cdx.json
```

You can then scan the SBOM for vulnerabilities using [Grype](https://github.com/anchore/grype):

```bash
grype sbom:sbom.cdx.json
```

## The Sigstore Ecosystem

[Sigstore](https://www.sigstore.dev/) is an open-source project that makes cryptographic signing and verification accessible. It eliminates the need to manage long-lived signing keys, which has historically been the main barrier to adoption of artifact signing.

The ecosystem has three core components:

- **cosign**: Signs and verifies container images and other OCI artifacts.
- **Fulcio**: A certificate authority that issues short-lived certificates based on OIDC identity (your existing identity provider).
- **Rekor**: A transparency log that creates an immutable, tamper-resistant record of signing events.

### How It Works

1. You authenticate with an OIDC provider (GitHub, Google, Microsoft, etc.).
2. Fulcio issues a short-lived certificate tied to your identity.
3. cosign uses that certificate to sign your artifact.
4. The signing event is recorded in Rekor's transparency log.
5. Anyone can verify the signature using the transparency log, without needing your public key.

This is called "keyless" signing. No keys to rotate, no secrets to manage, no PKI infrastructure to maintain.

### Signing Container Images with Cosign

**Install cosign:**

```bash
# Using Go
go install github.com/sigstore/cosign/v2/cmd/cosign@latest

# Or download a release
curl -sSfL https://github.com/sigstore/cosign/releases/latest/download/cosign-linux-amd64 -o /usr/local/bin/cosign
chmod +x /usr/local/bin/cosign
```

**Sign an image (keyless mode):**

```bash
cosign sign registry.example.com/myapp:v1.2.3
```

This will open a browser for OIDC authentication. In CI, you can use workload identity (e.g., GitHub Actions OIDC token) for non-interactive signing.

**Verify an image:**

```bash
cosign verify registry.example.com/myapp:v1.2.3 \
  --certificate-identity=user@example.com \
  --certificate-oidc-issuer=https://accounts.google.com
```

**Attach an SBOM to an image and sign it:**

```bash
# Attach the SBOM
cosign attach sbom --sbom sbom.cdx.json registry.example.com/myapp:v1.2.3

# Sign the SBOM attachment
cosign sign --attachment sbom registry.example.com/myapp:v1.2.3
```

## The SLSA Framework

[SLSA](https://slsa.dev/) (Supply-chain Levels for Software Artifacts, pronounced "salsa") is a framework that defines increasing levels of supply chain integrity guarantees.

### SLSA Levels

- **Level 0**: No guarantees. This is where most projects start.
- **Level 1**: The build process is documented and produces provenance (metadata about how an artifact was built).
- **Level 2**: The build is hosted on a hosted build service that generates authenticated provenance.
- **Level 3**: The build platform provides hardened builds with tamper-resistant provenance. The build environment is isolated and ephemeral.

Each level builds on the previous one. The goal is not to jump to Level 3 immediately but to incrementally improve your posture.

### SLSA Provenance

Provenance answers the critical questions: Who built this? What source was used? What build process was followed? Was the build environment tamper-proof?

SLSA provenance is a signed attestation in the [in-toto](https://in-toto.io/) format that captures this information.

## CI/CD Integration: GitHub Actions Example

Here is a practical GitHub Actions workflow that builds an image, generates an SBOM, signs everything, and generates SLSA provenance:

```yaml
name: Build, Sign, and Attest

on:
  push:
    tags:
      - 'v*'

permissions:
  contents: read
  packages: write
  id-token: write  # Required for keyless signing

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-sign-attest:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push image
        id: build
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.ref_name }}

      - name: Install cosign
        uses: sigstore/cosign-installer@v3

      - name: Install syft
        uses: anchore/sbom-action/download-syft@v0

      - name: Sign the image
        run: |
          cosign sign --yes \
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.ref_name }}@${{ steps.build.outputs.digest }}

      - name: Generate SBOM
        run: |
          syft packages \
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.ref_name }}@${{ steps.build.outputs.digest }} \
            -o cyclonedx-json > sbom.cdx.json

      - name: Attach and sign SBOM
        run: |
          cosign attach sbom --sbom sbom.cdx.json \
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.ref_name }}@${{ steps.build.outputs.digest }}
          cosign sign --yes --attachment sbom \
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.ref_name }}@${{ steps.build.outputs.digest }}

      - name: Verify signature
        run: |
          cosign verify \
            --certificate-identity-regexp="https://github.com/${{ github.repository }}/*" \
            --certificate-oidc-issuer=https://token.actions.githubusercontent.com \
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.ref_name }}@${{ steps.build.outputs.digest }}
```

The `id-token: write` permission is what enables keyless signing in GitHub Actions. The GitHub OIDC token is automatically used by cosign without any manual key management.

## Practical Adoption Roadmap

You do not need to do everything at once. Here is a sensible progression:

**Week 1-2: Visibility**
- Start generating SBOMs for your most critical applications using syft.
- Integrate Grype into your CI pipeline for vulnerability scanning against the SBOM.

**Week 3-4: Signing**
- Set up cosign keyless signing in your CI/CD pipelines.
- Sign your container images on every build.

**Month 2: Verification**
- Enforce signature verification in your deployment pipelines (e.g., Kubernetes admission controllers like Kyverno or Sigstore policy-controller).
- Attach SBOMs to images and sign them.

**Month 3+: SLSA Provenance**
- Add SLSA provenance generation using [slsa-github-generator](https://github.com/slsa-framework/slsa-github-generator).
- Work toward SLSA Level 2, then Level 3.
- Automate provenance verification in your deployment tooling.

## Key Takeaways

- **SBOMs give visibility** - You can't secure what you can't see. Generate SBOMs for every artifact.
- **Sigstore removes the excuse** - Keyless signing kills key management overhead. No good reason not to sign.
- **SLSA is a maturity model** - Use it to improve supply chain integrity incrementally, not as all-or-nothing.
- **Automate everything** - These tools are built for CI/CD integration. Manual doesn't scale.

The supply chain security ecosystem is maturing fast. Tools are production-ready, standards are solidifying, and regulatory pressure keeps rising. The best time to start was yesterday. The second-best is now.
