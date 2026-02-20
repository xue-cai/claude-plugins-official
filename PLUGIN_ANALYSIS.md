# Claude Code Plugins — Comprehensive Analysis

> This document analyzes the Claude Code plugin ecosystem: what plugins are, how they work, use cases, and technical deep-dives into specific plugins.

---

## Table of Contents

1. [What Is a Claude Code Plugin?](#what-is-a-claude-code-plugin)
2. [Plugin Anatomy — Building Blocks](#plugin-anatomy--building-blocks)
3. [How Claude Code Uses Plugins](#how-claude-code-uses-plugins)
4. [Plugin Deep-Dives](#plugin-deep-dives)
   - [Agent SDK Dev](#1-agent-sdk-dev)
   - [Vercel](#2-vercel)
   - [Notion](#3-notion)
   - [Plugin Developer Toolkit](#4-plugin-developer-toolkit-plugin-dev)
   - [Stripe](#5-stripe)
   - [Firebase](#6-firebase)
5. [Additional Notable Plugins](#additional-notable-plugins)
6. [AWS / GCP / Azure Plugin Availability](#aws--gcp--azure-plugin-availability)
7. [Plugin Catalog Summary](#plugin-catalog-summary)

---

## What Is a Claude Code Plugin?

A Claude Code plugin is a **self-contained extension package** that adds new capabilities to Claude Code (Anthropic's agentic coding assistant). Plugins can teach Claude new skills, add slash commands users can invoke, spin up autonomous sub-agents, connect to external services via MCP (Model Context Protocol) servers, and inject event-driven hooks into Claude's workflow.

**Installation** is one command:

```
/plugin install {plugin-name}@claude-plugin-directory
```

Or browse interactively via `/plugin > Discover`.

Plugins live in two places in this repository:
- **`/plugins`** — Internal plugins developed by Anthropic (~29 plugins)
- **`/external_plugins`** — Third-party plugins from partners (~13 local + several URL-sourced)

A central registry at **`.claude-plugin/marketplace.json`** catalogs every plugin with metadata, descriptions, source locations, and optional LSP/MCP configurations.

---

## Plugin Anatomy — Building Blocks

Every plugin follows a standard directory structure:

```
plugin-name/
├── .claude-plugin/
│   └── plugin.json          # Plugin manifest (REQUIRED)
├── .mcp.json                # MCP server configuration (optional)
├── commands/                # Slash commands (optional)
│   └── my-command.md
├── agents/                  # Autonomous sub-agents (optional)
│   └── my-agent.md
├── skills/                  # Context-aware skill definitions (optional)
│   └── my-skill/
│       ├── SKILL.md
│       ├── references/
│       ├── examples/
│       └── scripts/
├── hooks/                   # Event-driven hooks (optional)
│   └── hooks.json
├── README.md                # Documentation
└── LICENSE
```

### The 6 Core Building Blocks

| Component | What It Is | How It Works |
|-----------|-----------|--------------|
| **`plugin.json`** (manifest) | Identity card for the plugin | Declares name, description, author, version, and metadata. Required. |
| **Commands** (`commands/*.md`) | Slash commands users type (e.g., `/deploy`) | Markdown files with YAML frontmatter (description, allowed-tools, model, argument-hint). The body is an instruction prompt Claude follows when the command is invoked. |
| **Skills** (`skills/*/SKILL.md`) | Contextual knowledge that activates automatically | Markdown with YAML frontmatter containing trigger phrases. Claude reads these when the user's request matches trigger conditions. Supports progressive disclosure via `references/`, `examples/`, `scripts/` subdirectories. |
| **Agents** (`agents/*.md`) | Autonomous sub-agents with specific roles | Markdown with YAML frontmatter (name, description, model, color, tools). The body defines the agent's system prompt. Claude delegates to these agents for specialized tasks. |
| **MCP Servers** (`.mcp.json`) | Connections to external services via Model Context Protocol | JSON config that tells Claude Code how to launch or connect to an MCP server (stdio, HTTP, SSE). Provides Claude with external tools (e.g., Stripe API, Notion API, Firebase). |
| **Hooks** (`hooks/hooks.json`) | Event-driven automations | Trigger on events like `PreToolUse`, `PostToolUse`, `Stop`, `SessionStart`. Can be prompt-based (LLM-driven) or command-based (deterministic scripts). |

### Minimal Plugin vs. Full-Featured Plugin

A **minimal plugin** (e.g., Firebase, Supabase) needs just two files:

```
firebase/
├── .claude-plugin/plugin.json     # Name + description
└── .mcp.json                      # MCP server connection
```

A **full-featured plugin** (e.g., Plugin Dev Toolkit) can have dozens of files across all component types.

---

## How Claude Code Uses Plugins

### Discovery & Loading
1. Claude Code reads `marketplace.json` to know what plugins exist
2. When a plugin is installed, Claude Code scans its directory structure and auto-discovers components
3. Commands become available as `/command-name` slash commands
4. Skills are loaded and matched against user prompts via their trigger descriptions
5. Agents are available for delegation when their description matches the task
6. MCP servers are started (stdio) or connected to (HTTP/SSE) automatically
7. Hooks are registered for their specified event types

### Runtime Behavior
- **Commands**: User types `/deploy` → Claude reads `commands/deploy.md` → follows its instructions
- **Skills**: User says "deploy my app to production" → Claude matches against skill descriptions → reads matching `SKILL.md` for expert knowledge → uses that knowledge in its response
- **Agents**: Claude encounters a task matching an agent's description → spawns a sub-agent with its own model, tools, and system prompt → agent works autonomously → returns results
- **MCP Tools**: Claude needs to call an external API → invokes MCP tool (e.g., `mcp__plugin_stripe__create_checkout_session`) → receives structured response
- **Hooks**: Claude is about to use a tool → `PreToolUse` hook fires → hook can approve, warn, or block the action

### Key Technical Concepts
- **`${CLAUDE_PLUGIN_ROOT}`** — A portable path variable that resolves to the plugin's install location. Used in all file references within plugins.
- **`$ARGUMENTS`** / **`$1`, `$2`, `$3`** — Dynamic argument substitution in commands
- **`@file-path`** — File reference syntax to include file contents in command prompts
- **`!`command``** — Inline bash execution in command prompts
- **`allowed-tools`** — Frontmatter field to restrict which tools a command/agent can use

---

## Plugin Deep-Dives

### 1. Agent SDK Dev

> **Category**: Development | **Author**: Anthropic | **Source**: `plugins/agent-sdk-dev`

#### USE CASE (Why)

Developers building applications with Anthropic's Claude Agent SDK need to scaffold projects correctly, follow best practices, and avoid common pitfalls. This plugin provides an **opinionated project scaffolding workflow** and **automated verification** for both Python and TypeScript Agent SDK projects.

**Typical user**: "I want to build a new AI agent using the Claude Agent SDK. Set up a TypeScript project for me."

#### TECHNICAL DETAILS (How)

**Components**:
- 1 command: `/new-sdk-app`
- 2 agents: `agent-sdk-verifier-py`, `agent-sdk-verifier-ts`

**Directory structure**:
```
agent-sdk-dev/
├── .claude-plugin/plugin.json
├── commands/
│   └── new-sdk-app.md          # Interactive project scaffolding
├── agents/
│   ├── agent-sdk-verifier-py.md  # Python project verifier
│   └── agent-sdk-verifier-ts.md  # TypeScript project verifier
└── README.md
```

**Workflow**:
1. User runs `/new-sdk-app`
2. Claude asks 5 interactive questions: language (Python/TypeScript), project name, application type, starting point, and tooling preferences
3. Claude checks for the latest SDK version via web search
4. Creates project structure with proper configuration (package.json/pyproject.toml, tsconfig, etc.)
5. Verifies code passes type checking / syntax validation
6. Automatically triggers the appropriate verifier agent

**Verifier agents** (model: `sonnet`) check:
- SDK installation correctness
- Environment setup (tsconfig.json, venv, etc.)
- SDK usage patterns and best practices
- Configuration, security, and error handling
- Returns a status: PASS / PASS WITH WARNINGS / FAIL

---

### 2. Vercel

> **Category**: Deployment | **Author**: Vercel | **Source**: External (URL-sourced from `github.com/vercel/vercel-deploy-claude-code-plugin`)

#### USE CASE (Why)

Frontend developers using Vercel want to **deploy, monitor, and debug** their applications without leaving their coding workflow. Instead of switching to the Vercel dashboard or running CLI commands manually, they can ask Claude to handle deployments conversationally.

**Typical user**: "Deploy my Next.js app to production" or "Show me the deployment logs for the last deploy."

#### TECHNICAL DETAILS (How)

**Components**:
- 3 commands: `/deploy`, `/logs` (vercel-logs), `/setup` (vercel-setup)
- 3 skills: `deploy`, `logs`, `setup`

**Directory structure**:
```
vercel-deploy-claude-code-plugin/
├── .claude-plugin/plugin.json
├── commands/
│   ├── deploy.md        # Deploy to production
│   ├── logs.md          # View deployment logs
│   └── setup.md         # Set up Vercel CLI and project
├── skills/
│   ├── deploy/SKILL.md  # Triggers on "deploy", "push to production"
│   ├── logs/SKILL.md    # Triggers on "show logs", "check deployment"
│   └── setup/SKILL.md   # Triggers on "set up Vercel", "configure Vercel"
└── README.md
```

**`plugin.json`**:
```json
{
  "name": "vercel",
  "version": "1.0.0",
  "description": "Deploy applications to Vercel with deployment monitoring, log analysis, and error detection",
  "author": { "name": "Vercel" }
}
```

**How `/deploy` works** (from `commands/deploy.md`):
1. Check prerequisites (`vercel --version`, `vercel whoami`)
2. If not set up, run `vercel login`
3. Run `vercel --prod` for production deployment
4. Display the deployment URL

**How the `deploy` skill works** (from `skills/deploy/SKILL.md`):
- Triggers when user says "deploy", "deploy to Vercel", "push to production", "deploy my app", or "go live"
- Guides Claude through prerequisite checks, deployment execution, and post-deployment status

**Key design pattern**: Commands provide explicit user-invoked actions; skills provide the same capabilities but triggered automatically by natural language. This dual approach means the plugin works whether the user types `/deploy` or just says "deploy my app."

---

### 3. Notion

> **Category**: Productivity | **Author**: Notion Labs | **Source**: External (URL-sourced from `github.com/makenotion/claude-code-notion-plugin`)

#### USE CASE (Why)

Development teams use Notion as their knowledge base, project tracker, and documentation hub. This plugin lets Claude **search, read, create, and update Notion content** directly — bridging the gap between coding and documentation workflows.

**Typical users**:
- "Search our Notion workspace for the API design spec"
- "Create a task in our Notion task board for this bug fix"
- "Write up a technical spec in Notion from our discussion"

#### TECHNICAL DETAILS (How)

**Components**:
- 10+ commands (including namespaced `tasks/` subcommands)
- 4 skills: Knowledge Capture, Meeting Intelligence, Research Documentation, Spec to Implementation
- 1 MCP server: Notion's hosted MCP at `https://mcp.notion.com/mcp`

**Directory structure**:
```
claude-code-notion-plugin/
├── .claude-plugin/plugin.json
├── .mcp.json                        # Notion MCP server
├── commands/
│   ├── search.md                    # Search workspace
│   ├── create-page.md               # Create a new page
│   ├── create-task.md               # Create task in database
│   ├── create-database-row.md       # Insert database row
│   ├── database-query.md            # Query a database
│   ├── find.md                      # Quick title search
│   └── tasks/
│       ├── setup.md                 # Set up task board
│       ├── build.md                 # Build task from URL
│       ├── plan.md                  # Plan from URL
│       └── explain-diff.md          # Generate doc from code changes
├── skills/
│   └── notion/
│       └── SKILL.md                 # 4 skills (Knowledge Capture, Meeting Intelligence, etc.)
└── README.md
```

**`.mcp.json`**:
```json
{
  "mcpServers": {
    "notion": {
      "type": "http",
      "url": "https://mcp.notion.com/mcp"
    }
  }
}
```

**How it works architecturally**:
1. The MCP server connects Claude to Notion's API, providing tools for searching, reading, creating, and updating pages/databases
2. Skills teach Claude *how* to work intelligently with Notion (e.g., proper page structure, meeting note formats, research documentation patterns)
3. Commands provide quick-access workflows for common operations

**Key design pattern**: This plugin combines all three major mechanisms — MCP (for API access), skills (for domain knowledge), and commands (for quick actions). The MCP server provides the *capability* (API tools), skills provide the *intelligence* (how to use those tools well), and commands provide the *convenience* (fast access to common workflows).

---

### 4. Plugin Developer Toolkit (`plugin-dev`)

> **Category**: Development | **Author**: Anthropic | **Source**: `plugins/plugin-dev`

#### USE CASE (Why)

Developers creating new Claude Code plugins need deep knowledge of the plugin system — hook events, MCP integration patterns, command frontmatter syntax, agent design, and more. This plugin is a **comprehensive reference and AI-assisted development toolkit** that encodes all that knowledge into skills Claude can use.

**Typical user**: "Help me create a new Claude Code plugin that adds Kubernetes deployment commands."

#### TECHNICAL DETAILS (How)

**Components**:
- 1 command: `/create-plugin` (guided 8-phase workflow)
- 3 agents: `agent-creator`, `plugin-validator`, `skill-reviewer`
- 7 skills covering every aspect of plugin development

**Directory structure**:
```
plugin-dev/
├── .claude-plugin/plugin.json
├── commands/
│   └── create-plugin.md                    # 8-phase guided plugin creation
├── agents/
│   ├── agent-creator.md                    # AI-assisted agent generation
│   ├── plugin-validator.md                 # Comprehensive validation
│   └── skill-reviewer.md                   # Skill quality review
└── skills/
    ├── hook-development/SKILL.md           # ~710 lines — event-driven hooks
    ├── mcp-integration/SKILL.md            # ~540 lines — MCP server patterns
    ├── plugin-structure/SKILL.md           # ~475 lines — directory layout & manifest
    ├── plugin-settings/SKILL.md            # ~545 lines — configuration patterns
    ├── command-development/SKILL.md        # ~835 lines — slash commands
    ├── agent-development/SKILL.md          # ~415 lines — autonomous agents
    └── skill-development/SKILL.md          # Skill creation methodology
```

**The `/create-plugin` workflow** (8 phases):
1. **Discovery** — Understand what the plugin should do
2. **Component Planning** — Determine which components are needed (skills, commands, agents, hooks, MCP)
3. **Detailed Design** — Ask clarifying questions for each component
4. **Plugin Structure Creation** — Set up directory and manifest
5. **Component Implementation** — Create each component using specialist agents
6. **Validation & Quality Check** — Run `plugin-validator` agent to catch issues
7. **Testing & Verification** — Test in Claude Code
8. **Documentation & Next Steps** — Finalize README and distribution

**Agent specifications**:
- **`agent-creator`** (model: sonnet, color: magenta, tools: Write/Read) — Translates user requirements into agent specifications with proper frontmatter and system prompts
- **`plugin-validator`** (model: inherit, color: yellow, tools: Read/Grep/Glob/Bash) — Validates plugin structure, manifest, naming conventions, security
- **`skill-reviewer`** (model: inherit, color: cyan, tools: Read/Grep/Glob) — Reviews skill quality, trigger effectiveness, progressive disclosure

**Skill highlights**:
- `hook-development` covers all hook events (PreToolUse, PostToolUse, Stop, SubagentStop, UserPromptSubmit, SessionStart, SessionEnd, PreCompact, Notification), prompt-based vs command-based hooks, matchers, exit codes, parallel execution
- `mcp-integration` covers all transport types (stdio, SSE, HTTP, WebSocket), OAuth patterns, tool naming conventions (`mcp__plugin_<name>_<server>__<tool>`), `${CLAUDE_PLUGIN_ROOT}` for portable paths
- `command-development` covers frontmatter fields, dynamic arguments, file references, bash execution, plugin-specific patterns

**Key design pattern**: This is a **meta-plugin** — a plugin for creating plugins. It demonstrates the full power of the skill system with progressive disclosure (lean SKILL.md → detailed references → working examples → utility scripts).

---

### 5. Stripe

> **Category**: Development | **Author**: Stripe | **Source**: `external_plugins/stripe`

#### USE CASE (Why)

Developers building payment integrations need to follow Stripe's best practices, handle errors correctly, and use the right APIs. This plugin provides **real-time API access** via MCP, **error explanation commands**, **test card references**, and **comprehensive integration best practices** as an auto-loaded skill.

**Typical users**:
- "Set up a Stripe Checkout integration for my SaaS app"
- "Explain this Stripe error: card_declined"
- "What test cards should I use for 3D Secure testing?"

#### TECHNICAL DETAILS (How)

**Components**:
- 2 commands: `/explain-error`, `/test-cards`
- 1 skill: `stripe-best-practices`
- 1 MCP server: Stripe's hosted MCP at `https://mcp.stripe.com`

**`.mcp.json`**:
```json
{
  "mcpServers": {
    "stripe": {
      "type": "http",
      "url": "https://mcp.stripe.com"
    }
  }
}
```

**How the `stripe-best-practices` skill works**: This is one of the most content-rich skills in the marketplace. It encodes Stripe's official integration guidance, including:
- Always prefer CheckoutSessions API over legacy Charges API
- Use Stripe-hosted Checkout or Embedded Checkout over custom Payment Element where possible
- Never recommend legacy Card Element, Sources API, or Tokens API
- Advise dynamic payment methods over hardcoded `payment_method_types`
- Proper subscription integration via Billing APIs
- Connect platform guidance (direct charges vs destination charges, controller properties)

This means when a developer asks Claude to build a Stripe integration, Claude automatically follows Stripe's latest official best practices — avoiding deprecated APIs and anti-patterns.

---

### 6. Firebase

> **Category**: Database | **Author**: Google | **Source**: `external_plugins/firebase`

#### USE CASE (Why)

Developers working with Google Firebase want to manage Firestore databases, authentication, cloud functions, hosting, and storage directly from their development workflow without switching to the Firebase console.

**Typical user**: "Set up Firebase authentication for my React app" or "Query my Firestore database."

#### TECHNICAL DETAILS (How)

This is an example of a **minimal MCP-only plugin** — just two files:

**Directory structure**:
```
firebase/
├── .claude-plugin/plugin.json
└── .mcp.json
```

**`.mcp.json`**:
```json
{
  "firebase": {
    "command": "npx",
    "args": ["-y", "firebase-tools@latest", "mcp"]
  }
}
```

**How it works**: Unlike HTTP-based MCP servers (Stripe, Notion, Supabase), Firebase uses a **stdio transport** — Claude Code launches `npx firebase-tools@latest mcp` as a local process and communicates with it via stdin/stdout. This is because Firebase tools run locally and interact with the user's Firebase project configuration.

**Key design contrast**: Compare Firebase (stdio, local process, 2 files) vs Stripe (HTTP, cloud-hosted, 2 commands + 1 skill + MCP). The approach depends on the service's architecture and how much domain knowledge needs to be embedded.

---

## Additional Notable Plugins

### Language Server Protocol (LSP) Plugins

Eleven LSP plugins provide Claude Code with **real-time code intelligence** for their respective languages. These are configured directly in `marketplace.json` via the `lspServers` field (no separate plugin directory needed for LSP config):

| Plugin | Language | Server Command |
|--------|----------|----------------|
| `typescript-lsp` | TypeScript/JavaScript | `typescript-language-server --stdio` |
| `pyright-lsp` | Python | `pyright-langserver --stdio` |
| `gopls-lsp` | Go | `gopls` |
| `rust-analyzer-lsp` | Rust | `rust-analyzer` |
| `clangd-lsp` | C/C++ | `clangd --background-index` |
| `php-lsp` | PHP | `intelephense --stdio` |
| `swift-lsp` | Swift | `sourcekit-lsp` |
| `kotlin-lsp` | Kotlin | `kotlin-lsp --stdio` |
| `csharp-lsp` | C# | `csharp-ls` |
| `jdtls-lsp` | Java | `jdtls` |
| `lua-lsp` | Lua | `lua-language-server` |

### Productivity & Workflow Plugins

| Plugin | Description |
|--------|-------------|
| **`code-review`** | Multi-agent PR review with confidence-based scoring to filter false positives |
| **`pr-review-toolkit`** | Specialized PR review agents for comments, tests, error handling, type design, code quality |
| **`commit-commands`** | Commands for git commit, push, and PR creation workflows |
| **`feature-dev`** | End-to-end feature development with codebase exploration, architecture design, and quality review |
| **`code-simplifier`** | Agent that simplifies code for clarity and maintainability |
| **`hookify`** | Create custom hooks via simple markdown rules |
| **`claude-code-setup`** | Analyze codebases and recommend tailored Claude Code automations |
| **`claude-md-management`** | Audit and maintain CLAUDE.md project memory files |

### External Service Integrations

| Plugin | Service | Transport |
|--------|---------|-----------|
| **`github`** | GitHub API | HTTP (Bearer token) |
| **`gitlab`** | GitLab DevOps | Local |
| **`slack`** | Slack Workspace | HTTP (OAuth) |
| **`supabase`** | Supabase Backend | HTTP |
| **`linear`** | Linear Issue Tracking | Local |
| **`asana`** | Asana Project Management | Local |
| **`playwright`** | Browser Automation (Microsoft) | Local |
| **`context7`** | Documentation Lookup (Upstash) | Local |

### URL-Sourced Plugins (not in this repo)

| Plugin | Source Repository |
|--------|-------------------|
| **Notion** | `github.com/makenotion/claude-code-notion-plugin` |
| **Vercel** | `github.com/vercel/vercel-deploy-claude-code-plugin` |
| **Atlassian** | `github.com/atlassian/atlassian-mcp-server` |
| **Figma** | `github.com/figma/mcp-server-guide` |
| **Sentry** | `github.com/getsentry/sentry-for-claude` |
| **Pinecone** | `github.com/pinecone-io/pinecone-claude-code-plugin` |
| **Hugging Face** | `github.com/huggingface/skills` |
| **PostHog** | `github.com/PostHog/posthog-for-claude` |
| **CodeRabbit** | `github.com/coderabbitai/claude-plugin` |
| **Firecrawl** | `github.com/firecrawl/firecrawl-claude-plugin` |
| **Semgrep** | `github.com/semgrep/mcp-marketplace` |
| **Sonatype Guide** | `github.com/sonatype/sonatype-guide-claude-plugin` |
| **Superpowers** | `github.com/obra/superpowers` |
| **Qodo Skills** | `github.com/qodo-ai/qodo-skills` |
| **CircleBack** | `github.com/circlebackai/claude-code-plugin` |

---

## AWS / GCP / Azure Plugin Availability

### In This Repository: None

**There are no dedicated AWS, GCP, or Azure plugins** in the official Claude Code plugins directory (`claude-plugins-official`). The only Google-related plugin is **Firebase**, which covers Firebase services specifically (Firestore, Auth, Hosting, Functions, Storage) rather than GCP broadly.

### In the Broader Ecosystem

Community-maintained plugins exist outside this official directory:

| Plugin | Source | Coverage |
|--------|--------|----------|
| **AWS Skills** | [github.com/zxkane/aws-skills](https://github.com/zxkane/aws-skills) | AWS CDK, serverless patterns, Bedrock, cost optimization |
| **Cloud Infrastructure** | [claudepluginhub.com](https://www.claudepluginhub.com/plugins/jadecli-cloud-infrastructure-plugins-wshobson-agents-plugins-cloud-infrastructure) | Multi-cloud (AWS, Azure, GCP, Kubernetes) — architecture, Terraform/IaC, cost optimization |

### Claude Code's Native Cloud Support

Claude Code itself can be used through cloud provider integrations:
- **AWS**: Via Amazon Bedrock for API access, or purchased through AWS Marketplace
- **GCP**: Via Google Vertex AI
- **Azure**: Via Azure AI Foundry (formerly Azure AI Studio)

However, these are backends for running Claude models — not development/deployment plugins for managing cloud infrastructure from Claude Code.

### Gap Analysis

This represents a significant **gap** in the official plugin marketplace. Common cloud development workflows that could benefit from dedicated plugins include:
- AWS CDK / CloudFormation template generation and validation
- Terraform / OpenTofu infrastructure-as-code assistance
- Kubernetes manifest generation and cluster management
- Cloud-native CI/CD pipeline configuration
- Cost estimation and optimization
- IAM policy analysis and security review

---

## Plugin Catalog Summary

### By Category

| Category | Count | Examples |
|----------|-------|---------|
| **Development** | 16+ | Agent SDK Dev, Feature Dev, Frontend Design, Plugin Dev, Playground |
| **Productivity** | 10+ | Code Review, Commit Commands, GitHub, Slack, Linear, Asana |
| **Language Servers** | 11 | TypeScript, Python, Go, Rust, C/C++, PHP, Swift, Kotlin, C#, Java, Lua |
| **Database** | 3 | Firebase, Supabase, Pinecone |
| **Security** | 3 | Security Guidance, Semgrep, Sonatype Guide |
| **Deployment** | 1 | Vercel |
| **Design** | 1 | Figma |
| **Testing** | 1 | Playwright |
| **Monitoring** | 2 | Sentry, PostHog |
| **Learning** | 2 | Explanatory Output Style, Learning Output Style |

### By Complexity

| Complexity | Pattern | Examples |
|------------|---------|----------|
| **Minimal** (2 files) | `plugin.json` + `.mcp.json` | Firebase, Supabase, Slack, GitHub |
| **Medium** (commands + skills) | Commands + Skills + optional MCP | Vercel, Stripe, Commit Commands |
| **Full-featured** (all components) | Commands + Skills + Agents + Hooks + MCP | Plugin Dev, Feature Dev, Agent SDK Dev |

### By Source Type

| Source Type | How It Works | Examples |
|-------------|-------------|----------|
| **Local directory** (`./plugins/...`) | Files in this repo, maintained by Anthropic | Agent SDK Dev, Code Review, Plugin Dev |
| **Local external** (`./external_plugins/...`) | Files in this repo, maintained by partners | Firebase, Stripe, GitHub, Slack |
| **URL-sourced** (`source.url`) | Cloned from external Git repository at install time | Vercel, Notion, Figma, Atlassian, Sentry |
