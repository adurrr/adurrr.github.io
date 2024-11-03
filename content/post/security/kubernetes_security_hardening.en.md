+++
author = "Adur"
title = "Kubernetes security hardening guide"
date = "2024-11-03"
description = "Comprehensive guide to hardening Kubernetes clusters covering RBAC, Pod Security Standards, NetworkPolicies, secrets management, audit logging, and image policies."
tags = [
    "kubernetes",
    "security",
    "hardening",
    "rbac",
    "network-policies",
    "pod-security"
]
categories = [
    "Security",
    "Infrastructure"
]
toc = true
+++

## Kubernetes Attack Surface

Out-of-the-box Kubernetes isn't secure. The cluster exposes multiple attack vectors that need systematic attention:

- **API Server** - Central control point; unauthorized access = full cluster compromise
- **etcd** - Stores all cluster state including secrets in base64 (not encrypted by default)
- **Kubelet** - Node agent; exposed kubelet API leaks pod info and allows command execution
- **Container runtime** - Breakout vulnerabilities give host access
- **Network** - Pods can communicate freely with any other pod by default
- **Supply chain** - Malicious or vulnerable images introduce backdoors

This guide covers the key hardening measures organized by attack vector.

## API Server Hardening

The API server is the most critical component. Secure it first:

- **Disable anonymous authentication**: set `--anonymous-auth=false`
- **Enable audit logging**: track who did what (covered in detail below)
- **Restrict access**: use firewall rules or security groups to limit who can reach the API server
- **Enable admission controllers**: `PodSecurity`, `NodeRestriction`, `ResourceQuota`, and `LimitRanger` should all be active
- **Use OIDC for authentication**: integrate with your identity provider instead of relying on client certificates

```bash
# Check current API server flags (on kubeadm clusters)
kubectl -n kube-system get pod kube-apiserver-<node> -o jsonpath='{.spec.containers[0].command}' | tr ',' '\n'
```

## RBAC Best Practices

RBAC is your primary authorization mechanism. Enforce least privilege rigorously.

### Key Rules

1. **Never use `cluster-admin` for workloads** -- it grants unlimited access
2. **Use Roles (namespaced) over ClusterRoles** whenever possible
3. **Avoid wildcards** in resource or verb specifications
4. **Bind to ServiceAccounts, not users** for automated workloads
5. **Review bindings regularly** -- permissions accumulate over time

### Example: Restricted Deployment Role

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: deployment-manager
rules:
  # Can manage deployments
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  # Can view pods and logs
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
  # Can view services
  - apiGroups: [""]
    resources: ["services"]
    verbs: ["get", "list", "watch"]
  # Explicitly NO access to secrets, configmaps/write, or exec
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: production
  name: deployment-manager-binding
subjects:
  - kind: ServiceAccount
    name: ci-deployer
    namespace: production
roleRef:
  kind: Role
  name: deployment-manager
  apiGroup: rbac.authorization.k8s.io
```

Audit existing RBAC with:

```bash
# List all cluster-admin bindings
kubectl get clusterrolebindings -o json | \
  jq '.items[] | select(.roleRef.name == "cluster-admin") | {name: .metadata.name, subjects: .subjects}'

# Check what a specific service account can do
kubectl auth can-i --list --as=system:serviceaccount:production:ci-deployer -n production
```

## Pod Security Standards

Kubernetes Pod Security Standards (PSS) define three security restriction levels. Since 1.25, Pod Security Admission (PSA) is the built-in enforcement mechanism (replaces PodSecurityPolicy).

### The Three Levels

| Level | Description | Use Case |
|-------|------------|----------|
| **Privileged** | No restrictions | System-level workloads (CNI, storage drivers) |
| **Baseline** | Blocks known privilege escalations | General-purpose workloads |
| **Restricted** | Maximum hardening | Sensitive workloads, multi-tenant clusters |

### Enforcing Pod Security Standards

Apply standards at the namespace level using labels:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    # Enforce restricted: reject pods that violate
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    # Warn on baseline violations (shows warning but allows)
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: latest
    # Audit: log violations
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: latest
```

### Compliant Pod Example

A pod that passes the `restricted` level:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
  namespace: production
spec:
  securityContext:
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      image: myregistry.com/app:v1.2.3@sha256:abc123...
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        runAsUser: 1000
        runAsGroup: 1000
        capabilities:
          drop:
            - ALL
      resources:
        limits:
          memory: "256Mi"
          cpu: "500m"
        requests:
          memory: "128Mi"
          cpu: "250m"
```

Key security settings to always include:

- `runAsNonRoot: true` -- never run containers as root
- `readOnlyRootFilesystem: true` -- prevent filesystem modifications
- `allowPrivilegeEscalation: false` -- block setuid/setgid
- `capabilities.drop: ["ALL"]` -- remove all Linux capabilities
- `seccompProfile.type: RuntimeDefault` -- apply default seccomp profile
- Always specify resource limits to prevent DoS

## NetworkPolicies

By default, all pods can communicate with all other pods in a cluster. NetworkPolicies restrict this to only the traffic that is necessary.

### Default Deny All

Start every namespace with a default deny policy:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

### Allow Specific Traffic

Then explicitly allow only what is needed:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-to-db
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: api-server
      ports:
        - protocol: TCP
          port: 5432
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-server-egress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api-server
  policyTypes:
    - Egress
  egress:
    # Allow DNS resolution
    - to: []
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
    # Allow connection to database
    - to:
        - podSelector:
            matchLabels:
              app: database
      ports:
        - protocol: TCP
          port: 5432
    # Allow external HTTPS
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 10.0.0.0/8
              - 172.16.0.0/12
              - 192.168.0.0/16
      ports:
        - protocol: TCP
          port: 443
```

Note: NetworkPolicies require a CNI plugin that supports them (Calico, Cilium, Weave Net). The default `kubenet` does not enforce NetworkPolicies.

## Secrets Management

Kubernetes Secrets are base64-encoded, not encrypted. Anyone with read access to Secrets in a namespace can decode them trivially. Proper secrets management requires additional tooling.

### Option 1: Sealed Secrets

Sealed Secrets (by Bitnami) encrypts secrets client-side so they can be safely stored in Git:

```bash
# Install kubeseal CLI
# Encrypt a secret
kubectl create secret generic db-creds \
  --from-literal=password=supersecret \
  --dry-run=client -o yaml | \
  kubeseal --format yaml > sealed-db-creds.yaml

# The sealed secret can be committed to Git
# Only the cluster's controller can decrypt it
kubectl apply -f sealed-db-creds.yaml
```

### Option 2: External Secrets Operator

External Secrets Operator syncs secrets from external providers (AWS Secrets Manager, HashiCorp Vault, GCP Secret Manager) into Kubernetes:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: db-credentials
  data:
    - secretKey: password
      remoteRef:
        key: production/database
        property: password
```

### Enable Encryption at Rest

Ensure etcd encrypts secrets at rest:

```yaml
# /etc/kubernetes/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <base64-encoded-32-byte-key>
      - identity: {}
```

## Audit Logging

Kubernetes audit logs record every request to the API server, providing a detailed trail of who did what and when.

```yaml
# audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Log all requests to secrets at the Metadata level
  - level: Metadata
    resources:
      - group: ""
        resources: ["secrets"]
  # Log pod exec/attach at RequestResponse level
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["pods/exec", "pods/attach"]
  # Log all write operations at Request level
  - level: Request
    verbs: ["create", "update", "patch", "delete"]
  # Log everything else at Metadata level
  - level: Metadata
    omitStages:
      - RequestReceived
```

Enable in the API server with:

```
--audit-policy-file=/etc/kubernetes/audit-policy.yaml
--audit-log-path=/var/log/kubernetes/audit.log
--audit-log-maxage=30
--audit-log-maxbackup=10
--audit-log-maxsize=100
```

Ship audit logs to your SIEM or log aggregation system (Loki, Elasticsearch) for analysis and alerting.

## Image Policies with Kyverno

Kyverno is a policy engine for Kubernetes that validates, mutates, and generates resources based on policies. It is simpler to adopt than OPA Gatekeeper because policies are written as Kubernetes resources rather than Rego.

### Require Image Digest and Trusted Registry

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-image-digest
spec:
  validationFailureAction: Enforce
  rules:
    - name: require-digest
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        message: "Images must use a digest (@sha256:...) instead of a tag"
        pattern:
          spec:
            containers:
              - image: "*@sha256:*"
    - name: require-trusted-registry
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        message: "Images must come from the trusted registry"
        pattern:
          spec:
            containers:
              - image: "myregistry.com/*"
```

This policy ensures that only images from your trusted registry with pinned digests are deployed, preventing both supply chain attacks and tag mutability issues.

## CIS Kubernetes Benchmark

The CIS Kubernetes Benchmark provides a comprehensive set of security recommendations. Run automated checks with kube-bench:

```bash
# Run CIS benchmark checks
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml

# View results
kubectl logs job/kube-bench
```

Address findings by priority: critical items first (API server auth, etcd encryption), then high (RBAC, network policies), then medium and low.

## Hardening Checklist

Use this checklist as a starting point for securing your cluster:

- [ ] API server: anonymous auth disabled, audit logging enabled
- [ ] etcd: encrypted at rest, access restricted to API server only
- [ ] RBAC: no unnecessary `cluster-admin` bindings, least privilege enforced
- [ ] Pod Security: `restricted` PSS enforced on production namespaces
- [ ] NetworkPolicies: default deny in all namespaces, explicit allow rules
- [ ] Secrets: external secrets manager or sealed secrets, encryption at rest
- [ ] Images: signed images, trusted registry enforcement, digest pinning
- [ ] Nodes: automatic security updates, CIS-hardened OS
- [ ] Audit: API server audit logs shipped to SIEM
- [ ] Monitoring: alerts on RBAC changes, privileged pod creation, exec into pods
- [ ] Supply chain: vulnerability scanning in CI/CD, admission-time image scanning
- [ ] Network: TLS between all components, service mesh for mTLS between pods

Security hardening is not a one-time activity. Schedule quarterly reviews to reassess your posture, run CIS benchmarks, review RBAC bindings, and update policies as your cluster evolves.
