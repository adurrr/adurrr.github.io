+++
author = "Adur"
title = "How I hardened my K3s homelab"
date = "2023-09-14"
description = "Notes on what I found when auditing my home K3s cluster and the concrete changes I made to harden it."
tags = [
    "k3s",
    "homelab",
    "kubernetes",
    "hardening",
    "security",
    "linux"
]
categories = [
    "Security",
    "Infrastructure"
]
toc = true
+++

## Starting point

I have been running K3s on a couple of physical machines at home for a while. I originally set it up to learn Kubernetes without deploying something heavy, and over time I started using it for small projects: a few web apps, monitoring services, things like that. The setup worked fine but I had never sat down to think carefully about security.

A few months ago I decided to do that exercise. No rush, no generic checklist from the internet. Just look at what I had, understand what was wrong, and fix it.

What I found was not catastrophic, but there were quite a few things I did not like. I am writing this down because K3s has its own quirks compared to a standard Kubernetes cluster, and most hardening guides cover kubeadm or managed clusters, not home setups running on a single binary.

## What makes K3s different

Before getting into what I did, it is worth understanding what makes K3s distinct from a security standpoint.

K3s packages everything in a single binary: API server, scheduler, controller manager, kubelet, kube-proxy, and containerd. Instead of etcd it uses SQLite by default. The default CNI is Flannel. The default ingress controller is Traefik. All of this makes the installation very simple, but it also means there are active components you might not need, and some design decisions favor ease of use over security.

The main configuration file lives at `/etc/rancher/k3s/config.yaml`. That is where most of what I am going to touch ends up.

## The node before the cluster

Before touching anything Kubernetes-related, the operating system. The cluster runs on Debian, so I started there.

### SSH

I had password authentication enabled. That is the first thing to turn off:

```
# /etc/ssh/sshd_config
PasswordAuthentication no
PermitRootLogin no
AllowUsers myuser
```

I also changed the default port. It does not stop someone determined from finding you, but it eliminates a lot of noise in the logs.

### Firewall with nftables

The machine had its network interfaces completely open. With a home K3s cluster, the API server (port 6443) should not be reachable from outside your local network. My basic rules with nftables:

```bash
# List active ruleset
nft list ruleset

# Restrictive default policy
nft add chain inet filter input { type filter hook input priority 0 \; policy drop \; }

# Allow loopback and established connections
nft add rule inet filter input iifname lo accept
nft add rule inet filter input ct state established,related accept

# SSH only from local network
nft add rule inet filter input ip saddr 192.168.1.0/24 tcp dport 22 accept

# K3s API server only from local network
nft add rule inet filter input ip saddr 192.168.1.0/24 tcp dport 6443 accept

# Kubernetes node ports
nft add rule inet filter input tcp dport 30000-32767 accept

# Traefik (if used as ingress)
nft add rule inet filter input tcp dport { 80, 443 } accept

# ICMP
nft add rule inet filter input icmp type echo-request accept
```

What surprised me when reviewing this: the API server was listening on all interfaces and was reachable directly from the WAN because the router had a port forward for another service that was dragging along some traffic. Small scare, nothing exploited, but not something I wanted to leave.

### Kernel parameters

K3s with the `--protect-kernel-defaults` flag verifies that certain kernel parameters are set correctly. If they are not, startup fails with a clear message. Better to configure them beforehand:

```bash
# /etc/sysctl.d/90-k3s-hardening.conf
kernel.panic = 10
kernel.panic_on_oops = 1
vm.overcommit_memory = 1
vm.panic_on_oom = 0
fs.inotify.max_user_watches = 524288
fs.inotify.max_user_instances = 512
```

```bash
sysctl --system
```

## K3s configuration

With the node in order, onto the cluster.

### Base configuration file

```yaml
# /etc/rancher/k3s/config.yaml
write-kubeconfig-mode: "0600"
protect-kernel-defaults: true
secrets-encryption: true
kube-apiserver-arg:
  - "anonymous-auth=false"
  - "audit-log-path=/var/log/k3s/audit.log"
  - "audit-log-maxage=30"
  - "audit-policy-file=/etc/rancher/k3s/audit-policy.yaml"
kube-controller-manager-arg:
  - "terminated-pod-gc-threshold=10"
kubelet-arg:
  - "streaming-connection-idle-timeout=5m"
  - "protect-kernel-defaults=true"
  - "make-iptables-util-chains=true"
```

The three most important lines here:

- `secrets-encryption: true` enables encryption at rest for Kubernetes secrets. In K3s this is a first-class flag, no need to configure `EncryptionConfiguration` manually like in kubeadm.
- `anonymous-auth=false` removes anonymous access to the API server.
- `write-kubeconfig-mode: "0600"` enforces restrictive permissions on the kubeconfig that K3s generates at `/etc/rancher/k3s/k3s.yaml`.

### Audit policy

Without audit logs you have no idea what is happening in your cluster. This is a reasonable minimum:

```yaml
# /etc/rancher/k3s/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  - level: Metadata
    resources:
      - group: ""
        resources: ["secrets"]
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["pods/exec", "pods/attach"]
  - level: Request
    verbs: ["create", "update", "patch", "delete"]
  - level: None
    users: ["system:kube-proxy"]
    verbs: ["watch"]
    resources:
      - group: ""
        resources: ["endpoints", "services", "services/status"]
  - level: Metadata
    omitStages:
      - RequestReceived
```

The logs go to `/var/log/k3s/audit.log`. In my case I collect them with Promtail and send them to Loki, so I can run queries from Grafana when I need to review something.

### Kubeconfig

The kubeconfig K3s generates at `/etc/rancher/k3s/k3s.yaml` has admin credentials. A mistake I spotted in my own setup: I had that file copied to `~/.kube/config` with 644 permissions. Any process running under my user could read it.

```bash
# Correct permissions
chmod 600 ~/.kube/config
```

If you have additional users who need cluster access, create ServiceAccounts or use client certificates with limited RBAC. Do not distribute the admin kubeconfig.

## CNI: from Flannel to Cilium

Flannel is the K3s default. It works, but it does not support NetworkPolicies out of the box. That means all pods can communicate with each other without restriction.

I switched to Cilium. The process with K3s requires disabling Flannel first:

```yaml
# /etc/rancher/k3s/config.yaml (add)
flannel-backend: none
disable-network-policy: true
```

Then install Cilium with Helm:

```bash
helm repo add cilium https://helm.cilium.io/

helm install cilium cilium/cilium \
  --namespace kube-system \
  --set operator.replicas=1 \
  --set ipam.mode=kubernetes \
  --set kubeProxyReplacement=strict \
  --set k8sServiceHost=192.168.1.10 \
  --set k8sServicePort=6443
```

With `kubeProxyReplacement=strict`, Cilium also replaces kube-proxy, which gives better performance with eBPF and better network traffic visibility.

After the switch I started applying NetworkPolicies. The first and most important one: default deny in all application namespaces:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: apps
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

From there, explicitly allowing only what is necessary.

## Secrets: what to do when it is just K3s

In a corporate environment you would use Vault or External Secrets Operator backed by AWS Secrets Manager or similar. In a homelab that is overkill. My solution was simpler.

K3s's encryption at rest already protects secrets in the database. For manifests in Git I use `age` to encrypt sensitive values before committing them:

```bash
# Encrypt a value
echo "my-real-password" | age -r $(cat ~/.config/age/recipient.txt) | base64
```

The encrypted value goes into the repository. Deploying it requires a manual decryption step. Less convenient than an automatic operator, but for a homelab it is good enough and does not add operational complexity.

## Falco for runtime detection

Everything above is prevention. Falco is detection: it monitors system calls and generates alerts when something suspicious happens inside a container.

I installed it with Helm:

```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm install falco falcosecurity/falco \
  --namespace falco \
  --create-namespace \
  --set driver.kind=ebpf
```

The default rules already cover quite a bit: shell execution in containers, writes to system directories, privilege changes, unexpected network connections. I added some custom rules for my specific use case.

Alerts go to Loki as well. That way I have the API server audit logs and Falco alerts in the same place.

## Pod Security Standards

With K3s >= 1.25 you can use Pod Security Admission directly. I applied the `baseline` level on application namespaces and `restricted` where possible:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: apps
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/warn: restricted
```

The `restricted` level is strict: containers cannot run as root, the root filesystem must be read-only, and all Linux capabilities must be dropped. Some of the workloads I had deployed did not comply. I fixed them one by one before raising the `enforce` level.

## What I left for later

Not everything is perfect. There are things I know I should do that I have not gotten to yet.

**Rootless K3s**: you can run K3s entirely without root, which reduces the impact of a potential privilege escalation. I have looked into it and it has some limitations with certain network features, but for my use case it should work.

**Image verification with cosign**: signing and verifying images before deploying them. I have a private registry for my own images and the third-party ones I use, but I am not verifying signatures yet.

**CIS Benchmark**: K3s has official documentation for the CIS K3s benchmark. I have reviewed it partially but have not run kube-bench systematically to see what is still pending.

## What I learned

The default K3s installation is surprisingly reasonable for what it is: a distribution designed to be easy to deploy. But "reasonable" is not the same as "hardened."

What surprised me most was how easy most of the changes turned out to be. K3s has first-class flags for many things that require manual configuration in kubeadm. The `secrets-encryption`, `protect-kernel-defaults`, the audit policy directly in the config file. It is not that hard if you sit down and read the documentation.

The biggest risk in a homelab is not that someone on the internet attacks you directly. It is that you have a bunch of services running on the same network as your personal life, and if one gets compromised, the blast radius can be larger than it seems. Worth taking seriously even if it is a toy environment.
