+++
author = "Adur"
title = "SAST/DAST integration in CI/CD pipelines"
date = "2024-05-12"
description = "Practical guide to integrating Static and Dynamic Application Security Testing into CI/CD pipelines with tool comparisons, pipeline examples, and rollout strategies."
tags = [
    "sast",
    "dast",
    "ci-cd",
    "semgrep",
    "owasp-zap",
    "security-testing"
]
categories = [
    "Security",
    "devops"
]
toc = true
+++

## SAST vs DAST: Understanding the Difference

Application security testing has two fundamental approaches, and knowing when to use each matters for your security posture.

**SAST (Static Application Security Testing)** reads source code, bytecode, or binaries without running the app. It looks for patterns that match known vulnerabilities (SQL injection, XSS, hardcoded credentials, insecure deserialization). Think of it like a security linter that flags dangerous patterns.

**DAST (Dynamic Application Security Testing)** tests a *running application* by sending crafted requests and analyzing responses. It simulates what an attacker would do: probe endpoints, test authentication, look for misconfigurations. DAST sees the app from outside, like a real attacker would.

| Aspect | SAST | DAST |
|--------|------|------|
| When it runs | Build time, on source code | Runtime, against deployed app |
| What it finds | Code flaws, hardcoded secrets, insecure patterns | Runtime vulnerabilities, misconfigurations, auth issues |
| False positive rate | Higher (no runtime context) | Lower (tests real behavior) |
| Language dependency | Yes (needs language-specific rules) | No (tests HTTP/API layer) |
| Coverage | All code paths (including dead code) | Only reachable endpoints |
| Speed | Fast (seconds to minutes) | Slower (minutes to hours) |
| Best stage | Every commit/PR | Staging or pre-production |

Short answer: **use both**. SAST catches problems early and cheap. DAST catches problems that only show up at runtime. They complement each other.

## Tool comparison

### SAST tools

| Tool | Languages | Strengths | License |
|------|-----------|-----------|---------|
| **Semgrep** | 30+ languages | Fast, custom rules, great CI integration | OSS + Commercial |
| **SonarQube** | 25+ languages | Broad ecosystem, quality gates, dashboard | Community + Commercial |
| **Bandit** | Python only | Python-specific, lightweight, easy setup | OSS (Apache 2.0) |
| **CodeQL** | 10+ languages | Deep semantic analysis, GitHub native | Free for OSS |

### DAST tools

| Tool | Type | Strengths | License |
|------|------|-----------|---------|
| **OWASP ZAP** | Proxy-based scanner | Full-featured, scriptable, community rules | OSS (Apache 2.0) |
| **Burp Suite** | Proxy-based scanner | Best-in-class manual + automated testing | Commercial |
| **Nuclei** | Template-based scanner | Fast, huge template library, CI-friendly | OSS (MIT) |

For most teams, **Semgrep + OWASP ZAP** provides an excellent open-source foundation that covers both SAST and DAST without licensing costs.

## GitLab CI pipeline integration

Here is a complete `.gitlab-ci.yml` example that integrates both SAST and DAST into a pipeline with proper gating:

```yaml
stages:
  - build
  - test
  - sast
  - deploy-staging
  - dast
  - deploy-production

variables:
  SEMGREP_RULES: "p/owasp-top-ten p/security-audit"
  ZAP_TARGET_URL: "https://staging.example.com"

# --- SAST Stage ---
semgrep-scan:
  stage: sast
  image: semgrep/semgrep:latest
  script:
    - semgrep ci --config "$SEMGREP_RULES" --json --output semgrep-results.json
    - |
      # Fail pipeline if high/critical findings exceed threshold
      HIGH_COUNT=$(cat semgrep-results.json | jq '[.results[] | select(.extra.severity == "ERROR")] | length')
      echo "High severity findings: $HIGH_COUNT"
      if [ "$HIGH_COUNT" -gt 0 ]; then
        echo "Pipeline blocked: $HIGH_COUNT high severity findings"
        exit 1
      fi
  artifacts:
    paths:
      - semgrep-results.json
    when: always
  allow_failure: false

bandit-scan:
  stage: sast
  image: python:3.11-slim
  script:
    - pip install bandit
    - bandit -r src/ -f json -o bandit-results.json --severity-level medium || true
    - |
      HIGH_COUNT=$(cat bandit-results.json | jq '.results | map(select(.issue_severity == "HIGH")) | length')
      echo "Bandit high severity findings: $HIGH_COUNT"
      if [ "$HIGH_COUNT" -gt 3 ]; then
        exit 1
      fi
  artifacts:
    paths:
      - bandit-results.json
    when: always
  only:
    - merge_requests
    - main

# --- Deploy Staging ---
deploy-staging:
  stage: deploy-staging
  script:
    - echo "Deploying to staging..."
    - ./deploy.sh staging
  environment:
    name: staging
    url: https://staging.example.com

# --- DAST Stage ---
owasp-zap-scan:
  stage: dast
  image: ghcr.io/zaproxy/zaproxy:stable
  script:
    - mkdir -p /zap/wrk
    - |
      zap-full-scan.py \
        -t "$ZAP_TARGET_URL" \
        -r zap-report.html \
        -J zap-results.json \
        -l WARN \
        -d
    - |
      # Parse results and enforce policy
      HIGH_ALERTS=$(cat zap-results.json | jq '[.site[].alerts[] | select(.riskcode == "3")] | length')
      echo "High risk alerts: $HIGH_ALERTS"
      if [ "$HIGH_ALERTS" -gt 0 ]; then
        echo "Pipeline blocked: $HIGH_ALERTS high risk vulnerabilities found"
        exit 1
      fi
  artifacts:
    paths:
      - zap-report.html
      - zap-results.json
    when: always
  dependencies:
    - deploy-staging

# --- Deploy Production ---
deploy-production:
  stage: deploy-production
  script:
    - echo "Deploying to production..."
    - ./deploy.sh production
  environment:
    name: production
  when: manual
  only:
    - main
```

The key design decisions in this pipeline:

- **SAST runs on every merge request**, catching issues before code merges
- **DAST runs against staging**, testing the deployed application
- **Thresholds are configurable**: zero tolerance for high/critical SAST findings, but some flexibility for lower severities
- **Reports are always saved as artifacts**, even when the pipeline passes
- **Production deploy is manual**, gated behind both SAST and DAST stages

## Handling False Positives

False positives are the number one reason teams abandon security scanning. Handle them systematically:

### For Semgrep

Create a `.semgrepignore` file or use inline annotations:

```python
# nosemgrep: python.lang.security.audit.hardcoded-password
DEFAULT_TEST_PASSWORD = "test123"  # Only used in test fixtures
```

For persistent false positives, create a `semgrep-exclusions.yml`:

```yaml
rules:
  - id: ignore-test-passwords
    pattern: $X = "..."
    paths:
      exclude:
        - tests/
        - fixtures/
```

### For OWASP ZAP

Use a context file to exclude known false positives:

```xml
<alertFilter>
  <ruleId>10038</ruleId>
  <url>https://staging.example.com/api/health</url>
  <urlIsRegex>false</urlIsRegex>
  <enabled>true</enabled>
  <newRisk>-1</newRisk> <!-- -1 = False Positive -->
</alertFilter>
```

The golden rule: **every suppression must include a justification comment** explaining why it is safe to ignore. Review suppressions quarterly.

## Threshold and Gate Policies

Define clear policies per environment and severity:

| Severity | PR/MR Gate | Main Branch Gate | Pre-Production |
|----------|-----------|-----------------|----------------|
| Critical | Block | Block | Block |
| High | Block | Block | Warn |
| Medium | Warn | Warn | Info |
| Low | Info | Info | Info |

Implement these as exit codes in your CI scripts. Start permissive and tighten over time -- blocking on everything from day one will create frustration and workarounds.

## Reporting and Dashboards

Aggregate results from both SAST and DAST into a centralized view:

- **DefectDojo**: open-source vulnerability management platform that ingests reports from Semgrep, ZAP, Bandit, and dozens of other tools
- **GitLab Security Dashboard**: native integration if you are on GitLab Ultimate
- **Custom dashboards**: push scan results to Elasticsearch and visualize in Grafana

Track these metrics over time:

- Mean time to remediate (MTTR) per severity
- Total open vulnerabilities by age
- False positive rate (suppressions vs total findings)
- Scan coverage (percentage of repos with security scanning enabled)

## IAST as a Complement

**IAST (Interactive Application Security Testing)** combines elements of both SAST and DAST by instrumenting the application at runtime. An agent runs inside the application during testing, correlating HTTP requests with code execution paths.

IAST tools like Contrast Security and Datadog Application Security provide:

- Lower false positive rates than SAST
- Code-level context that DAST lacks
- Real-time feedback during QA testing

Consider IAST as a third layer once SAST and DAST are mature in your pipeline.

## Practical Rollout Strategy

Deploying security scanning organization-wide requires phases:

### Phase 1: Visibility (Weeks 1-4)
- Enable SAST in CI with `allow_failure: true` (don't block)
- Generate reports, keep as artifacts
- Find top vulnerability categories
- Establish baseline metrics

### Phase 2: Selective Gating (Weeks 5-8)
- Block only critical/high SAST findings
- Add DAST scans to staging for key apps
- Create suppression and triage workflows
- Train developers how to interpret and fix findings

### Phase 3: Full Integration (Weeks 9-12)
- Enforce SAST gates everywhere
- Enforce DAST gates on all web apps
- Integrate with vulnerability management platform
- Auto-create tickets for findings

### Phase 4: Continuous Improvement (Ongoing)
- Write custom Semgrep rules for your patterns
- Tune ZAP policies to reduce scan time
- Track MTTR, set improvement targets
- Quarterly reviews of false positives

Biggest mistake: jumping straight to Phase 3. Start with visibility, build trust, tighten gradually. Scanning should help, not block.
