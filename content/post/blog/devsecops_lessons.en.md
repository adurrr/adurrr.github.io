+++
author = "Adur"
title = "Lessons learned in DevSecOps"
date = "2025-10-20"
description = "Practical lessons from working in DevSecOps: what actually works, what does not, and what we wish we had known sooner."
featured = true
tags = [
    "devsecops",
    "lessons-learned",
    "culture",
    "automation",
    "security",
]
categories = [
    "Blog",
    "devops",
]
thumbnail = "images/building.png"
toc = true
+++

DevSecOps gets thrown around in job descriptions and conference talks a lot. But behind the buzzword are real lessons that only come from doing the work. From building pipelines that break when you add security gates, to watching teams ignore the tools you spent months deploying, to finally finding what actually works.

These are lessons we learned the hard way. They're opinionated, practical, shaped by experience.

<!--more-->

## Security is everyone's responsibility

Sounds like a break room poster, but it's the most important lesson here. If security is only the security team's job, you've lost.

Developers make security decisions every time they write code, whether they know it or not. How they validate input. How they handle secrets. How they configure network access. Every PR is a security event.

What works: make security part of the normal development workflow, not a gate at the end. Developers learn when they get fast feedback on security issues in their PR. They resent finding out three weeks later from an auditor.

We've seen this repeatedly: teams that treat security as shared responsibility find fewer critical vulnerabilities in production. Teams that silo it find them in the news.

## Automate everything you can

Manual security processes do not scale. Period. If your security review is a human reading a checklist, it will be skipped under deadline pressure, inconsistently applied, and resented by everyone involved.

Automate the things that can be automated:

- **Dependency scanning** in every CI build (Dependabot, Snyk, Trivy)
- **Static analysis** on every pull request (Semgrep, SonarQube)
- **Secret detection** as a pre-commit hook and CI check (gitleaks, detect-secrets)
- **Container image scanning** before deployment (Trivy, Grype)
- **Infrastructure as Code scanning** (tfsec, Checkov, KICS)
- **Compliance as Code** for runtime policy enforcement (OPA, Kyverno)

The goal is not to catch everything automatically. The goal is to catch the easy stuff automatically so that human reviewers can focus on the hard stuff: business logic flaws, design-level security issues, threat modeling.

## Start Small

One of the biggest mistakes we have made is trying to secure everything at once. You roll out SAST, DAST, SCA, container scanning, IaC scanning, and runtime protection in one quarter. The result? Alert fatigue, developer rebellion, and a wall of unresolved findings that nobody looks at.

Start with one tool, one pipeline, one team. Get it working well. Get developers comfortable with it. Resolve the false positives. Tune the rules. Then expand.

A practical progression:

1. **Month 1**: Secret detection in pre-commit hooks and CI. This is uncontroversial and catches real issues.
2. **Month 2**: Dependency scanning with automated PR creation for updates. Developers see the value immediately.
3. **Month 3**: Container image scanning blocking deployments of critical/high vulnerabilities.
4. **Month 4+**: Static analysis, gradually expanding rule sets.

Each step should be stable before moving to the next. Rushing creates noise, and noise teaches people to ignore alerts.

## Blameless culture matters

When a security incident happens because someone pushed a secret to a public repo, or because a vulnerability was not patched in time, the response matters more than the incident itself.

If people get blamed, they hide things. They do not report near-misses. They cover up mistakes. And the next incident will be worse because nobody shared the lessons from the last one.

Blameless postmortems are not about letting people off the hook. They are about understanding systemic failures. Why was it possible to push a secret? Why was there no scanning? Why was the patching process slow? Fix the system, not the person.

We have found that teams with genuinely blameless cultures have significantly better security postures. People report suspicious things. They ask for help early. They flag risks before they become incidents.

## Tooling is not enough without culture change

We once deployed a comprehensive security scanning pipeline with beautiful dashboards, Slack notifications, Jira ticket creation, the works. Six months later, there were 3,000 unresolved findings and the Slack channel was muted by every developer.

The tools were fine. The culture was not ready.

Before you deploy tooling, invest in:

- **Training**: Developers need to understand why the tool exists and how to act on its findings.
- **Ownership**: Someone needs to own the backlog of findings and triage them. If nobody owns it, nobody does it.
- **SLAs**: Define clear timelines for remediating findings by severity. Critical gets 48 hours. High gets a week. Medium gets a sprint. Low gets a quarter.
- **Feedback loops**: When a tool produces a false positive, there must be an easy way to report it and get the rule tuned. Otherwise, developers learn to ignore everything.

## Invest in developer experience for security tools

If your security tool makes developers' lives harder, they will find a way around it. This is not a character flaw. It is human nature and good engineering instinct: remove obstacles to shipping.

The security tools that get adopted are the ones that:

- **Run fast**: A SAST scan that takes 20 minutes will be bypassed. One that takes 30 seconds will be tolerated.
- **Integrate natively**: Show results in the PR, not in a separate portal. Nobody wants to log into another dashboard.
- **Have low false positive rates**: Every false positive erodes trust. Invest time in tuning.
- **Provide actionable guidance**: "SQL injection vulnerability on line 42" is useless without "here is how to fix it."
- **Fail gracefully**: If the scanner is down, the pipeline should warn, not block. Availability of the development pipeline is non-negotiable.

We think of it this way: if a developer has to change their workflow to accommodate a security tool, the tool has failed. The best security tooling is invisible.

## Monitoring and observability are non-negotiable

You cannot secure what you cannot see. Security monitoring is not optional, and it is not something you bolt on after the fact.

What this means in practice:

- **Centralized logging**: All application, infrastructure, and security tool logs in one place. If you have to SSH into a box to read logs, you are already behind.
- **Audit trails**: Who did what, when, and from where. Every deployment, every config change, every access request.
- **Alerting on anomalies**: Not just "is the service up?" but "is this access pattern normal?" Unusual API call volumes, access from new locations, privilege escalations.
- **Runtime security**: Tools like Falco for container runtime monitoring. Know when something unexpected happens in production.

Monitoring is also how you prove to auditors and customers that your security controls are working. "Trust us" is not a compliance strategy.

## Open source is your ally

Some of the best security tools available are open source. Trivy, Falco, OPA, Semgrep, gitleaks, cosign, KICS, Checkov. The ecosystem is rich and maturing fast.

Benefits of open source security tooling:

- **Transparency**: You can read the rules and understand exactly what is being checked.
- **Community**: Thousands of contributors finding edge cases and adding detection rules.
- **No vendor lock-in**: You can switch tools without renegotiating a contract.
- **Cost**: Start for free, scale as needed.

This does not mean commercial tools have no place. Some provide valuable aggregation, management, and support. But you can build a very solid security pipeline with open source tools alone, and we think every team should start there.

## Continuous learning is essential

The threat landscape changes constantly. The tools change. The best practices evolve. What was considered secure two years ago might have a CVE today.

What we do to stay current:

- **Dedicate time for learning**: At least a few hours per sprint for the team to read about new vulnerabilities, tools, and techniques. This is not a nice-to-have. It is a professional requirement.
- **Run internal CTFs and tabletop exercises**: Nothing teaches security like trying to break things. Regular exercises keep skills sharp and reveal gaps in your defenses.
- **Participate in the community**: Attend meetups, contribute to open source, read advisories. The security community is generous with knowledge. Take advantage of it.
- **Review and update**: Quarterly reviews of your security tooling, policies, and incident response procedures. What worked last quarter may not work next quarter.

## Final Thoughts

DevSecOps isn't a destination. There's no point where you say "we're done, we're secure." It's a continuous practice of reducing risk, improving visibility, building a culture where security is as natural as writing tests.

The most important lesson: perfect is the enemy of good. A basic security pipeline that developers actually use beats a comprehensive one they bypass. Start where you are, improve iteratively, never stop.
