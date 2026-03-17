+++
author = "Adur"
title = "Shift-left security: integrating Trivy and Semgrep in GitLab CI"
date = "2026-03-16"
description = "Practical setup for Trivy and Semgrep in GitLab CI pipelines to catch vulnerabilities in code and dependencies before they reach production."
tags = [
    "shift-left",
    "trivy",
    "semgrep",
    "gitlab-ci",
    "sast",
    "security",
    "ci-cd"
]
categories = [
    "Security",
    "devops"
]
toc = true
+++

## Why these two tools and not others

The security tooling landscape for pipelines is huge, and it is easy to end up with a pipeline that spends twenty minutes just on scans. After trying several combinations, I stick with Trivy and Semgrep for a simple reason: they cover two distinct attack surfaces with minimal friction.

Semgrep analyzes your source code looking for dangerous patterns — SQL injections, insecure deserialization, hardcoded secrets. It does this fast and without needing to compile anything. Trivy, on the other hand, takes care of everything that is not your code: dependencies with known CVEs, outdated base images, problematic IaC configurations. Between the two you cover your own code and third-party code.

Both are open-source, run without an external server, and produce JSON output that you can easily parse in CI. No licenses or third-party dashboards needed to get started.

## Pipeline structure

The idea is that security should not be an isolated stage at the end, but something that runs in parallel with the rest of your checks. Here is the general layout:

```yaml
stages:
  - test
  - security
  - build
  - deploy

variables:
  TRIVY_SEVERITY: "HIGH,CRITICAL"
  SEMGREP_RULES: "p/owasp-top-ten p/security-audit"
```

The `security` stage runs at the same level as `test`. If any scan fails, the pipeline stops before building the image or deploying anything.

## Setting up Semgrep

### Basic job

```yaml
semgrep:
  stage: security
  image: semgrep/semgrep:latest
  script:
    - semgrep ci --config "$SEMGREP_RULES" --json --output semgrep-results.json
  artifacts:
    paths:
      - semgrep-results.json
    when: always
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

This runs Semgrep on every merge request and on every push to the main branch. Results are always saved as an artifact, even if the pipeline fails — you will want to review them later.

### Custom rules

Generic rulesets are fine to start with, but as soon as you have project-specific patterns you want to catch, you will need custom rules. Create a `.semgrep/` directory at the project root:

```yaml
# .semgrep/no-env-secrets.yml
rules:
  - id: no-os-environ-secrets
    patterns:
      - pattern: os.environ[$KEY]
      - metavariable-regex:
          metavariable: $KEY
          regex: ".*(SECRET|PASSWORD|TOKEN|KEY).*"
    message: "Direct access to secrets from environment variables. Use the secrets manager."
    languages: [python]
    severity: WARNING
```

Then reference it in the pipeline:

```yaml
variables:
  SEMGREP_RULES: "p/owasp-top-ten p/security-audit .semgrep/"
```

### Handling false positives

There will be false positives. That is unavoidable. What matters is how you handle them. The worst reaction is to disable the entire rule or slap `allow_failure: true` on the job. Instead, use inline annotations:

```python
# nosemgrep: python.lang.security.audit.hardcoded-password
TEST_PASSWORD = "dummy"  # test fixture, not used in production
```

Every suppression should have a comment explaining why it is safe to ignore. No exceptions. If you cannot justify it, do not suppress it.

For broader suppressions, use `.semgrepignore`:

```
# Exclude test fixtures
tests/fixtures/
# Exclude auto-generated code
*_generated.py
```

## Setting up Trivy

### Dependency scanning

```yaml
trivy-fs:
  stage: security
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  script:
    - trivy fs --severity "$TRIVY_SEVERITY" --exit-code 1 --format json --output trivy-fs.json .
  artifacts:
    paths:
      - trivy-fs.json
    when: always
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

Trivy examines the project's lockfiles (`package-lock.json`, `requirements.txt`, `go.sum`, etc.) and cross-references versions against vulnerability databases. The `--exit-code 1` flag makes the job fail if it finds anything HIGH or CRITICAL.

### Container image scanning

If you build Docker images, scan them before pushing to the registry:

```yaml
trivy-image:
  stage: security
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  needs:
    - job: build-image
      artifacts: true
  script:
    - trivy image --severity "$TRIVY_SEVERITY" --exit-code 1 --format json --output trivy-image.json "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA"
  artifacts:
    paths:
      - trivy-image.json
    when: always
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

This job depends on the image build (via `needs`) and only runs on the main branch, not on every MR. Scanning images is slower than scanning the filesystem, so reserve that for what actually gets deployed.

### IaC scanning

One often overlooked advantage of Trivy is that it also analyzes infrastructure configurations:

```yaml
trivy-iac:
  stage: security
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  script:
    - trivy config --severity "$TRIVY_SEVERITY" --exit-code 1 --format json --output trivy-iac.json .
  artifacts:
    paths:
      - trivy-iac.json
    when: always
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      changes:
        - "**/*.tf"
        - "**/Dockerfile"
        - "**/*.yml"
        - "**/*.yaml"
```

It catches Dockerfiles running as root, Terraform files with overly open security groups, and Kubernetes configs without resource limits. The `changes` block ensures it only runs when relevant files are modified, avoiding unnecessary scans.

## Blocking policies

Do not start blocking everything from day one. That breeds frustration, creative workarounds, and eventually someone puts `allow_failure: true` on every security job.

Better to do it in phases:

```yaml
# Phase 1: Report only
semgrep:
  allow_failure: true

# Phase 2: Block critical only
semgrep:
  script:
    - semgrep ci --config "$SEMGREP_RULES" --json --output semgrep-results.json
    - |
      CRITICAL=$(jq '[.results[] | select(.extra.severity == "ERROR")] | length' semgrep-results.json)
      if [ "$CRITICAL" -gt 0 ]; then
        echo "Blocked: $CRITICAL critical findings"
        exit 1
      fi
  allow_failure: false

# Phase 3: Block high + critical
# ... adjust the jq filter
```

Same goes for Trivy. The `--severity` flag already lets you control which levels block the pipeline. Start with CRITICAL only, and once the team has adapted, add HIGH.

## The complete pipeline

Putting it all together:

```yaml
stages:
  - test
  - security
  - build
  - deploy

variables:
  TRIVY_SEVERITY: "HIGH,CRITICAL"
  SEMGREP_RULES: "p/owasp-top-ten p/security-audit"

semgrep:
  stage: security
  image: semgrep/semgrep:latest
  script:
    - semgrep ci --config "$SEMGREP_RULES" --json --output semgrep-results.json
    - |
      CRITICAL=$(jq '[.results[] | select(.extra.severity == "ERROR")] | length' semgrep-results.json)
      echo "Critical findings: $CRITICAL"
      if [ "$CRITICAL" -gt 0 ]; then
        echo "Pipeline blocked"
        exit 1
      fi
  artifacts:
    paths:
      - semgrep-results.json
    when: always
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

trivy-fs:
  stage: security
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  script:
    - trivy fs --severity "$TRIVY_SEVERITY" --exit-code 1 --format json --output trivy-fs.json .
  artifacts:
    paths:
      - trivy-fs.json
    when: always
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

trivy-config:
  stage: security
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  script:
    - trivy config --severity "$TRIVY_SEVERITY" --exit-code 1 --format json --output trivy-iac.json .
  artifacts:
    paths:
      - trivy-iac.json
    when: always
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      changes:
        - "**/*.tf"
        - "**/Dockerfile"
        - "**/*.yml"

trivy-image:
  stage: security
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  needs:
    - job: build
      artifacts: true
  script:
    - trivy image --severity "$TRIVY_SEVERITY" --exit-code 1 --format json --output trivy-image.json "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA"
  artifacts:
    paths:
      - trivy-image.json
    when: always
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

Semgrep and the Trivy scans run in parallel within the `security` stage. If any of them fails, the pipeline stops before building or deploying.

## Surfacing results in merge requests

JSON reports are fine for auditing, but developers need feedback visible directly in the MR. GitLab supports native security reports if you use official templates, but with external tools you can parse the JSON and comment on the MR:

```yaml
comment-results:
  stage: .post
  image: alpine:latest
  script:
    - apk add --no-cache curl jq
    - |
      SEMGREP_COUNT=$(jq '.results | length' semgrep-results.json 2>/dev/null || echo "0")
      TRIVY_COUNT=$(jq '.Results[]?.Vulnerabilities // [] | length' trivy-fs.json 2>/dev/null || echo "0")

      BODY="### Security summary\n- Semgrep: ${SEMGREP_COUNT} findings\n- Trivy: ${TRIVY_COUNT} vulnerabilities"

      curl --request POST \
        --header "PRIVATE-TOKEN: ${GITLAB_API_TOKEN}" \
        --header "Content-Type: application/json" \
        --data "{\"body\": \"$BODY\"}" \
        "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/merge_requests/${CI_MERGE_REQUEST_IID}/notes"
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
  allow_failure: true
```

Not the most elegant solution, but it works. If you are on GitLab Ultimate you get integrated security reports. If not, this gives enough visibility.

## Cache and performance

Scans add time to the pipeline, and if that time is excessive the team will end up disabling them. A few things that help:

```yaml
trivy-fs:
  variables:
    TRIVY_CACHE_DIR: ".trivycache/"
  cache:
    key: trivy-db
    paths:
      - .trivycache/
  # ...rest of the job
```

Caching Trivy's vulnerability database avoids downloading it on every run. It is roughly 40MB downloaded from GitHub, and on shared runners that download can take a while.

For Semgrep, the binary itself is already fast, but if your repository is large, limit the paths:

```yaml
semgrep:
  script:
    - semgrep ci --config "$SEMGREP_RULES" --include="src/" --include="app/" --json --output semgrep-results.json
```

No point scanning `node_modules/`, `vendor/`, or asset directories.

## What I learned running this in practice

I have been using this setup in production for a while now, and some things only become clear with actual use.

Trivy flags a lot of CVEs in base images that have no fix available. If you do not filter with `--ignore-unfixed`, you will have constant noise. Better to add that flag and focus on what you can actually fix:

```bash
trivy image --ignore-unfixed --severity HIGH,CRITICAL my-image:latest
```

With Semgrep, community rulesets are a good starting point, but the rules that deliver the most value are the ones you write yourself, tailored to your project's patterns. A single rule that catches unparameterized ORM queries is worth more than a hundred generic rules.

And the most important thing: do not try to plug every hole at once. Start with scans in report-only mode, review what comes up, tune the rules, and only then start blocking. If you block everything on day one, someone will have put `allow_failure: true` on every job by day two.
