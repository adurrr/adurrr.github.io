---
title: "DevSecOps maturity model"
date: 2023-10-08T10:00:00+01:00
draft: false
tags:
  - "devsecops"
  - "maturity-model"
  - "security"
  - "automation"
  - "culture"

categories:
  - "devops"
  - "Security"

toc: true
---

## Why a Maturity Model Helps

Most teams know they should "shift security left," but knowing where to start is the hard part. A maturity model gives you a structured way to assess your current state, identify gaps, and plan a realistic roadmap for improvement.

Without a model, security improvements tend to be reactive (triggered by incidents or audit findings rather than deliberate planning). A maturity model turns security from a fire drill into an engineering discipline with measurable progress.

The model described here has five levels. The goal is not to rush to the highest level but to make steady, sustainable progress. Each level builds on the previous one.

## The Five Maturity Levels

### Level 1: Ad-Hoc

At this level, security is an afterthought. There are no formal processes, and security activities happen sporadically if at all.

**What it looks like:**
- No security testing in CI/CD pipelines.
- Vulnerabilities discovered in production or by external parties.
- No dedicated security tooling.
- Developers have little to no security training.
- Incident response is improvised.
- Compliance is addressed manually before audits.

**Typical tools:** None specifically for security. Maybe a firewall and antivirus.

### Level 2: Reactive

Security is recognized as important, but the approach is reactive. The team responds to vulnerabilities and incidents but doesn't proactively prevent them.

**What it looks like:**
- Basic static analysis (SAST) runs occasionally, but findings are not always addressed.
- Dependency scanning is done manually or on an ad-hoc basis.
- There's some security documentation, but it's outdated.
- Incident response exists as a documented process, though it's rarely practiced.
- Security reviews happen late in the development cycle (right before release).

**Typical tools:** SonarQube (basic rules), OWASP Dependency-Check, manual penetration testing.

### Level 3: Proactive

Security is integrated into the development workflow. The team actively seeks to prevent vulnerabilities rather than just reacting to them.

**What it looks like:**
- SAST and DAST run automatically in CI/CD pipelines.
- Dependency scanning with automated alerts for known vulnerabilities.
- Container image scanning before deployment (Trivy, Grype).
- Infrastructure as Code is scanned for misconfigurations (Checkov, tfsec).
- Threat modeling is performed for new features and architecture changes.
- Security champions exist within development teams.
- Blameless postmortems are conducted after security incidents.
- Regular security training for developers.

**Typical tools:** Semgrep, Trivy, Checkov, OWASP ZAP, HashiCorp Vault, Falco.

### Level 4: Optimized

Security is deeply embedded in every stage of the software lifecycle. Metrics drive decisions, and the team continuously improves based on data.

**What it looks like:**
- Security gates in pipelines that block deployment if critical issues are found.
- Mean time to remediate (MTTR) is tracked and continuously reduced.
- Software Bill of Materials (SBOM) generated for every release.
- Signed artifacts and verified supply chain.
- Automated compliance checks mapped to frameworks (SOC2, ISO 27001, PCI-DSS).
- Runtime security monitoring with automated response (Falco + custom rules).
- Regular red team exercises and chaos engineering for security.
- Security metrics are part of engineering dashboards.

**Typical tools:** Sigstore/cosign, OPA/Gatekeeper, Kyverno, SIEM integration, automated compliance platforms.

### Level 5: Innovative

Security is a competitive advantage. The team contributes to the broader security community and pushes the state of the art.

**What it looks like:**
- Bug bounty programs actively managed.
- Custom security tooling developed for organization-specific risks.
- Machine learning applied to anomaly detection and threat hunting.
- Security is a feature sold to customers (certifications, transparency reports).
- Active participation in open-source security projects.
- Zero-trust architecture fully implemented.
- Policy as code governs all infrastructure and application security.

**Typical tools:** Custom-built platforms, eBPF-based security tools, advanced SIEM with ML, zero-trust service mesh.

## Key Dimensions

A maturity model isn't one-dimensional. Assess your organization across these dimensions, as progress is rarely uniform:

### Code Security

| Level | Practices |
|-------|-----------|
| Ad-Hoc | No code scanning |
| Reactive | Occasional SAST, manual code reviews for security |
| Proactive | Automated SAST/DAST in CI, security-focused code review guidelines |
| Optimized | Custom rules for organization-specific patterns, MTTR tracked |
| Innovative | AI-assisted code review, automatic fix suggestions |

### Infrastructure Security

| Level | Practices |
|-------|-----------|
| Ad-Hoc | Manual server configuration, no hardening standards |
| Reactive | Basic hardening checklists, occasional audits |
| Proactive | IaC scanning, automated hardening, CIS benchmarks |
| Optimized | Policy as code (OPA), drift detection, automated remediation |
| Innovative | Self-healing infrastructure, zero-trust networking |

### Monitoring and Detection

| Level | Practices |
|-------|-----------|
| Ad-Hoc | No security monitoring |
| Reactive | Basic log collection, manual review after incidents |
| Proactive | Centralized logging, alerting on known patterns, runtime monitoring |
| Optimized | SIEM with correlation rules, automated response playbooks |
| Innovative | ML-based anomaly detection, threat hunting programs |

### Incident Response

| Level | Practices |
|-------|-----------|
| Ad-Hoc | No process, ad-hoc response |
| Reactive | Documented runbooks, rarely tested |
| Proactive | Regular tabletop exercises, blameless postmortems, on-call rotation |
| Optimized | Automated incident classification, SLA-driven response times |
| Innovative | Chaos engineering for security, automated containment |

### Compliance

| Level | Practices |
|-------|-----------|
| Ad-Hoc | Manual evidence collection before audits |
| Reactive | Spreadsheet-based tracking, periodic reviews |
| Proactive | Automated evidence collection, continuous monitoring |
| Optimized | Compliance as code, real-time dashboards, automated reporting |
| Innovative | Continuous certification, public transparency reports |

## Self-Assessment Checklist

Rate your organization on each item (Yes / Partial / No):

**Build Phase:**
- [ ] SAST runs automatically on every pull request.
- [ ] Dependency scanning alerts on known CVEs.
- [ ] Container images are scanned before being pushed to a registry.
- [ ] IaC templates are scanned for misconfigurations.
- [ ] Secrets detection prevents credentials from being committed.

**Deploy Phase:**
- [ ] Security gates can block deployment for critical findings.
- [ ] Artifacts are signed and signatures are verified.
- [ ] SBOM is generated for every release.
- [ ] Infrastructure changes go through policy-as-code validation.

**Run Phase:**
- [ ] Runtime security monitoring is active (Falco, Sysdig, etc.).
- [ ] Centralized logging with security-relevant alerts.
- [ ] Network segmentation limits blast radius.
- [ ] Secrets are managed through a dedicated vault.

**Culture and Process:**
- [ ] Developers receive regular security training.
- [ ] Security champions are embedded in development teams.
- [ ] Blameless postmortems are conducted after incidents.
- [ ] Threat modeling is part of the design process for new features.
- [ ] Security metrics are tracked and reviewed regularly.

## Roadmap for Progression

Moving up the maturity levels doesn't happen overnight. Here's a practical roadmap:

### From Ad-Hoc to Reactive (3-6 months)

1. Add a SAST tool to your CI pipeline (start with Semgrep - it has good defaults and is fast).
2. Enable dependency scanning (GitHub Dependabot, or `trivy fs` in CI).
3. Document your incident response process, even if it's simple.
4. Run a single security training session for the team.

### From Reactive to Proactive (6-12 months)

1. Add container image scanning and IaC scanning to pipelines.
2. Implement secrets detection in pre-commit hooks (`gitleaks`, `detect-secrets`).
3. Appoint security champions in each team.
4. Start threat modeling for major features.
5. Conduct your first blameless postmortem after an incident.
6. Deploy runtime monitoring (Falco).

### From Proactive to Optimized (12-18 months)

1. Implement security gates that can block deployments.
2. Track MTTR and set reduction targets.
3. Generate SBOMs and sign artifacts.
4. Implement policy-as-code for infrastructure (OPA/Gatekeeper).
5. Map automated checks to compliance frameworks.
6. Integrate security metrics into engineering dashboards.

### From Optimized to Innovative (18+ months)

1. Launch a bug bounty program.
2. Build custom security tooling for organization-specific risks.
3. Implement zero-trust architecture.
4. Run regular red team exercises.
5. Contribute to open-source security projects.

## Cultural Aspects

Tools and processes are necessary but insufficient. Culture determines whether security practices actually stick.

### Blameless Postmortems

When a security incident occurs, the instinct is often to find someone to blame. This drives people to hide mistakes and cover up near-misses. Blameless postmortems flip this around: they focus on systemic failures and process improvements rather than individual fault. The question changes from "who made this mistake?" to "what allowed this mistake to happen, and how do we prevent it?"

### Security Champions

A security champion is a developer who takes on extra responsibility for security within their team. They are not full-time security engineers --- they are developers who act as a bridge between the security team and the development team. Their role includes:

- Reviewing security-relevant pull requests.
- Staying current on security topics and sharing knowledge.
- Participating in threat modeling sessions.
- Being the first point of contact for security questions.

This model scales far better than having a central security team review everything.

### Making Security Easy

If security practices are painful, people will find workarounds. The goal is to make security the easiest path:

- Provide secure templates and starter projects.
- Automate as much as possible so developers don't have to remember manual steps.
- Give fast feedback. A SAST scan that takes 30 minutes will be ignored; one that takes 30 seconds will be used.
- Celebrate security improvements just as you celebrate feature delivery.

## Conclusion

A DevSecOps maturity model is a compass, not a destination. The value comes from honest self-assessment, setting realistic goals, and making steady progress. Start where you are, pick the dimension where improvement will have the most impact, and build from there. Security is a team sport. The best security cultures are built incrementally, one practice at a time.
