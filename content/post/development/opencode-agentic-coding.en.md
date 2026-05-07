---
title: "OpenCode, Agent Orchestration and Subagent-Driven Development: A Complete Guide"
date: 2026-05-02T10:00:00+01:00
draft: false
tags:
  - "OpenCode"
  - "AI"
  - "Agentic Coding"
  - "SDD"
  - "MCP"
  - "LSP"
  - "Software Development"
  - "DevOps"
  - "SysAdmin"

categories:
  - "development"
---

## Introduction

OpenCode is an open-source, terminal-native AI coding agent built in Go. Unlike cloud-based IDEs like Cursor or Claude Code, it is entirely provider-agnostic: you bring your own API keys, run models locally, or subscribe to managed plans, and OpenCode handles the orchestration. With support for 75+ LLM providers, native LSP integration, MCP extensibility, and a thriving plugin ecosystem, it has become a standard-bearer for agentic coding.

This guide is structured to take you from installation to production workflows. We begin with core concepts — agents, subagents, LSP, and MCP — then introduce **Subagent-Driven Development (SDD)**, the methodology that ties orchestration, planning, and execution into a coherent workflow. From there, we map SDD to real-world domains: full-stack development, sysadmin operations, and DevOps infrastructure, drawing on documented use cases from developers in the field.

---

## What is OpenCode?

OpenCode is an open-source AI coding agent (MIT license) designed to run in your terminal, desktop, or IDE. It treats agents as a runtime system, not loose prompts. Agents are defined in code or loaded from Markdown, merged into a shared registry, and executed through a unified prompt, permission, and session pipeline.

### Key Features

- **Provider-agnostic**: Claude, OpenAI, Google Gemini, Groq, Fireworks, Together AI, OpenRouter, Azure, AWS Bedrock, and local models via Ollama.
- **Native Terminal UI**: Built with Bubble Tea (Go) for a smooth TUI experience.
- **Multi-session support**: Run multiple agents in parallel, each with its own context.
- **LSP integration**: Automatically loads Language Server Protocol servers for code intelligence.
- **MCP support**: Extend capabilities via the Model Context Protocol.
- **Plugin system**: TypeScript/JavaScript plugins with 25+ lifecycle hooks.
- **Client/Server architecture**: Run the server headlessly and connect from multiple clients.
- **Privacy-first**: Does not store your code or context data.

### Installation

```bash
# Via install script
curl -fsSL https://opencode.ai/install | bash

# Via npm
npm install -g opencode-ai

# Start OpenCode
opencode
```

On first launch, OpenCode detects your project structure, initializes a `.opencode/` directory, and prompts you to configure a model provider.

---

## OpenCode Go: Low-Cost Model Access

**OpenCode Go** is a low-cost subscription ($5 for the first month, then $10/month) that provides reliable access to curated open-source coding models. It is designed for developers who want generous limits and stable global access without managing multiple API keys.

### What You Get

- **Price**: $5 first month, then $10/month. Cancel anytime.
- **Models included**: GLM-5.1, GLM-5, Kimi K2.5, Kimi K2.6, MiMo-V2-Pro, MiMo-V2-Omni, MiMo-V2.5-Pro, MiMo-V2.5, Qwen3.5 Plus, Qwen3.6 Plus, MiniMax M2.5, MiniMax M2.7, DeepSeek V4 Pro, DeepSeek V4 Flash.
- **Hosting**: US, EU, and Singapore for stable global access.
- **Privacy**: Zero-retention policy; providers do not use your data for model training.

### Usage Limits

Limits are defined in dollar value rather than fixed request counts:

| Window | Limit |
|--------|-------|
| Per 5 hours | $12 |
| Per week | $30 |
| Per month | $60 |

Cheaper models stretch further. DeepSeek V4 Flash allows ~31,650 requests per 5 hours, while GLM-5.1 allows ~880.

### Setup

1. Subscribe at [opencode.ai/go](https://opencode.ai/go).
2. Copy your API key.
3. In the TUI, run `/connect`, select **OpenCode Go**, and paste your key.
4. Run `/models` to see available models.

Model IDs use the format `opencode-go/<model-id>`, for example:

```json
{
  "model": "opencode-go/kimi-k2.6"
}
```

---

## Core Concepts

Before diving into workflows, it is essential to understand the building blocks of OpenCode: agents, subagents, LSP, MCP, and the project rules system.

### Agents and Subagents

OpenCode has two types of agents:

- **Primary agents**: The main assistants you interact with directly. Cycle through them with the **Tab** key. Built-in primary agents include **Build** (full tools) and **Plan** (read-only, cannot modify files).
- **Subagents**: Specialized assistants that primary agents invoke for specific tasks. You can also invoke them manually with `@mention`.

Built-in subagents:

| Subagent | Role | Tools |
|----------|------|-------|
| **General** | Multi-step research and execution | Full access except todo |
| **Explore** | Fast, read-only codebase exploration | Read, search only |

When one agent delegates work, it does not simply append a prompt. It creates a **child session** with fresh context, passes a scoped instruction, and receives a structured result back. This makes delegation session-based, resumable, and inspectable.

### LSP (Language Server Protocol)

LSP integration gives OpenCode deep code intelligence. The AI can see type information, function signatures, import paths, and diagnostics — not just raw text.

Supported operations: `goToDefinition`, `findReferences`, `hover`, `documentSymbol`, `workspaceSymbol`, `goToImplementation`, `prepareCallHierarchy`, `incomingCalls`, `outgoingCalls`.

The `lsp` tool is available when `OPENCODE_EXPERIMENTAL_LSP_TOOL=true` is set. OpenCode includes pre-configured LSP servers for 30+ languages.

### MCP (Model Context Protocol)

MCP servers extend OpenCode with external tools and services. Built-in MCP-like tools include:

- **websearch**: Performs web searches via Exa AI (no API key required with OpenCode provider).
- **webfetch**: Fetches and reads specific URLs.
- **lsp**: Interacts with configured Language Servers.

Custom MCP servers are configured in `opencode.json`:

```json
{
  "mcpServers": {
    "github": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": ["GITHUB_PERSONAL_ACCESS_TOKEN=ghp_..."]
    }
  }
}
```

Be selective: MCP servers add tool definitions to your context window. The GitHub MCP alone can consume significant tokens.

### AGENTS.md and Project Rules

Run `/init` to generate an `AGENTS.md` file in your project root. This file teaches OpenCode about your project structure, conventions, and coding patterns. It is similar to Cursor's rules and improves the quality of generated code.

Example from a production TypeScript monorepo:

```markdown
# Project: Payment Processing API

This is a TypeScript monorepo using Bun workspaces.

## Structure
- `packages/core/` - Shared business logic
- `packages/api/` - Express API handlers
- `packages/workers/` - Background job processors

## Conventions
- Use Zod for all input validation
- All database queries go through the repository pattern
- Prefer composition over inheritance
- Test files live next to source files (*.test.ts)

## Commands
- `bun test` - Run all tests
- `bun run lint` - Run ESLint and Prettier
- `bun run build` - TypeScript compilation
```

### Permissions

OpenCode filters tools before the model sees them, then checks permissions again at execution time. This makes orchestration bounded by policy, not trust.

```json
{
  "permission": {
    "bash": "ask",
    "edit": "ask",
    "write": "allow"
  }
}
```

Values: `allow`, `deny`, `ask`. For production or sensitive environments, set `bash` and `edit` to `ask`.

---

## Agent Orchestration with oh-my-opencode-slim

**oh-my-opencode-slim** is an agent orchestration plugin for OpenCode. Instead of forcing a single model to handle every task, it routes jobs to specialized subagents, balancing quality, speed, and cost.

### The Pantheon

| Agent | Role | Default Model |
|-------|------|---------------|
| **Orchestrator** | Master delegator and strategic coordinator | openai/gpt-5.4 |
| **Explorer** | Fast codebase search and pattern matching | openai/gpt-5.4-mini |
| **Librarian** | External documentation and library research | openai/gpt-5.4-mini |
| **Oracle** | Strategic technical advisor, code reviewer, simplifier | openai/gpt-5.4 |
| **Designer** | UI/UX specialist for visual polish | openai/gpt-5.4-mini |
| **Fixer** | Fast implementation specialist for bounded tasks | openai/gpt-5.4-mini |
| **Observer** | Read-only visual analysis (images, PDFs, diagrams) | Disabled by default |

### Installation

```bash
bunx oh-my-opencode-slim@latest install
```

The installer generates a default OpenAI configuration. Edit `~/.config/opencode/oh-my-opencode-slim.json` to use Kimi, GitHub Copilot, or other providers. The configuration supports JSONC and includes an official JSON Schema for autocompletion.

### Key Features

- **Council**: Run multiple models in parallel and synthesise a single answer with `@council`.
- **Multiplexer Integration**: Watch agents work live in Tmux or Zellij panes.
- **Session Management**: Reuse recent child-agent sessions with short aliases.
- **Auto-continue**: Automatically resume orchestrator sessions with cooldowns and safety checks.
- **Preset Switching**: Switch agent model presets at runtime with `/preset`.
- **Cartography Skill**: Generate hierarchical codemaps to understand large codebases faster.
- **Interview**: Turn rough ideas into structured markdown specs via a browser-based Q&A flow.

---

## Subagent-Driven Development (SDD)

Subagent-Driven Development is the methodology that makes agent orchestration practical. It is not a single tool but a workflow pattern: decompose work into independent tasks, dispatch a fresh subagent for each task, and enforce review before completion. SDD prevents context pollution, controls costs, and maintains quality.

There are two complementary interpretations of SDD in the OpenCode ecosystem:

1. **Spec-Driven Development**: Requirements → Design → Tasks → Implementation. Specs are temporary scaffolding; code is the source of truth. Delete specs after implementation.
2. **Subagent-Driven Development**: Each independent task gets a fresh subagent with isolated context, followed by automatic two-stage review.

In practice, these merge: you write a spec, decompose it into tasks, and dispatch subagents for each task with review gates.

### When to Use SDD

Use SDD when:
- You have a detailed implementation plan.
- Tasks are mostly independent with weak dependencies.
- You want to complete all tasks in one session without switching contexts.
- Quality gates (spec compliance + code review) are non-negotiable.

Avoid SDD for tiny one-file changes where the overhead of subagent dispatch exceeds the benefit. In those cases, use a single Build agent directly.

### The SDD Workflow

#### Phase 1: Exploration and Spec Writing

The Orchestrator or Planner agent analyses the request and creates a specification.

```bash
# Initialize SDD scaffolding (if using sdd-flow or Agent Teams Lite)
/sdd-init

# Or manually: switch to Plan mode and ask for a spec
"Plan a user authentication system with JWT tokens, refresh token rotation, and role-based access control. Write the spec to specs/auth.md"
```

#### Phase 2: Task Decomposition

Break the spec into independent tasks. Each task should have a single, bounded scope.

Example tasks for an auth system:
1. Implement JWT token generation and validation utilities.
2. Create login and refresh endpoints.
3. Add middleware for role-based access control.
4. Write unit tests for token utilities.

#### Phase 3: Subagent Dispatch

For each task, the Orchestrator spawns a fresh subagent via the Task tool. Each subagent starts with **zero context** from previous tasks, preventing pollution. The Orchestrator injects only the relevant spec section and project standards.

```
Orchestrator → Task(subagent="sdd-apply", prompt="Implement task 1: JWT utilities. Read specs/auth.md section 3.1. Follow TDD. Run tests before returning.")
```

#### Phase 4: Two-Stage Review

After a subagent completes its task, SDD enforces two review stages before marking the task complete:

1. **Spec Compliance Review**: A reviewer subagent checks whether the implementation matches the spec exactly. Did it implement what was asked? Are the interfaces correct?
2. **Code Quality Review**: A second reviewer checks security, performance, maintainability, and test coverage.

If either review fails, the task is sent back to the implementer subagent with feedback. This creates automatic checkpoints.

#### Phase 5: Integration and Validation

Once all tasks pass review, the Orchestrator integrates the work, runs the full test suite, and validates end-to-end behaviour.

### SDD Tools and Plugins

Several community projects implement SDD workflows for OpenCode:

#### cc-sdd (Spec-Driven Development)

[cc-sdd](https://github.com/gotalab/cc-sdd) brings structured Spec-Driven Development to OpenCode with slash commands:

```bash
npx cc-sdd@latest --opencode-skills
```

Commands include:
- `/kiro-spec-init`: Start a new feature spec.
- `/kiro-spec-requirements`: Write requirements.
- `/kiro-spec-design`: Create architecture design.
- `/kiro-spec-tasks`: Generate task checklist.
- `/kiro-impl`: Autonomous implementation with per-task subagents, TDD, and independent review.

Each task gets a fresh implementer running TDD (RED → GREEN), an independent reviewer, and an auto-debug pass if blocked.

#### sdd-flow

[sdd-flow](https://github.com/Heldinhow/sdd-flow) is a plugin that embeds SDD directly into your repository:

```bash
# One-time bootstrap
/sdd-init

# Switch to the "Spec Driven" planning agent
# Describe your feature in natural language
# Approve each phase before it advances

# Execute the plan
/implement
```

Assets are repo-local: `.opencode/skills/`, `.specify/`, `specs/`, and `AGENTS.md`. This means the workflow travels with the code, not just the developer's local config.

#### Agent Teams Lite (Gentleman Programming)

[Agent Teams Lite](https://github.com/Gentleman-Programming/agent-teams-lite) provides a complete SDD orchestrator with 10 specialised sub-agents and slash commands:

| Command | Purpose |
|---------|---------|
| `/sdd-init` | Initialise SDD context |
| `/sdd-explore` | Investigate an idea |
| `/sdd-new` | Start a new change |
| `/sdd-apply` | Implement tasks |
| `/sdd-verify` | Validate implementation |
| `/sdd-archive` | Archive completed change |

The system uses a skill registry: `.atl/skill-registry.md` captures project conventions, and the orchestrator injects compact rules into each subagent prompt as `## Project Standards (auto-resolved)`.

#### SDD Profiles

[OpenCode SDD Profiles](https://github.com/Gentleman-Programming/gentle-ai) allow you to create named model configurations and switch between them with **Tab** inside OpenCode. Each profile generates its own orchestrator plus 10 sub-agents in `opencode.json`:

```bash
gentle-ai sync \
  --profile cheap:anthropic/claude-haiku-3.5-20241022 \
  --profile-phase cheap:sdd-apply:anthropic/claude-sonnet-4-20250514
```

This creates a "cheap" profile where everything runs on Haiku except `sdd-apply`, which uses Sonnet. Press **Tab** to cycle between `sdd-orchestrator`, `sdd-orchestrator-cheap`, and `sdd-orchestrator-premium`.

### Subagent-to-Subagent Delegation

OpenCode PR #7756 introduced subagent-to-subagent delegation with configurable `task_budget` and depth limits to prevent infinite loops. By default, only primary agents can task subagents. To enable subagent delegation, set a budget:

```json
{
  "agent": {
    "sdd-apply": {
      "task_budget": 3,
      "permission": {
        "task": {
          "sdd-debug": "allow"
        }
      }
    }
  }
}
```

This allows an implementer subagent to spawn a debugger subagent up to 3 times if it gets stuck.

---

## Model Configuration and Presets

### High-Performance Setup

```json
{
  "$schema": "https://opencode.ai/config.json",
  "agents": {
    "coder": {
      "model": "anthropic/claude-sonnet-4-5-20250929",
      "maxTokens": 8000,
      "reasoningEffort": "medium"
    },
    "task": {
      "model": "openai/gpt-4.1-mini",
      "maxTokens": 5000
    },
    "title": {
      "model": "openai/gpt-4.1-mini"
    }
  }
}
```

### Budget Setup (~$30/month)

```json
{
  "agents": {
    "coder": {
      "model": "opencode-go/kimi-k2.6",
      "maxTokens": 8000
    },
    "task": {
      "model": "opencode-go/deepseek-v4-flash",
      "maxTokens": 5000
    },
    "title": {
      "model": "copilot/gpt-4o-mini"
    }
  }
}
```

### Free Setup

```json
{
  "provider": {
    "ollama": {
      "api_url": "http://localhost:11434/v1"
    }
  },
  "model": {
    "default": "opencode/minimax-m2.5-free",
    "fast": "ollama/qwen3.6-35b-a3b"
  }
}
```

### DevOps-Focused Configuration

```json
{
  "$schema": "https://opencode.ai/config.json",
  "model": "opencode-go/qwen3.6-plus",
  "permission": {
    "bash": "ask",
    "edit": "ask",
    "write": "allow"
  },
  "agents": {
    "devops-engineer": {
      "model": "opencode-go/glm-5",
      "maxTokens": 8000,
      "instructions": "You are a DevOps specialist. Confirm before applying Kubernetes manifests. Prefer idempotent patterns."
    }
  },
  "lsp": {
    "yaml": {
      "command": "yaml-language-server",
      "args": ["--stdio"]
    },
    "terraform": {
      "command": "terraform-ls",
      "args": ["serve"]
    }
  }
}
```

### Model Variants and Profiles

Many models support reasoning variants. Use `variant_cycle` to switch between `low`, `medium`, `high`, and `xhigh`.

```json
{
  "provider": {
    "anthropic": {
      "models": {
        "claude-sonnet-4-5-20250929": {
          "options": {
            "thinking": {
              "type": "enabled",
              "budgetTokens": 16000
            }
          }
        }
      }
    }
  }
}
```

Multi-provider fallbacks prevent session failure:

```json
{
  "agents": {
    "coder": {
      "model": "opencode-go/kimi-k2.6",
      "fallback_models": ["opencode-go/glm-5", "anthropic/claude-sonnet-4-5"]
    }
  }
}
```

---

## Domain-Specific Applications

The following sections map the SDD methodology to real-world use cases documented by developers, sysadmins, and DevOps engineers.

### Full-Stack Development

#### Greenfield Application with SDD

In a project-based tutorial by [ZBuild](https://www.zbuild.io/resources/news/opencode-tutorial-2026), a developer built a full-stack bookmark manager in ~30 minutes using OpenCode. The stack included Express.js/TypeScript, SQLite with FTS5, and a vanilla frontend.

Mapped to SDD phases:

1. **Spec**: Plan mode outlined the data model, API routes, and database schema. The spec was saved to `.opencode/plans/bookmark-manager.md`.
2. **Tasks**: Decomposed into scaffolding, data model, API endpoints, search, tags, frontend, and tests.
3. **Dispatch**: Build mode implemented each task sequentially, with the agent running `curl` to verify API behaviour after each endpoint.
4. **Review**: The agent ran the test suite and fixed failures before proceeding.

This Plan/Build cycle prevents the AI from making large structural decisions blindly.

#### Multi-Agent Refactoring with Orchestration

A Vercel engineer documented using [OpenCode with the Vercel AI Gateway](https://vercel.com/kb/guide/how-i-use-opencode-with-vercel-ai-gateway-to-build-features-fast) to migrate an authentication module from session-based auth to JWT tokens using the `ulw` (ultrawork) keyword.

The SDD-style workflow:

- **Orchestrator** (Claude Opus 4.6) received the request.
- It spawned two background subagents in parallel:
  - **Explore** (GPT-5 Mini): Mapped 12 files in the auth flow.
  - **Librarian** (Claude Sonnet 4.6): Found the framework's recommended JWT pattern.
- After receiving findings, the Orchestrator delegated subtasks:
  - Cryptographic logic → GPT-5.4 worker.
  - Middleware updates → Claude Haiku 4.5 worker.
- **Agent-browser** verified the login flow in a real browser.

Result: ~70% cost reduction compared to a single large model, with no manual model switching.

#### Multi-Lens Code Review as SDD Verification

[JP Caparas](https://jpcaparas.medium.com/one-reviewer-three-lenses-building-a-multi-agent-code-review-system-with-opencode-21ceb28dde10) built a multi-agent review system that mirrors the SDD two-stage review pattern. A lead reviewer agent analyses the diff, then spawns specialist reviewers in parallel:

- **review-frontend**: `.tsx`, `.jsx`, `.vue`, `.css`, `.scss` files.
- **review-backend**: `.py`, `.go`, `.ts` in `api/` or `services/`.
- **review-devops**: `Dockerfile`, `*.yaml`, `*.tf`, `.github/workflows/*`.

The lead agent synthesises findings into a unified report: **LGTM**, **NEEDS CHANGES**, or **DISCUSS**. This is the verification stage of SDD applied to existing code rather than new implementation.

#### E2B Sandbox for Teams

A [Reddit user](https://www.reddit.com/r/opencodeCLI/comments/1r85nve/running_opencode_in_e2b_cloud_sandboxes_so_my/) documented running OpenCode inside E2B cloud sandboxes for non-technical users. The workflow uses:

- A tailored `AGENTS.md` with persona rules and anti-hallucination guardrails.
- A three-file context system: `PROJECT.md` (spec), `MEMORY.md` (build notes), and a slim conversation log.
- Automated verification after each build (schema checks, API wiring validation).
- Auto-commit every 5 minutes.

This shows SDD principles applied in a managed environment: spec first, execution second, verification always.

---

### SysAdmin and Operations

#### Natural-Language Remote Server Management

The [argv.cloud blog](https://argv.cloud/blog/2025/ai-remote-ops/) introduced "Agentic Sysadmin" using OpenCode with a custom `remote.ts` tool that executes commands over SSH.

Setup:
- Standard `.ssh/config` with host aliases.
- Custom tool wrapping `execa` to run `ssh -o BatchMode=yes <host> <command>`.
- Optional sudo via `OC_SSH` env variable injected into stdin (the LLM never sees the password).

SDD mapping:
- **Spec**: "Audit test-server with Lynis and generate a Markdown summary."
- **Task**: Run Lynis quietly, then read the report.
- **Dispatch**: The agent runs the command remotely and fetches the report.
- **Review**: The agent synthesises the raw output into structured Markdown locally.

Example prompts:

```bash
opencode ask "Update system packages on server-33 and check for leftover services"
opencode ask "Create a crontab on big-pc that runs Lynis weekly and saves the report to /var/log/lynis-weekly.log"
```

There is no YAML, no Ansible playbook, and no Puppet manifest. The LLM reasons about the current state and generates appropriate commands on the fly.

#### Server Documentation and Migration

[Pedro Serey](https://pserey.substack.com/p/using-opencode-as-a-sysadmin) mapped a "black box" server into structured documentation using the Explore agent. Discovery commands (`lsblk`, `ip`, `systemctl list-units`) were synthesised into:

- `storage.md`: Drives, mount points, filesystems.
- `services/README.md`: Containers, Docker Compose files, environment variables.
- `network.md`: Interfaces, routes, firewall rules.

After mapping, the documentation served as the **spec** for a Proxmox migration plan. The agent generated a disciplined backup routine with proper database dumps, preventing data corruption.

> **Critical lesson**: After the agent staged unintended `git` commits during migration, the author pivoted to a **human-in-the-loop** workflow: Plan agent proposes commands, human executes in `tmux`, output is pasted back. This is Plan mode used as an SDD spec/review gate before execution.

#### AI-Powered Shell Command Generation

The [zsh-ask-opencode](https://github.com/andreacasarin/zsh-ask-opencode) plugin integrates OpenCode into ZSH. Press `Ctrl+O` to transform natural language into optimised shell commands:

```bash
$ list all files modified in the last 24 hours
# ^O generates: find . -mtime -1 -type f

$ compress all jpg files in this directory to 50% size
# ^O generates: convert *.jpg -quality 50% compressed.jpg
```

The plugin ranks three options by speed, safety, and reliability. You review before execution — a manual review gate for one-line tasks.

#### Interactive PTY Management

The [opencode-pty](https://github.com/JosXa/opencode-pty) plugin gives OpenCode control over pseudo-terminals. Unlike the synchronous `bash` tool, `pty_spawn` allows:

- Background processes (e.g., `tail -f /var/log/syslog`).
- Interactive input (`Ctrl+C`, arrow keys).
- Terminal snapshots without ANSI noise.
- Waiting until screen content matches a regex.

This is essential for sysadmin tasks involving long-running processes, such as monitoring deployments or stepping through interactive installers.

---

### DevOps and Infrastructure

#### Infrastructure-as-Code Generation

[ComputingForGeeks](https://computingforgeeks.com/ai-coding-agents-devops-terraform-ansible-kubernetes/) documented using OpenCode for DevOps workflows. The agent excels at generating boilerplate:

- **Terraform**: Variables, outputs, resource scaffolding.
- **Ansible**: Playbooks with SELinux awareness and handlers.
- **Kubernetes**: Deployment, Service, Ingress, HPA manifests.

Example prompt:

```bash
opencode run "Create Kubernetes manifests for a Python Flask app: a Deployment with 3 replicas, resource limits, health checks, and a non-root security context. Add a ClusterIP Service and an Ingress with TLS."
```

**Honest assessment**: AI agents consistently get provider version pinning and complex state dependencies wrong. Treat AI-generated infrastructure like a pull request from a junior engineer: always run `terraform plan` and test in staging.

SDD mapping for IaC:
- **Spec**: "We need a three-tier AWS architecture with VPC, private subnets, RDS, and an EKS cluster."
- **Tasks**: VPC module, subnet module, RDS module, EKS module, outputs.
- **Dispatch**: Each module gets a subagent. Terraform LSP validates syntax.
- **Review**: `terraform plan` is the spec compliance check. A second reviewer checks for security anti-patterns (open security groups, hardcoded secrets).

#### Multi-Agent CI/CD Pipeline Creation

Using `ultrawork` mode:

```bash
opencode run "@ultrawork Set up a complete CI/CD pipeline with GitHub Actions that builds a Docker image, pushes to ECR, runs Trivy security scan, deploys to EKS staging with Helm, runs integration tests, and promotes to production on approval."
```

The planner agent ensures cohesion:
- `.github/workflows/deploy.yml` references exact Helm values files.
- The Docker image tag propagates through every step.
- `scripts/integration-test.sh` hits the correct staging URL.

This is SDD at scale: one spec becomes dozens of coordinated files, with the planner acting as the Orchestrator ensuring cross-file consistency.

#### Project-Specific DevOps Agents

The [jon23d/opencode-configs](https://github.com/jon23d/opencode-configs) repository demonstrates a mature agent hierarchy:

- `dockerfile-best-practices/SKILL.md`
- `deployment-planning/SKILL.md`
- `kubernetes-manifests/SKILL.md`

The pipeline models a full software delivery lifecycle:

```
User request
  → build (clarify)
  → architect (plan)
  → build (review plan)
  → backend/frontend engineers (implement)
  → code-reviewer → security-reviewer → observability-reviewer
  → qa (E2E + OpenAPI)
  → devops-engineer (infra change)
  → developer-advocate (docs, docker-compose)
  → build (report)
```

This is SDD with specialised subagents for each review stage.

#### Kubernetes-Native Agents with KubeOpenCode

[KubeOpenCode](https://kubeopencode.github.io/kubeopencode/) runs OpenCode agents as Kubernetes CRDs:

```yaml
apiVersion: kubeopencode.io/v1alpha1
kind: Agent
metadata:
  name: dev-agent
spec:
  profile: "Interactive development agent"
  workspaceDir: /workspace
  persistence:
    sessions:
      size: "2Gi"
  standby:
    idleTimeout: "30m"
```

Attach from your terminal:

```bash
kubeoc agent attach default -n kubeopencode-system
```

This is ideal for CI/CD pipelines and team-shared agents. It uses a two-container pattern: an init container copies the OpenCode binary, and the worker container executes tasks.

#### Docker and Local Model Runners

Community projects provide ready-made Docker setups:

- [nimbleflux/opencode-docker](https://github.com/fluxbase-eu/opencode-docker): Docker image, Compose, and Helm chart.
- [utek/opencode-docker](https://github.com/utek/opencode-docker): Lightweight environment with Node.js, Python, Git, and GitHub CLI.
- [Docker Model Runner](https://docs.docker.com/guides/opencode-model-runner/): Connect OpenCode to locally served models.

---

## Security, Safety and Best Practices

### Default Permissions Are Permissive

By default, OpenCode enables most tools without approval. A [Reddit PSA](https://www.reddit.com/r/LocalLLaMA/comments/1r8oehn/opencode_arbitrary_code_execution_major_security/) highlighted that the agent can generate a Python script and immediately execute it. Lock this down:

```json
{
  "permission": {
    "bash": "ask",
    "edit": "ask",
    "write": "ask"
  }
}
```

### Use Plan Mode for Audits

For sysadmins and SREs, Plan mode is essential for auditing scripts and infrastructure-as-code without accidentally running anything.

### Human-in-the-Loop for Production

Even with good intentions, an agent with SSH access can stage unintended changes. The recommended workflow:

1. Agent proposes the command in Plan mode.
2. Human reviews and executes manually (e.g., in a `tmux` pane).
3. Human pastes the output back to the agent.

### Sandboxing

- **Containers/VMs**: Run OpenCode inside Docker or a VM.
- **OS-level sandboxing**: [nono](https://github.com/always-further/nono) uses Landlock (Linux) and Seatbelt (macOS) for default-deny access.

```bash
nono run --allow ./my_project_dir -- opencode
```

### Context Window Management

Developers have [reported](https://www.reddit.com/r/opencodeCLI/comments/1rr5kkg/are_you_running_out_of_context_tokens/) that context compaction can occur multiple times during large tasks. Mitigations:

- Enable `autoCompact: true`.
- Use the **Cartography** skill to generate codemaps.
- Break large tasks into smaller, scoped sessions.
- In SDD, fresh subagents per task naturally limit context sprawl.

---

## Advanced Configuration

### Custom Agents

Create project-specific agents by adding markdown files to `.opencode/agents/`:

```markdown
---
description: "Reviews code for best practices and potential issues"
mode: subagent
model: anthropic/claude-sonnet-4-20250514
permission:
  edit: deny
---
You are a code reviewer. Focus on security, performance, and maintainability.
Do not make changes; only provide feedback.
```

### Context Paths

Include additional context files:

```json
{
  "contextPaths": [
    ".github/copilot-instructions.md",
    ".cursorrules",
    "opencode.md"
  ]
}
```

### Auto-Compact

```json
{
  "autoCompact": true
}
```

### Session Management

- Use **multi-session** to work on multiple features in parallel.
- Use `/undo` and `/redo` to revert changes.
- Share session links with teammates.

---

## Conclusion

OpenCode represents a new paradigm in AI-assisted development: open-source, provider-agnostic, and deeply integrated into the tools developers already use. But the tool alone is not enough. **Subagent-Driven Development** provides the methodology that makes agent orchestration coherent, safe, and scalable.

The SDD workflow — explore, spec, decompose, dispatch, review, integrate — maps naturally to full-stack development, sysadmin operations, and DevOps infrastructure. Real-world practitioners have shown that this pattern scales from a 30-minute bookmark manager to multi-agent CI/CD pipelines, from natural-language server management to Kubernetes-native agent platforms.

Key takeaways:

1. **Start with a spec.** Use Plan mode, AGENTS.md, and the Cartography skill to build context before writing code.
2. **Decompose into independent tasks.** Each task gets a fresh subagent with isolated context.
3. **Enforce two-stage review.** Spec compliance first, code quality second. No task passes without both.
4. **Configure permissions carefully.** Set `bash` and `edit` to `ask` in production. Use sandboxes.
5. **Leverage specialised agents.** Let the Orchestrator route work; do not force one model to do everything.
6. **Use OpenCode Go** for reliable, low-cost access to curated open-source models.
7. **Keep a human in the loop** for production operations, especially when the agent has shell or SSH access.

Happy agentic coding!

---

## References

- [OpenCode Official Site](https://opencode.ai/)
- [OpenCode Documentation](https://open-code.ai/en/docs)
- [OpenCode Go](https://opencode.ai/go)
- [OpenCode GitHub Repository](https://github.com/opencode-ai/opencode)
- [OpenCode Agents Documentation](https://opencode.ai/docs/agents/)
- [OpenCode Rules Documentation](https://dev.opencode.ai/docs/rules/)
- [oh-my-opencode-slim GitHub](https://github.com/alvinunreal/oh-my-opencode-slim)
- [Oh My OpenCode (Original Plugin)](https://ohmyopencode.com/)
- [cc-sdd: Spec-Driven Development](https://github.com/gotalab/cc-sdd)
- [sdd-flow: Spec-Driven Development for OpenCode](https://github.com/Heldinhow/sdd-flow)
- [Agent Teams Lite](https://github.com/Gentleman-Programming/agent-teams-lite)
- [Gentleman AI: SDD Profiles](https://github.com/Gentleman-Programming/gentle-ai)
- [Subagent-Driven Development Tutorial](https://lzw.me/docs/opencodedocs/obra/superpowers/advanced/subagent-development/index.html)
- [Agentic Coding: Spec-Driven Development](https://agenticoding.ai/docs/practical-techniques/lesson-12-spec-driven-development)
- [DEV Community: Agent Orchestration in OpenCode](https://dev.to/gc-victor/agent-orchestration-in-opencode-3n4k)
- [Vercel KB: OpenCode with AI Gateway](https://vercel.com/kb/guide/how-i-use-opencode-with-vercel-ai-gateway-to-build-features-fast)
- [ZBuild: Full-Stack Bookmark Manager Tutorial](https://www.zbuild.io/resources/news/opencode-tutorial-2026)
- [JP Caparas: Multi-Agent Code Review](https://jpcaparas.medium.com/one-reviewer-three-lenses-building-a-multi-agent-code-review-system-with-opencode-21ceb28dde10)
- [Graphwiz: Vibe Coding with OpenCode](https://graphwiz.ai/ai/opencode-guide)
- [AI for You: OpenCode Practical Examples](https://aiforyou.life/use-cases/for-work/opencode-examples/)
- [argv.cloud: Agentic Sysadmin](https://argv.cloud/blog/2025/ai-remote-ops/)
- [Pedro Serey: Using OpenCode as a SysAdmin](https://pserey.substack.com/p/using-opencode-as-a-sysadmin)
- [zsh-ask-opencode GitHub](https://github.com/andreacasarin/zsh-ask-opencode)
- [opencode-pty GitHub](https://github.com/JosXa/opencode-pty)
- [ComputingForGeeks: AI Agents for DevOps](https://computingforgeeks.com/ai-coding-agents-devops-terraform-ansible-kubernetes/)
- [jon23d/opencode-configs GitHub](https://github.com/jon23d/opencode-configs)
- [KubeOpenCode](https://kubeopencode.github.io/kubeopencode/)
- [Docker Docs: OpenCode with Docker Model Runner](https://docs.docker.com/guides/opencode-model-runner/)
- [Sidekick Agent Hub (VS Code Extension)](https://github.com/cesarandreslopez/sidekick-agent-hub)
- [Reddit: E2B Cloud Sandboxes](https://www.reddit.com/r/opencodeCLI/comments/1r85nve/running_opencode_in_e2b_cloud_sandboxes_so_my/)
- [Reddit: Context Window Management](https://www.reddit.com/r/opencodeCLI/comments/1rr5kkg/are_you_running_out_of_context_tokens/)
- [Reddit: Security and Permissions](https://www.reddit.com/r/LocalLLaMA/comments/1r8oehn/opencode_arbitrary_code_execution_major_security/)
