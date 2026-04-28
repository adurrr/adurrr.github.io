# Adur — DevSecOps & AI Infrastructure Blog

[![Build & Deploy](https://github.com/adurrr/adurrr.github.io/actions/workflows/deploy.yml/badge.svg)](https://github.com/adurrr/adurrr.github.io/actions/workflows/deploy.yml)
[![Hugo](https://img.shields.io/badge/Hugo-0.154.2-ff4088?logo=hugo&logoColor=white)](https://gohugo.io/)
[![GitHub Pages](https://img.shields.io/badge/GitHub%20Pages-Live-222?logo=github&logoColor=white)](https://adurrr.github.io)
[![License](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

> **Live Site:** [https://adurrr.github.io](https://adurrr.github.io)

A bilingual (English / Español) technical blog and knowledge base focused on **DevSecOps**, **AI/ML infrastructure**, **cloud-native security**, and **technological sovereignty**. Built as a static site with Hugo, deployed via GitHub Actions, and developed entirely in a containerized environment.

---

## What This Repository Covers

| Domain | Topics |
|--------|--------|
| **DevOps** | GitOps, CI/CD, Terraform, Ansible, observability stacks |
| **DevSecOps** | Shift-left security, SAST/DAST, container hardening, supply chain security |
| **MLOps / AIOps / LLMOps** | ML pipeline automation, model deployment, intelligent monitoring |
| **DataOps** | Data pipeline orchestration, quality assurance, observability |
| **Cloud-Native** | Kubernetes, Docker, K3s homelab architectures |
| **Security** | Kubernetes hardening, container scanning, network segmentation |
| **Sovereignty** | Self-hosting, Raspberry Pi clusters, OPNSense, privacy tools |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      GitHub Repository                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐   │
│  │   Content    │  │   Config     │  │   Assets/SCSS    │   │
│  │  (Markdown)  │  │   (YAML)     │  │   (Custom)       │   │
│  └──────┬───────┘  └──────┬───────┘  └────────┬─────────┘   │
└─────────┼─────────────────┼───────────────────┼─────────────┘
          │                 │                   │
          ▼                 ▼                   ▼
┌─────────────────────────────────────────────────────────────┐
│              GitHub Actions CI/CD Pipeline                    │
│  ┌─────────────┐ ┌─────────────┐ ┌───────────────────────┐  │
│  │  Checkout   │ │  Setup Go   │ │  Setup Hugo Extended  │  │
│  │  + Submods  │ │  + Node.js  │ │  + Dart Sass          │  │
│  └──────┬──────┘ └──────┬──────┘ └───────────┬───────────┘  │
│         └─────────────────┴────────────────────┘             │
│                           │                                  │
│                    ┌──────▼──────┐                           │
│                    │  hugo build │                           │
│                    │  --minify   │                           │
│                    │  --gc       │                           │
│                    └──────┬──────┘                           │
│                           │                                  │
│                    ┌──────▼──────┐                           │
│                    │ Cache Save  │                           │
│                    └──────┬──────┘                           │
│                           │                                  │
│                    ┌──────▼──────┐                           │
│                    │  Artifact   │                           │
│                    │  Upload     │                           │
│                    └──────┬──────┘                           │
│                           │                                  │
│                    ┌──────▼──────┐                           │
│                    │ Deploy to   │                           │
│                    │ GitHub Pages│                           │
│                    └─────────────┘                           │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │  adurrr.github.io │
                    │   (GitHub Pages)  │
                    └─────────────────┘
```

---

## Repository Structure

```
.
├── .devcontainer/          # VS Code DevContainer config (Hugo Extended + Node + Go)
├── .github/
│   └── workflows/
│       ├── deploy.yml      # Main CI/CD: build + deploy to GitHub Pages
│       └── update-theme.yml # Automated daily theme updates via Hugo modules
├── assets/
│   ├── icons/              # Custom SVG icons
│   ├── img/                # Images, avatars, favicons
│   └── scss/
│       └── custom.scss     # Theme style overrides
├── config/_default/        # Hugo site configuration
│   ├── config.yaml         # Base URL, language, pagination
│   ├── languages.yaml      # i18n: English + Spanish
│   ├── markup.yaml         # Markdown rendering, code highlighting, math
│   ├── menu.yaml           # Social + navigation links
│   ├── module.yaml         # Hugo theme module import
│   ├── params.yaml         # Theme parameters (sidebar, widgets, footer)
│   ├── permalinks.yaml     # URL structure
│   └── related.yaml        # Related content algorithm
├── content/
│   ├── _index.{en,es}.md   # Homepage metadata
│   ├── categories/         # Taxonomy definitions
│   ├── page/               # Static pages (about, archives, links, search)
│   └── post/               # Blog posts organized by topic
│       ├── blog/
│       ├── data_analysis/  # aiops, dataops, llmops, mlops
│       ├── development/
│       ├── devops/
│       ├── research/
│       ├── security/
│       ├── sysadmin/
│       └── technological_sovereignty/
├── go.mod                  # Hugo module: theme dependency
├── go.sum                  # Module checksums
├── LICENSE                 # MIT License
└── README.md               # This file
```

---

## Tech Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| **Static Site Generator** | [Hugo](https://gohugo.io/) Extended 0.154.2 | Ultra-fast static builds |
| **Theme** | [Hugo Theme Stack](https://github.com/CaiJimmy/hugo-theme-stack) v4 | Clean, modern blog theme |
| **Styling** | Dart Sass 1.97.1 | CSS preprocessing |
| **Modules** | Go 1.25.5 | Hugo theme dependency management |
| **Build Tooling** | Node.js 24.12.0 | npm-based asset pipelines (if needed) |
| **CI/CD** | GitHub Actions | Automated build, cache, deploy |
| **Hosting** | GitHub Pages | Free, fast static hosting with CDN |
| **Dev Environment** | VS Code DevContainer | Reproducible containerized development |

---

## CI/CD Pipeline

The `.github/workflows/deploy.yml` pipeline runs on every push to `main`:

1. **Checkout** — Fetches repository with full history (`fetch-depth: 0`) for Hugo's GitInfo
2. **Tooling Setup** — Go, Node.js, Dart Sass, and Hugo Extended installed
3. **Build** — `hugo --gc --minify --baseURL <pages_url>` with Hugo cache optimization
4. **Cache** — Restores/saves `hugo_cache` between runs for faster builds
5. **Deploy** — Uploads `./public` artifact and deploys to GitHub Pages

A second workflow (`update-theme.yml`) runs daily via cron to automatically update the Hugo theme module and commit changes.

---

## Local Development

### Option 1: DevContainer (Recommended)

This repository includes a fully configured [DevContainer](https://containers.dev/).

1. Open in VS Code
2. Run **"Reopen in Container"**
3. The container includes Hugo Extended, Node.js, Go, and Dart Sass pre-installed
4. Forwarded port: `1313`

### Option 2: Local Hugo Installation

**Requirements:**
- Hugo Extended ≥ 0.154.0
- Go ≥ 1.20
- Node.js ≥ 20 (optional, for asset pipelines)
- Dart Sass (optional, for SCSS compilation)

```bash
# Clone
git clone https://github.com/adurrr/adurrr.github.io.git
cd adurrr.github.io

# Install theme module
hugo mod get
hugo mod tidy

# Start development server with draft preview
hugo server -D

# Build for production
hugo --minify --gc
```

The built site outputs to `./public/`.

### VS Code Tasks

Included tasks for quick access (`Ctrl+Shift+P` → "Run Task"):

| Task | Command |
|------|---------|
| **Serve Drafts** | `hugo server -D` |
| **Build** | `hugo` |

---

## Content Management

Posts are organized by topic under `content/post/`. Each post supports:

- **Bilingual content**: `filename.en.md` and `filename.es.md`
- **Front matter**: Title, date, description, tags, categories, featured image, TOC
- **Math rendering**: LaTeX via passthrough delimiters (`$$`, `\[`, `\(`)
- **Code highlighting**: Line numbers, syntax guessing, fenced blocks
- **Series**: Group related posts (e.g., "DevOps" series)

### Creating a New Post

```bash
hugo new content post/devops/my-new-post.en.md
```

---

## Deployment

Pushing to the `main` branch automatically triggers the deployment workflow. No manual steps required.

To deploy manually, go to **Actions → Build and deploy → Run workflow**.

---

## Automated Maintenance

| Automation | Schedule | Action |
|-----------|----------|--------|
| Theme Update | Daily at 00:00 UTC | `hugo mod get -u` + auto-commit |
| Dependency Refresh | On push | `hugo mod tidy` |

---

## License

This repository is licensed under the [MIT License](LICENSE).  
Blog content is licensed under [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/).

---

<p align="center">
  Built with <a href="https://gohugo.io/">Hugo</a> · Deployed on <a href="https://pages.github.com/">GitHub Pages</a> · Developed in <a href="https://containers.dev/">DevContainers</a>
</p>
