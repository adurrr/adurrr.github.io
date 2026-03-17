+++
author = "Adur"
title = "Building an IaC repository for the homelab"
date = "2023-11-20"
description = "How I organized my homelab infrastructure into a Git repository with Terraform, Ansible, Kubernetes manifests, and OPA/Gatekeeper policies."
tags = [
    "homelab",
    "iac",
    "terraform",
    "ansible",
    "kubernetes",
    "opa",
    "gatekeeper",
    "seccomp"
]
categories = [
    "devops",
    "sysadmin"
]
toc = true
+++

## What triggered this

A couple of months ago I spent time hardening the K3s cluster. I went through an entire weekend changing configurations, tuning kernel parameters, swapping Flannel for Cilium, writing network policies. By the end the cluster was in much better shape.

But I had done all of it by hand.

If I need to recreate that node from scratch tomorrow, how long does it take? Probably two or three days digging through my own notes scattered across text files, terminal history, and chat messages. And I would still miss things. That is not sustainable.

So I decided to build a proper infrastructure repository. Not a demo, not a proof of concept — the repository where the definition of everything I run at home lives, sanitized enough to publish.

## What the homelab looks like

Before talking about the repository structure, it helps to explain what needs to be managed. The setup is:

- A main server running Proxmox where several VMs live
- Two additional physical nodes forming the K3s cluster
- A router running OpenWrt
- A NAS running TrueNAS

It is not a huge environment, but it has enough variety that managing everything by hand is a real problem. Especially because Proxmox, the VMs, the cluster, and the NAS have configurations that interact with each other: IPs, internal DNS, certificates, users.

## Repository structure

After a few experiments, this is the layout that works for me:

```
homelab-infra/
├── terraform/
│   ├── modules/
│   │   ├── proxmox-vm/
│   │   ├── dns-record/
│   │   └── network-vlan/
│   └── environments/
│       ├── main.tf
│       ├── variables.tf
│       └── terraform.tfvars.example
├── ansible/
│   ├── inventory/
│   │   ├── hosts.yml
│   │   └── group_vars/
│   ├── playbooks/
│   │   ├── bootstrap.yml
│   │   ├── k3s-server.yml
│   │   ├── k3s-agent.yml
│   │   └── hardening.yml
│   └── roles/
│       ├── common/
│       ├── cis-level1/
│       └── k3s/
├── kubernetes/
│   ├── base/
│   ├── apps/
│   │   ├── monitoring/
│   │   ├── storage/
│   │   └── networking/
│   └── policies/
│       ├── gatekeeper/
│       └── seccomp/
├── .gitlab-ci.yml
├── .sops.yaml
└── README.md
```

Three clearly separated layers: provisioning (Terraform), node configuration (Ansible), and Kubernetes workloads. Security policies live inside `kubernetes/policies/` because they are Kubernetes resources, but I treat them as a conceptually distinct layer.

## Terraform: provisioning with Proxmox

The Proxmox provider for Terraform is the Telmate one (`telmate/proxmox`). It is not official, but it is the most widely used and works reasonably well.

The `proxmox-vm` module wraps VM creation with the parameters I use regularly:

```hcl
# terraform/modules/proxmox-vm/main.tf
resource "proxmox_vm_qemu" "vm" {
  name        = var.name
  target_node = var.target_node
  clone       = var.template
  full_clone  = true

  cores   = var.cores
  memory  = var.memory
  sockets = 1

  disk {
    size    = var.disk_size
    type    = "virtio"
    storage = var.storage_pool
    discard = "on"
  }

  network {
    model  = "virtio"
    bridge = var.network_bridge
    tag    = var.vlan_tag
  }

  ipconfig0    = "ip=${var.ip_address}/24,gw=${var.gateway}"
  nameserver   = var.nameserver
  searchdomain = var.searchdomain

  ciuser  = var.ssh_user
  sshkeys = var.ssh_public_key

  lifecycle {
    ignore_changes = [
      network,
    ]
  }
}
```

Sensitive variables — the Proxmox API token, SSH keys — are not in the repository in plaintext. I use SOPS to encrypt the `terraform.tfvars` file with age:

```yaml
# .sops.yaml
creation_rules:
  - path_regex: .*\.tfvars$
    age: age1xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
  - path_regex: ansible/inventory/group_vars/.*\.yml$
    age: age1xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

The encrypted file (`terraform.tfvars`) is committed to Git. Decrypting it requires the age private key, which lives on the CI server and my local machine — never in the repository.

## Ansible: node configuration

Once Terraform provisions the VMs, Ansible configures them. The `bootstrap.yml` playbook does the minimum needed to get a freshly created node into working shape:

```yaml
# ansible/playbooks/bootstrap.yml
---
- name: Bootstrap new nodes
  hosts: all
  become: true
  roles:
    - common
    - cis-level1

- name: Configure K3s server nodes
  hosts: k3s_servers
  become: true
  roles:
    - k3s
```

The `common` role installs base packages, configures NTP, hardens SSH, sets sysctl parameters, and creates system users. The `cis-level1` role applies CIS Benchmark Level 1 recommendations for Debian.

I did not write the CIS role from scratch. I started from the community role `dev-sec/ansible-collection-hardening` and adapted it. There are quite a few tasks the default role applies that do not fit a homelab — things designed for production servers with strict audit requirements. I went through each task, understood what it did, and decided if it applied to my case.

Some things I disabled:

```yaml
# ansible/roles/cis-level1/defaults/main.yml
os_auth_pam_pwquality_enable: false  # No local users with passwords
os_security_users_allow: ["vagrant"] # Dev environment only
os_filesystem_whitelist:
  - vfat  # Required for UEFI boot
```

And some things I added specifically for K3s:

```yaml
# Kernel parameters required by K3s with protect-kernel-defaults
kernel_parameters:
  - { name: "kernel.panic", value: "10" }
  - { name: "kernel.panic_on_oops", value: "1" }
  - { name: "vm.overcommit_memory", value: "1" }
  - { name: "vm.panic_on_oom", value: "0" }
  - { name: "fs.inotify.max_user_watches", value: "524288" }
  - { name: "fs.inotify.max_user_instances", value: "512" }
```

## Kubernetes: manifests with Kustomize

For Kubernetes manifests I use Kustomize over Helm where possible. Helm is more powerful for complex things, but for my own applications Kustomize is sufficient and produces readable YAML.

The basic Kustomize structure:

```
kubernetes/apps/monitoring/
├── base/
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   ├── prometheus-deployment.yaml
│   └── grafana-deployment.yaml
└── overlays/
    └── homelab/
        ├── kustomization.yaml
        └── patches/
            └── resource-limits.yaml
```

The `homelab` overlay adds environment-specific settings without modifying the base manifests:

```yaml
# kubernetes/apps/monitoring/overlays/homelab/patches/resource-limits.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: monitoring
spec:
  template:
    spec:
      containers:
        - name: prometheus
          resources:
            requests:
              memory: "256Mi"
              cpu: "100m"
            limits:
              memory: "512Mi"
              cpu: "500m"
```

## OPA/Gatekeeper: policies as code

Gatekeeper is a Kubernetes admission controller that uses OPA (Open Policy Agent) to evaluate policies written in Rego. Instead of letting any pod deploy with any configuration, policies reject manifests that do not meet security requirements.

Active policies:

### No containers running as root

```yaml
# kubernetes/policies/gatekeeper/no-root-containers.yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sPSPAllowedUsers
metadata:
  name: psp-pods-allowed-user-ranges
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    excludedNamespaces:
      - kube-system
      - falco
  parameters:
    runAsUser:
      rule: MustRunAsNonRoot
    runAsGroup:
      rule: MustRunAs
      ranges:
        - min: 1000
          max: 65535
```

### Required resource limits

Without resource limits, a pod can consume all the node's memory and bring down the cluster. This policy prevents that:

```yaml
# kubernetes/policies/gatekeeper/require-resource-limits.yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredresources
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredResources
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredresources

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.limits.memory
          msg := sprintf("Container '%v' has no memory limit defined", [container.name])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.limits.cpu
          msg := sprintf("Container '%v' has no CPU limit defined", [container.name])
        }
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredResources
metadata:
  name: require-resource-limits
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    excludedNamespaces:
      - kube-system
```

### Trusted image registries

```yaml
# kubernetes/policies/gatekeeper/allowed-registries.yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sallowedrepos
spec:
  crd:
    spec:
      names:
        kind: K8sAllowedRepos
      validation:
        openAPIV3Schema:
          type: object
          properties:
            repos:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sallowedrepos

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not any_repo_matches(container.image)
          msg := sprintf("Image '%v' does not come from an allowed registry", [container.image])
        }

        any_repo_matches(image) {
          repo := input.parameters.repos[_]
          startswith(image, repo)
        }
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRepos
metadata:
  name: allowed-registries
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    excludedNamespaces:
      - kube-system
  parameters:
    repos:
      - "registry.homelab.internal/"
      - "ghcr.io/my-user/"
      - "quay.io/prometheus/"
      - "grafana/"
```

## Seccomp profiles

Seccomp profiles restrict which system calls a container can make. Kubernetes has a default profile (`RuntimeDefault`) that is already reasonable, but for applications I know well I define tighter profiles.

The profiles live in the repository and are deployed as ConfigMaps or directly to the nodes:

```json
// kubernetes/policies/seccomp/web-app-profile.json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": ["SCMP_ARCH_X86_64"],
  "syscalls": [
    {
      "names": [
        "accept4", "bind", "brk", "clone", "close", "connect",
        "epoll_create1", "epoll_ctl", "epoll_wait", "execve",
        "exit_group", "fcntl", "fstat", "futex", "getdents64",
        "getpid", "getsockname", "getsockopt", "listen", "lstat",
        "mmap", "mprotect", "munmap", "nanosleep", "newfstatat",
        "openat", "poll", "prctl", "read", "recvfrom", "rt_sigaction",
        "rt_sigprocmask", "rt_sigreturn", "sendto", "set_robust_list",
        "setsockopt", "sigaltstack", "socket", "stat", "write"
      ],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

Referenced from the pod spec:

```yaml
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: "web-app-profile.json"
```

Building a seccomp profile from scratch is tedious. My approach is to start with `RuntimeDefault`, use `strace` to see what syscalls the application actually makes, and then build a tighter profile for workloads I want to restrict further.

## CI/CD pipeline

The repository has a GitLab CI pipeline that automates applying changes. The flow is:

- On merge requests: `terraform plan` and `ansible-lint` to catch problems before merging
- On merge to `main`: `terraform apply` and, if Ansible files changed, the corresponding playbook

```yaml
# .gitlab-ci.yml (excerpt)
stages:
  - validate
  - plan
  - apply

variables:
  TF_ROOT: "${CI_PROJECT_DIR}/terraform/environments"
  ANSIBLE_CONFIG: "${CI_PROJECT_DIR}/ansible/ansible.cfg"

terraform-validate:
  stage: validate
  image: hashicorp/terraform:1.6
  script:
    - cd "$TF_ROOT"
    - terraform init -backend=false
    - terraform validate
  rules:
    - changes:
        - terraform/**/*

terraform-plan:
  stage: plan
  image: hashicorp/terraform:1.6
  script:
    - cd "$TF_ROOT"
    - terraform init
    - terraform plan -out=tfplan
  artifacts:
    paths:
      - "${TF_ROOT}/tfplan"
    expire_in: 1 week
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      changes:
        - terraform/**/*

terraform-apply:
  stage: apply
  image: hashicorp/terraform:1.6
  script:
    - cd "$TF_ROOT"
    - terraform init
    - terraform apply -auto-approve
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
      changes:
        - terraform/**/*
  when: manual

ansible-lint:
  stage: validate
  image: python:3.11-slim
  script:
    - pip install ansible ansible-lint
    - ansible-lint ansible/
  rules:
    - changes:
        - ansible/**/*
```

The `terraform apply` is manual — I do not want infrastructure changing automatically without my approval. The plan runs automatically on the MR for visibility.

## What I sanitized

Publishing the repository required reviewing what should not be there:

- **Internal IPs**: replaced with example ranges (`192.168.1.x`)
- **Domain names**: the internal homelab domain (`homelab.internal` in the repo, something different in production)
- **Usernames**: real usernames are not in the repository
- **SSH public keys**: replaced with placeholders
- **Password hashes**: removed from the Ansible inventory
- **Application secrets**: encrypted with SOPS or removed, with an `.example` file alongside

The rule I followed: if someone with access to my local network could use that information to attack something, it does not go into the repository in plaintext. Everything else can be there.

## What is still pending

There are still things I manage by hand that should be codified:

**OpenWrt**: the router configuration is the hardest to bring into IaC. There is a Terraform module for OpenWrt that is not very well maintained. For now I manage it with an Ansible script that backs up the configuration and another that restores it. Not idempotent, but it works.

**TrueNAS**: it has a fairly complete REST API. There is a Terraform provider in development. It is on my radar for the next iteration.

**Backups**: I have backups, but the process is not in the repository. It lives in another loose script that will eventually end up here.

The repository is never "finished." What matters is that the current state of the homelab is represented in it, and that any change goes through Git.
