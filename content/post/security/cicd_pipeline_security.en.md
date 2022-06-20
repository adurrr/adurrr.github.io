+++
author = "Adur"
title = "CI/CD pipeline security: a shift-left approach"
date = "2022-06-20"
description = "How to integrate security into your CI/CD pipelines from the start with SAST, DAST, secret scanning, and dependency analysis."
featured = true
tags = [
    "ci-cd",
    "security",
    "shift-left",
    "sast",
    "dast",
    "supply-chain"
]
categories = [
    "Security",
    "devops"
]
series = ["Security"]
thumbnail = "images/seguridad.png"
toc = true
+++

## What is shift-left security?

Shift-left security means moving security practices earlier in the software development lifecycle. Rather than treating security as a final gate (or worse, an afterthought) before production, you embed security checks directly into your CI/CD pipelines. This catches vulnerabilities when they're cheapest to fix: at development time.

The traditional "build it, then audit it" model doesn't scale. Modern applications ship dozens of times a day. If your security review is a manual quarterly process, you're deploying vulnerable code most of the time.

## Threat vectors in CI/CD pipelines

Before hardening your pipeline, understand what you're defending against. CI/CD systems are high-value targets because they have access to production credentials, source code, and build artifacts.

Common threat vectors include:

- **Compromised dependencies**: A malicious package in your dependency tree (supply chain attack).
- **Leaked secrets**: API keys, tokens, or passwords committed to source code or exposed in build logs.
- **Code vulnerabilities**: SQL injection, XSS, insecure deserialization, and other OWASP Top 10 issues introduced in your application code.
- **Insecure pipeline configuration**: Overly permissive runner access, unprotected branch rules, or missing approval gates.
- **Container image vulnerabilities**: Base images with known CVEs that end up in production.
- **Build artifact tampering**: Unsigned or unverified artifacts that attackers can replace.

## Integrating SAST into your pipeline

**Static Application Security Testing (SAST)** analyzes source code without executing it. It catches vulnerabilities early, before code even runs.

### Recommended tools

- **Semgrep**: Fast, flexible, and supports custom rules. Works across many languages. My top recommendation for most teams.
- **Bandit**: Python-specific. Excellent for Python projects because it understands Python-specific security patterns.
- **SonarQube**: Broader scope (code quality and security combined). Heavier to set up but useful if you want a single dashboard.

### Running Semgrep locally

```bash
# Install
pip install semgrep

# Run with default rulesets
semgrep --config auto .

# Run with OWASP Top 10 rules specifically
semgrep --config "p/owasp-top-ten" .
```

### Running Bandit for Python projects

```bash
pip install bandit
bandit -r ./src -f json -o bandit-report.json
```

The key is to run these tools on every pull request, not just on the main branch. Developers should see findings before code is merged.

## Integrating DAST into your pipeline

**Dynamic Application Security Testing (DAST)** tests the running application from the outside, simulating an attacker. It catches issues that SAST misses (misconfigurations, authentication flaws, and runtime vulnerabilities).

### OWASP ZAP

OWASP ZAP is the standard open-source DAST tool. You can run it in your pipeline against a staging deployment:

```bash
# Pull the ZAP Docker image
docker pull ghcr.io/zaproxy/zaproxy:stable

# Run a baseline scan against a target URL
docker run -t ghcr.io/zaproxy/zaproxy:stable zap-baseline.py \
  -t https://staging.example.com \
  -r zap-report.html
```

DAST is typically slower than SAST. Run it after deployment to a staging environment rather than on every commit. A nightly or per-release cadence works well for most teams.

## Secret scanning

Leaked secrets are one of the most common and damaging security failures. Once a secret makes it into Git history, consider it compromised, even if you remove it in a later commit.

### Tools

- **gitleaks**: Scans Git repositories for secrets using regex and entropy-based detection.
- **trufflehog**: Searches through Git history for high-entropy strings and known secret patterns.

### Running gitleaks

```bash
# Install
brew install gitleaks  # or download binary from GitHub releases

# Scan the current repo
gitleaks detect --source . --report-path gitleaks-report.json

# Scan in CI (detect only new leaks in the PR)
gitleaks detect --source . --log-opts="origin/main..HEAD"
```

### Pre-commit hook

Block secrets before they even reach the remote:

```bash
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.16.1
    hooks:
      - id: gitleaks
```

## Dependency scanning

Your application is only as secure as its weakest dependency. Supply chain attacks are increasing, so you need to scan your dependency tree regularly.

### Tools

- **Trivy**: Scans container images, filesystems, and Git repos for vulnerabilities. Fast and comprehensive.
- **Snyk**: Commercial option with a generous free tier. Good developer experience.
- **OWASP Dependency-Check**: Mature, open-source, supports multiple ecosystems.

```bash
# Scan a container image with Trivy
trivy image my-app:latest

# Scan the filesystem for vulnerable dependencies
trivy fs --severity HIGH,CRITICAL .
```

## GitHub Actions workflow with security stages

Here is a practical GitHub Actions workflow that integrates all the security stages discussed above:

```yaml
name: Secure CI/CD Pipeline

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  secret-scan:
    name: Secret Scanning
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - name: Run gitleaks
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  sast:
    name: Static Analysis
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Semgrep
        uses: returntocorp/semgrep-action@v1
        with:
          config: >-
            p/security-audit
            p/owasp-top-ten
      - name: Run Bandit (Python)
        run: |
          pip install bandit
          bandit -r ./src -f json -o bandit-report.json || true
      - name: Upload SAST reports
        uses: actions/upload-artifact@v4
        with:
          name: sast-reports
          path: "*.json"

  dependency-scan:
    name: Dependency Scanning
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Trivy filesystem scan
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: fs
          severity: HIGH,CRITICAL
          exit-code: 1

  build-and-scan-image:
    name: Build & Scan Image
    needs: [secret-scan, sast, dependency-scan]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build Docker image
        run: docker build -t my-app:${{ github.sha }} .
      - name: Scan image with Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: my-app:${{ github.sha }}
          severity: HIGH,CRITICAL
          exit-code: 1

  deploy-staging:
    name: Deploy to Staging
    needs: [build-and-scan-image]
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - name: Deploy to staging
        run: echo "Deploy to staging environment"

  dast:
    name: Dynamic Analysis
    needs: [deploy-staging]
    runs-on: ubuntu-latest
    steps:
      - name: OWASP ZAP Baseline Scan
        uses: zaproxy/action-baseline@v0.9.0
        with:
          target: "https://staging.example.com"
```

Notice the deliberate ordering: secret scanning, SAST, and dependency scanning run in parallel (fast feedback). Image scanning runs after build. DAST runs after staging deployment.

## Best practices checklist

Here is a checklist to assess the security maturity of your CI/CD pipeline:

- [ ] **Secret scanning** runs on every PR and blocks merge on findings.
- [ ] **Pre-commit hooks** prevent secrets from being committed locally.
- [ ] **SAST** runs on every PR with rules covering OWASP Top 10.
- [ ] **Dependency scanning** runs on every PR and flags HIGH/CRITICAL CVEs.
- [ ] **Container image scanning** runs before pushing images to a registry.
- [ ] **DAST** runs against staging on every release (or nightly at minimum).
- [ ] **Branch protection** requires passing security checks before merge.
- [ ] **Least privilege** is enforced on CI runner permissions and secrets access.
- [ ] **Signed commits** and **signed artifacts** are enforced for production releases.
- [ ] **Build logs** are reviewed to ensure secrets are not leaked in output.
- [ ] **Dependency pinning** is used (exact versions, not ranges) to prevent supply chain attacks.
- [ ] **Regular rotation** of all CI/CD secrets and service account keys.

## Final thoughts

Shift-left security isn't about buying a tool, it's about changing the culture. Security checks in CI/CD should be fast, automated, and non-negotiable. Start with secret scanning (highest ROI), add SAST, then layer in dependency scanning and DAST.

The goal isn't to catch everything in the pipeline. The goal is to raise the bar high enough that obvious issues never reach production, freeing your security team to focus on the subtle, high-impact threats that require human judgment.
