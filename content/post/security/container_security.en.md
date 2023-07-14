---
title: "Container security best practices"
date: 2023-07-14T10:00:00+01:00
draft: false
tags:
  - "docker"
  - "kubernetes"
  - "security"
  - "containers"
  - "trivy"
  - "falco"

categories:
  - "Security"
  - "devops"

toc: true
description: "Best practices for securing Docker and Kubernetes containers across the full lifecycle, from build to runtime."
---

## The Attack Surface of Containers

Containers give you process isolation, not security isolation. A misconfigured container can expose the host kernel, leak secrets, or become a pivot point for lateral movement. Understand the attack surface first:

- **Container images** - Vulnerable libraries, hardcoded credentials, unnecessary tools
- **Container runtime** - Vulnerabilities that allow breakouts
- **Orchestrator misconfigurations** (Kubernetes RBAC, network policies) - Expose services, grant excessive permissions
- **Supply chain** - Compromised base images or dependencies
- **Secrets** - Baked into images or environment variables that any process can access

Security requires defense in depth across the entire lifecycle: build, ship, run.

## Image security

### Use minimal base images

Every additional package in a container image is a potential vulnerability. Prefer minimal base images:

```dockerfile
# Prefer distroless or Alpine over full distributions
FROM gcr.io/distroless/static-debian11
# or
FROM alpine:3.18
```

**Distroless images** contain only your application and its runtime dependencies --- no shell, no package manager, no unnecessary binaries. This dramatically reduces the attack surface.

### Multi-stage builds

Multi-stage builds let you compile in one stage and copy only the final artifact to a minimal runtime image:

```dockerfile
# Build stage
FROM golang:1.21 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /app/server

# Runtime stage
FROM gcr.io/distroless/static-debian11
COPY --from=builder /app/server /server
USER nonroot:nonroot
ENTRYPOINT ["/server"]
```

The build tools, source code, and intermediate files never make it into the final image.

### Never run as root

Running containers as root means a container escape gives the attacker root on the host. Always specify a non-root user:

```dockerfile
# Create a non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
```

In Kubernetes, enforce this with Pod Security Standards (more on this below).

### Secure vs. insecure Dockerfile comparison

Here is a side-by-side comparison of common mistakes versus secure practices:

```dockerfile
# INSECURE - Do NOT do this
FROM ubuntu:latest
RUN apt-get update && apt-get install -y curl wget vim
COPY . /app
RUN echo "DB_PASSWORD=supersecret" > /app/.env
EXPOSE 22 80 443
USER root
CMD ["python3", "/app/main.py"]
```

```dockerfile
# SECURE - Follow this pattern
FROM python:3.11-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

FROM python:3.11-slim
RUN groupadd -r appgroup && useradd -r -g appgroup appuser
WORKDIR /app
COPY --from=builder /root/.local /home/appuser/.local
COPY --chown=appuser:appgroup . .
ENV PATH=/home/appuser/.local/bin:$PATH
EXPOSE 8080
USER appuser
HEALTHCHECK --interval=30s --timeout=3s CMD curl -f http://localhost:8080/health || exit 1
CMD ["python3", "main.py"]
```

Key differences: minimal base image, multi-stage build, no secrets in the image, non-root user, only necessary ports exposed, health check included.

## Image scanning with Trivy

Trivy is an open-source vulnerability scanner that checks container images, filesystems, and IaC configurations. Integrate it into your CI pipeline to catch vulnerabilities before deployment.

### Basic image scan

```bash
# Scan an image for vulnerabilities
trivy image python:3.11-slim

# Scan with severity filter
trivy image --severity HIGH,CRITICAL myapp:latest

# Fail CI if critical vulnerabilities are found
trivy image --exit-code 1 --severity CRITICAL myapp:latest
```

### Scanning in CI/CD

Add Trivy to your pipeline so that no image with critical vulnerabilities reaches production:

```yaml
# Example GitHub Actions step
- name: Run Trivy vulnerability scanner
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: 'myapp:${{ github.sha }}'
    format: 'table'
    exit-code: '1'
    severity: 'CRITICAL,HIGH'
```

### Scanning IaC and filesystems

Trivy goes beyond container images:

```bash
# Scan Kubernetes manifests for misconfigurations
trivy config ./k8s-manifests/

# Scan a filesystem for secrets and vulnerabilities
trivy fs --security-checks vuln,secret ./
```

## Runtime security with Falco

While scanning catches vulnerabilities at build time, Falco monitors containers at runtime. It uses kernel-level instrumentation to detect suspicious behavior:

- Unexpected shell spawns inside containers.
- Processes reading sensitive files (`/etc/shadow`, `/etc/passwd`).
- Outbound network connections to unexpected destinations.
- Privilege escalation attempts.

### Example Falco rules

```yaml
- rule: Terminal shell in container
  desc: Detect a shell being spawned in a container
  condition: >
    spawned_process and container and
    proc.name in (bash, sh, zsh, dash)
  output: >
    Shell spawned in container
    (user=%user.name container=%container.name
    shell=%proc.name parent=%proc.pname)
  priority: WARNING

- rule: Read sensitive file in container
  desc: Detect reads of sensitive files
  condition: >
    open_read and container and
    fd.name in (/etc/shadow, /etc/passwd)
  output: >
    Sensitive file read in container
    (user=%user.name file=%fd.name container=%container.name)
  priority: ERROR
```

Deploy Falco as a DaemonSet in your Kubernetes cluster to get visibility into runtime behavior across all nodes.

## Kubernetes security

### Pod Security Standards

Kubernetes Pod Security Standards define three levels of restriction:

- **Privileged** --- Unrestricted (for system-level workloads only).
- **Baseline** --- Prevents known privilege escalations.
- **Restricted** --- Heavily restricted, following security best practices.

Apply them at the namespace level:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

### NetworkPolicies

By default, all pods in Kubernetes can communicate with each other. NetworkPolicies let you restrict traffic:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-allow
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: database
      ports:
        - protocol: TCP
          port: 5432
```

This policy ensures the API pod only receives traffic from the frontend and only sends traffic to the database.

### RBAC

Follow the principle of least privilege. Avoid giving `cluster-admin` to service accounts. Create specific roles:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: deployment-manager
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "update", "patch"]
```

Regularly audit RBAC bindings with tools like `kubectl-who-can` or `rbac-tool`.

## Secrets management

Never store secrets in container images, environment variables in plain Dockerfiles, or version control. Instead:

- Use **Kubernetes Secrets** (encrypted at rest with KMS).
- Use external secrets managers: **HashiCorp Vault**, **AWS Secrets Manager**, **Azure Key Vault**.
- Use the **External Secrets Operator** to sync secrets from external providers into Kubernetes.
- Rotate secrets regularly and audit access.

## Supply chain security

### Signed images

Sign your container images with **cosign** (part of the Sigstore project) and verify signatures before deployment:

```bash
# Sign an image
cosign sign --key cosign.key myregistry/myapp:v1.0

# Verify a signature
cosign verify --key cosign.pub myregistry/myapp:v1.0
```

### SBOM (Software Bill of Materials)

Generate an SBOM for every image so you can quickly check whether a newly disclosed CVE affects your running containers:

```bash
# Generate SBOM with Trivy
trivy image --format spdx-json --output sbom.json myapp:latest

# Or use syft
syft myapp:latest -o spdx-json > sbom.json
```

## Container security checklist

Use this checklist to assess your container security posture:

- [ ] Base images are minimal (distroless or Alpine).
- [ ] Multi-stage builds are used to exclude build tools.
- [ ] Containers run as non-root users.
- [ ] Images are scanned for vulnerabilities in CI/CD.
- [ ] No secrets are stored in images or plain environment variables.
- [ ] Kubernetes Pod Security Standards are enforced.
- [ ] NetworkPolicies restrict pod-to-pod communication.
- [ ] RBAC follows least privilege.
- [ ] Runtime security monitoring is in place (Falco or equivalent).
- [ ] Images are signed and signatures are verified before deployment.
- [ ] SBOMs are generated and stored for all production images.
- [ ] Secrets are managed through an external secrets manager.
- [ ] Image pull policies are set to `Always` for mutable tags.
- [ ] Regular security audits and penetration tests are conducted.

## Conclusion

Container security is an ongoing practice, not a one-time task. Start with basics: minimal images, non-root users, scanning. Then layer on runtime monitoring, network policies, supply chain verification, and automated enforcement. Each layer shrinks the blast radius and moves you closer to actual security.
