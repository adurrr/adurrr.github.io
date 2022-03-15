+++
author = "Adur"
title = "GitOps principles and workflow: a practical guide"
date = "2022-03-15"
description = "Understanding GitOps core principles, workflow, and a hands-on comparison of ArgoCD vs Flux for Kubernetes deployments."
featured = true
tags = [
    "gitops",
    "kubernetes",
    "argocd",
    "flux",
    "git"
]
categories = [
    "devops"
]
series = ["DevOps"]
thumbnail = "images/devops.png"
toc = true
+++

## What is GitOps?

GitOps is an operational framework that takes DevOps best practices from application development (version control, collaboration, CI/CD) and applies them to infrastructure automation. At its core, GitOps treats **Git as the single source of truth** for both infrastructure and application configuration.

Rather than manually applying changes to your clusters or infrastructure, you declare the desired state in Git repositories. Automated agents then ensure your systems converge to that declared state. If something drifts, the system corrects itself automatically.

This isn't just a trendy buzzword. GitOps fundamentally changes how teams operate Kubernetes environments, bringing real predictability, auditability, and deployment speed.

## Core Principles

GitOps rests on four foundational principles. If your workflow does not honor all four, you are probably doing "Git-flavored ops" rather than true GitOps.

### 1. Declarative Configuration

The entire desired state of your system must be described declaratively. For Kubernetes, that means YAML manifests, Helm charts, or Kustomize overlays stored in Git. No imperative `kubectl apply` commands run from someone's laptop.

### 2. Versioned and Immutable

Since Git is your source of truth, every change is versioned. You get a complete audit trail automatically (who changed what, when, and why). Rolling back is just reverting a commit. No more guessing about what the previous production state was.

### 3. Pulled Automatically

Approved changes are **automatically pulled** and applied by agents running inside the cluster. This is the crucial difference: instead of a CI pipeline pushing changes into the cluster (which means giving CI credentials to the cluster), the agent inside the cluster pulls changes from Git. This pull-based model significantly improves your security.

### 4. Continuously Reconciled (Self-Healing)

The agent continuously compares the desired state in Git with the actual state in the cluster. If someone manually modifies a resource (or if a node fails and resources are rescheduled), the agent detects the drift and reconciles it. The system is self-healing by design.

## GitOps Workflow

Here's a simplified text-based diagram of a typical GitOps workflow:

```
Developer ──> Git Push ──> Application Repo
                                  │
                                  ▼
                          CI Pipeline (build, test, push image)
                                  │
                                  ▼
                          Config Repo (update image tag in manifests)
                                  │
                                  ▼
                          GitOps Agent (ArgoCD / Flux)
                            watches Config Repo
                                  │
                                  ▼
                          Kubernetes Cluster
                            (desired state applied)
                                  │
                                  ▼
                          Continuous Reconciliation
                            (drift detection + self-healing)
```

**Key points:**

- The **Application Repo** contains your source code. CI builds and pushes container images.
- The **Config Repo** (sometimes called the "environment repo") holds your Kubernetes manifests. CI or an automated process updates the image tag here after each successful build.
- The **GitOps Agent** watches the Config Repo and applies changes to the cluster.

Separating application code from deployment configuration is a best practice because it keeps concerns isolated and allows independent versioning.

## ArgoCD vs Flux: A Practical Comparison

The two dominant GitOps tools for Kubernetes are **ArgoCD** and **Flux**. Both are CNCF projects and both are production-ready. Here's an honest comparison based on hands-on experience.

| Feature | ArgoCD | Flux |
|---|---|---|
| **UI** | Rich web UI with visualization | No built-in UI (use Weave GitOps or similar) |
| **Architecture** | Centralized server with API | Decentralized, controller-based |
| **Multi-tenancy** | AppProjects with RBAC | Namespace-scoped controllers |
| **Helm support** | Native rendering | HelmRelease CRD with HelmController |
| **Kustomize** | Native | Native |
| **Notifications** | Built-in notification controller | Separate notification-controller |
| **Image automation** | Not built-in (use Argo Image Updater) | Built-in image automation controllers |
| **Learning curve** | Lower (UI helps) | Slightly steeper (CLI/CRD-first) |
| **GitOps Toolkit** | Monolithic | Modular (pick what you need) |

### When to choose ArgoCD

- Your team values a visual dashboard for cluster state.
- You need strong multi-tenancy with role-based access control.
- You want a lower barrier to entry for team members new to GitOps.

### When to choose Flux

- You prefer a modular, composable architecture.
- You need built-in image automation (automatically updating image tags in Git).
- You favor a CLI-first, infrastructure-as-code approach without relying on a UI.

Honestly, both are solid choices. I lean toward ArgoCD for teams that need visibility and toward Flux for platform teams that prefer composability.

## ArgoCD Application Manifest Example

Here's a minimal ArgoCD `Application` resource that deploys an application from a Git repository:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/my-org/my-app-config.git
    targetRevision: main
    path: overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: my-app
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

Here's what each important field does:

- **`source.repoURL`**: Points to the Git repository containing your manifests.
- **`source.targetRevision`**: The branch or tag to track.
- **`source.path`**: The directory within the repo containing the manifests (useful with Kustomize overlays).
- **`destination.server`**: The target Kubernetes cluster API server.
- **`syncPolicy.automated`**: Enables automatic syncing. `prune: true` removes resources that are no longer in Git. `selfHeal: true` re-applies desired state when drift is detected.

To apply this manifest after installing ArgoCD:

```bash
kubectl apply -f application.yaml
```

ArgoCD will immediately begin syncing the declared state to the cluster.

## Benefits for Teams

Adopting GitOps isn't just a technical improvement. It changes how your team actually works:

- **Faster onboarding**: New team members can understand the entire system state by reading the config repo. No tribal knowledge needed.
- **Reliable rollbacks**: Revert a Git commit, and the cluster follows. No need to remember the exact `kubectl` commands that were run.
- **Improved security posture**: Developers never need direct `kubectl` access to production. All changes go through Git PRs with code review.
- **Audit compliance**: Every change is a Git commit with author, timestamp, and rationale. This satisfies many compliance requirements out of the box.
- **Reduced cognitive load**: Developers focus on writing code and updating manifests. The GitOps agent handles the rest.
- **Disaster recovery**: If a cluster is destroyed, you can recreate the entire state from Git. The config repo is your backup.

## Getting Started

If you're new to GitOps, here's a pragmatic path forward:

1. **Start with a non-critical environment.** Set up ArgoCD or Flux in a staging cluster first.
2. **Migrate one application at a time.** Don't try to convert everything overnight.
3. **Establish a config repo convention early.** Decide on directory structure, naming, and branching strategy before scaling.
4. **Use Kustomize or Helm for environment differences.** Avoid copying manifests between `staging/` and `production/` directories.
5. **Set up notifications.** Connect ArgoCD or Flux to Slack or your preferred channel so the team sees sync events.

GitOps is one of those practices where the investment pays off quickly. Once your team experiences the confidence of knowing that Git reflects reality, there's no going back.
