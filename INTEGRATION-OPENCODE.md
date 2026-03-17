# OpenCode Integration Guide

How to use `zbik-agents` with [OpenCode](https://opencode.ai) (TUI, custom agents, plugins, and MCP).

---

## Table of Contents

- [Feasibility Analysis: Claude Code vs OpenCode](#feasibility-analysis-claude-code-vs-opencode)
- [Prerequisites](#prerequisites)
- [Setup: Add zbik-agents to Your Project](#setup-add-zbik-agents-to-your-project)
- [How to Create a New Project (Step-by-Step)](#how-to-create-a-new-project-step-by-step)
  - [Version A: Local Only (no remote)](#version-a-local-only-no-remote)
  - [Version B: With GitHub Remote](#version-b-with-github-remote)
- [Using Agents in OpenCode](#using-agents-in-opencode)
  - [1. Custom Agents (recommended)](#1-custom-agents-recommended)
  - [2. AGENTS.md (project-wide rules)](#2-agentsmd-project-wide-rules)
  - [3. CLI Flags (one-off sessions)](#3-cli-flags-one-off-sessions)
  - [4. Plugins (lifecycle hooks)](#4-plugins-lifecycle-hooks)
  - [5. GitHub Integration (autonomous PRs)](#5-github-integration-autonomous-prs)
- [Model Configuration](#model-configuration)
- [Orchestration Flow](#orchestration-flow)
- [Quick Reference](#quick-reference)
- [File Structure](#file-structure)

---

## Feasibility Analysis: Claude Code vs OpenCode

| Mechanism | Claude Code | OpenCode | Parity |
|---|---|---|---|
| Project-wide instructions | `CLAUDE.md` with `@import` | `AGENTS.md` (falls back to `CLAUDE.md`) | Full |
| Specialized agent definitions | `.claude/agents/*.md` subagents | `.opencode/agents/*.md` custom agents | Full |
| Per-agent model selection | `model: opus` in frontmatter | `model: anthropic/claude-opus-4-6` in frontmatter | Full |
| Per-agent tool control | `tools:` in frontmatter | `tools:` + `permission:` in frontmatter | Full (richer) |
| Subagent delegation | Claude picks subagent automatically | Primary agents invoke subagents via Task tool; manual via `@name` | Full |
| File import / inheritance | `@../../path/file.md` | `instructions` array in `opencode.json` + `{file:path}` substitution | Comparable |
| Lifecycle hooks | `SessionStart`, `PostToolUse`, `SubagentStart` | Plugin system: `session.created`, `tool.execute.before/after`, 25+ events | Full (richer) |
| CLI / scripting | `claude -p "..."` with flags | `opencode run "..."` with `--model`, `--agent` flags | Full |
| Non-interactive mode | `claude -p` | `opencode run` | Full |
| Session persistence | `--continue`, `--resume` | `--continue`/`-c`, `--session`/`-s`, `/sessions` | Full |
| MCP support | Local MCP servers | Local + remote MCP with OAuth | Full (richer) |
| Multi-provider | Anthropic only | 75+ providers (Anthropic, OpenAI, Google, local Ollama, etc.) | OpenCode advantage |
| GitHub automation | Not built-in | `opencode github install` -- mention `/opencode` in issues/PRs | OpenCode advantage |
| Agent SDK (programmatic) | `claude_agent_sdk` | `opencode serve` HTTP API + plugin SDK | Comparable |

### Bottom Line

**OpenCode has the highest parity with Claude Code** of all three alternative integrations
(Copilot, Gastown, OpenCode). Every major mechanism maps directly:

- Custom agent `.md` files with YAML frontmatter work the same way
- `AGENTS.md` replaces `CLAUDE.md` (and falls back to `CLAUDE.md` if no `AGENTS.md` exists)
- Subagent delegation works via `@agent_name` or automatic Task tool invocation
- The plugin system is richer than Claude Code hooks (25+ event types vs 3)
- Multi-provider support means you can use Claude, GPT, Gemini, or local models
- CLI scripting via `opencode run` matches `claude -p`

**What changes:** Frontmatter field names differ slightly (`permission` instead of just
`tools`). Agent files go in `.opencode/agents/` instead of `.claude/agents/`. The
`@import` syntax is replaced by the `instructions` config array and `{file:}` substitution.

**What is gained:** 75+ model providers, remote MCP with OAuth, GitHub/GitLab automation,
plugin system with 25+ events, server mode (`opencode serve`), session sharing/export.

---

## Prerequisites

Install OpenCode (pick one):

```bash
# Homebrew (macOS/Linux)
brew install opencode

# npm
npm install -g opencode

# curl (Linux/macOS)
curl -fsSL https://opencode.ai/install | bash

# Go
go install github.com/opencode-ai/opencode@latest
```

Also required: Git 2.25+, an API key for at least one model provider.

### Connect a Provider

```bash
opencode
# Then inside the TUI:
/connect
# Follow prompts to add your Anthropic, OpenAI, or other API key
```

Or set environment variables directly:

```bash
export ANTHROPIC_API_KEY="sk-ant-..."
# or
export OPENAI_API_KEY="sk-..."
```

---

## Setup: Add zbik-agents to Your Project

Same three methods as Claude Code and Copilot. The agent definition files (`_base.md`,
`roles/*`, `specialists/*`) are identical -- only the integration layer changes.

### Method A: Simple Copy (quickest)

```bash
cp -r /path/to/zbik-agents your-project/zbik-agents
```

### Method B: Git Submodule (recommended for teams)

```bash
cd your-project
git submodule add https://github.com/<your-org>/zbik-agents.git zbik-agents
git commit -m "Add zbik-agents submodule"
```

### Method C: Symlink (personal use)

```bash
cd your-project
ln -s /path/to/zbik-agents zbik-agents
```

---

## How to Create a New Project (Step-by-Step)

### Version A: Local Only (no remote)

```bash
# 1. Create project
mkdir my-new-project && cd my-new-project
git init

# 2. Copy zbik-agents into project
cp -r /path/to/zbik-agents .

# 3. Create AGENTS.md (project-wide instructions)
cat > AGENTS.md << 'EOF'
# My New Project

## Team Standards

Follow all rules defined in zbik-agents/_base.md. Use the Read tool to load
zbik-agents/_base.md at the start of every task for the constitutional guardrails
(typing, testing, docstrings, security, ethics).

## Team Workflow

Follow the workflow defined in zbik-agents/CLAUDE.md. This defines the 4-phase
orchestration flow with human review gates:
1. BA researches and produces functional spec
2. Architect designs + Security hardens + DevOps plans rollout
3. PM creates delivery plan with milestones
4. Specialists implement in sprints

The Reviewer challenges all agents at every phase independently.

## Project-Specific

- Describe your project tech stack here
- Example: Python 3.12, PostgreSQL, React frontend
- Always use `.venv` virtual environment for Python
- Run `pytest` before committing
EOF

# 4. Create opencode.json
cat > opencode.json << 'EOF'
{
  "$schema": "https://opencode.ai/config.json",
  "model": "anthropic/claude-opus-4-6",
  "instructions": [
    "zbik-agents/_base.md",
    "zbik-agents/CLAUDE.md"
  ]
}
EOF

# 5. Create agent files
mkdir -p .opencode/agents

# --- L1 Roles (coordination agents) ---

cat > .opencode/agents/business_analyst.md << 'AGENT'
---
description: >
  Initiates all projects. Deep research from scientific papers, official docs,
  GitHub repos, verified blogs. Produces functional specs with three solution
  tiers and cost analysis.
mode: subagent
model: anthropic/claude-opus-4-6
tools:
  write: false
  edit: false
---

Use the Read tool to load zbik-agents/_base.md and zbik-agents/roles/business_analyst.md
before starting any work. Follow all rules defined in those files.

You are the Business Analyst. You are the first agent to engage on every project.
Research the topic provided, verify sources following the Research Hierarchy, and
produce a functional specification with three solution tiers (simplified,
transitional, full-flagged) and cost analysis.
AGENT

cat > .opencode/agents/architect.md << 'AGENT'
---
description: >
  System architecture design with mermaid and drawio diagrams. Three design tiers
  with technology justifications.
mode: subagent
model: anthropic/claude-opus-4-6
---

Use the Read tool to load zbik-agents/_base.md and zbik-agents/roles/architect.md
before starting any work. Follow all rules defined in those files.

You are the Architect. Design the system architecture based on the approved
functional spec. Produce three design tiers with diagrams, technology
justifications ("why this, not that"), and cost analysis.
AGENT

cat > .opencode/agents/project_manager.md << 'AGENT'
---
description: >
  Project coordination, sprint delivery planning, and progress tracking.
  Prepares three-tier delivery plans with milestones.
mode: subagent
model: anthropic/claude-opus-4-6
tools:
  write: false
  edit: false
---

Use the Read tool to load zbik-agents/_base.md and zbik-agents/roles/project_manager.md
before starting any work. Follow all rules defined in those files.

You are the Project Manager. Create delivery plans with milestones, sprint
breakdowns, specialist assignments, and external source documentation based on the
approved architecture.
AGENT

cat > .opencode/agents/reviewer.md << 'AGENT'
---
description: >
  Persistent critic and quality gate. Challenges all agents at every phase.
  Verifies types, tests, docs, and security.
mode: subagent
model: anthropic/claude-opus-4-6
tools:
  write: false
  edit: false
  bash: false
---

Use the Read tool to load zbik-agents/_base.md and zbik-agents/roles/reviewer.md
before starting any work. Follow all rules defined in those files.

You are the Reviewer (Critic). Critically evaluate the artifact provided. Check for
correctness, completeness, security, types, tests, and documentation. Produce a
structured review report with severity ratings. Never rubber-stamp.
AGENT

cat > .opencode/agents/researcher.md << 'AGENT'
---
description: >
  Technology research, evaluation, and architecture recommendations.
mode: subagent
model: anthropic/claude-opus-4-6
tools:
  write: false
  edit: false
---

Use the Read tool to load zbik-agents/_base.md and zbik-agents/roles/researcher.md
before starting any work. Follow all rules defined in those files.

You are the Researcher. Research the technology or topic provided. Evaluate
options with evidence, compare alternatives, and produce a recommendation.
AGENT

cat > .opencode/agents/developer.md << 'AGENT'
---
description: >
  General software development. Use for implementation tasks that don't need
  a specific specialist.
mode: subagent
model: anthropic/claude-opus-4-6
---

Use the Read tool to load zbik-agents/_base.md and zbik-agents/roles/developer.md
before starting any work. Follow all rules defined in those files.

You are the Developer. Implement the feature or fix described. All functions must
have type annotations, docstrings with examples, and unit tests.
AGENT

cat > .opencode/agents/product_owner.md << 'AGENT'
---
description: >
  Product vision, backlog prioritization, stakeholder alignment. Bridges
  business goals with technical execution.
mode: subagent
model: anthropic/claude-opus-4-6
tools:
  write: false
  edit: false
---

Use the Read tool to load zbik-agents/_base.md and zbik-agents/roles/product_owner.md
before starting any work. Follow all rules defined in those files.

You are the Product Owner. Define the product backlog, prioritize by business value,
write user stories with acceptance criteria, and make scope decisions.
AGENT

# --- L2 Specialists (implementation agents) ---

cat > .opencode/agents/backend_developer.md << 'AGENT'
---
description: >
  Backend development in Python (venv), Java, Rust, C, C++, SQL, and databases.
  Use for server-side code, APIs, database schemas, and data processing.
mode: subagent
model: anthropic/claude-opus-4-6
---

Use the Read tool to load zbik-agents/_base.md and
zbik-agents/specialists/backend_developer.md before starting any work.
Follow all rules defined in those files.

You are the Backend Developer. Implement the backend feature or fix described.
Python must use .venv. All functions must have type annotations, docstrings with
Args/Returns/Raises/Example, and unit tests.
AGENT

cat > .opencode/agents/frontend_developer.md << 'AGENT'
---
description: >
  Frontend development with TypeScript, React, and Next.js. Use for UI
  components, web apps, and accessibility work.
mode: subagent
model: anthropic/claude-opus-4-6
---

Use the Read tool to load zbik-agents/_base.md and
zbik-agents/specialists/frontend_developer.md before starting any work.
Follow all rules defined in those files.

You are the Frontend Developer. Implement the frontend feature described. Use
TypeScript with strict mode. All components must be accessible and tested.
AGENT

cat > .opencode/agents/mobile_app_developer.md << 'AGENT'
---
description: >
  Mobile development for iOS (Swift), Android (Kotlin), React Native, and
  Flutter. Use for native and cross-platform mobile apps.
mode: subagent
model: anthropic/claude-opus-4-6
---

Use the Read tool to load zbik-agents/_base.md and
zbik-agents/specialists/mobile_app_developer.md before starting any work.
Follow all rules defined in those files.

You are the Mobile App Developer. Implement the mobile feature described with
proper architecture, testing, and platform conventions.
AGENT

cat > .opencode/agents/devops_engineer.md << 'AGENT'
---
description: >
  CI/CD pipelines, Docker, Kubernetes, Helm, GitHub Actions, git flow, and
  local dev environment setup.
mode: subagent
model: anthropic/claude-opus-4-6
---

Use the Read tool to load zbik-agents/_base.md and
zbik-agents/specialists/devops_engineer.md before starting any work.
Follow all rules defined in those files.

You are the DevOps Engineer. Set up or modify the infrastructure, CI/CD, Docker,
Kubernetes, or deployment configuration as described.
AGENT

cat > .opencode/agents/qa_automation_engineer.md << 'AGENT'
---
description: >
  QA test framework design and automation. Provides test briefs, patterns,
  and templates to developers. Every feature must have automated tests.
mode: subagent
model: anthropic/claude-opus-4-6
---

Use the Read tool to load zbik-agents/_base.md and
zbik-agents/specialists/qa_automation_engineer.md before starting any work.
Follow all rules defined in those files.

You are the QA Automation Engineer. Design test frameworks, write test briefs,
create test patterns, and ensure every feature has automated tests before
developers start implementing.
AGENT

cat > .opencode/agents/security_developer.md << 'AGENT'
---
description: >
  Security design and implementation. Reviews architecture for vulnerabilities,
  creates threat models (STRIDE), implements auth and encryption.
mode: subagent
model: anthropic/claude-opus-4-6
---

Use the Read tool to load zbik-agents/_base.md and
zbik-agents/specialists/security_developer.md before starting any work.
Follow all rules defined in those files.

You are the Security Developer. Review designs or code for security
vulnerabilities, create threat models (STRIDE), and implement or recommend
security controls. Iterate with the Architect until the design is secure.
AGENT

echo "Created all 13 agent files in .opencode/agents/"

# 6. Commit
git add . && git commit -m "Add zbik-agents team setup for OpenCode"

# 7. Start OpenCode
opencode
```

---

### Version B: With GitHub Remote

Same as Version A, but with a GitHub remote and optional GitHub integration.

#### Steps 1-5: Same as Version A

Follow all steps above through creating the agent files.

#### Step 6: Create GitHub repo and push

```bash
gh repo create <your-org>/my-project --private --clone=false
git remote add origin git@github.com:<your-org>/my-project.git
git add . && git commit -m "Add zbik-agents team setup for OpenCode"
git push -u origin main
```

#### Step 7: (Optional) Install GitHub integration

This lets you mention `/opencode` or `/oc` in GitHub issue and PR comments
to trigger agent actions (triage, fix, review, etc.):

```bash
opencode github install
```

Follow the prompts to install the GitHub App and configure the workflow.

#### Step 8: Start OpenCode

```bash
opencode
```

```
You: I want to build a payment processing API with...

-> @business_analyst researches and produces functional spec
-> @reviewer challenges BA automatically
-> You review spec -> "approved"
-> @architect designs + @security_developer hardens + @devops_engineer plans rollout
-> You review -> "approved"
-> @project_manager plans delivery with milestones
-> You approve milestones, repos, data sources
-> @backend_developer, @frontend_developer, etc. implement in sprints
-> You review at milestones only
```

---

## Using Agents in OpenCode

### 1. Custom Agents (recommended)

Agent definitions live in `.opencode/agents/` (project-level) or
`~/.config/opencode/agents/` (global). Each file uses YAML frontmatter + markdown body.

#### Frontmatter Reference

```yaml
---
description: >                          # Required -- brief description
  Backend development in Python, Java, Rust, C, C++, SQL.
mode: subagent                          # subagent | primary | all
model: anthropic/claude-opus-4-6       # provider/model-id
temperature: 0.1                        # 0.0-1.0
steps: 50                              # Max agentic iterations
tools:                                  # Enable/disable specific tools
  write: true
  edit: true
  bash: true
  read: true
  grep: true
  glob: true
  list: true
  webfetch: true
  websearch: true
permission:                             # Fine-grained access control
  bash:
    "git *": allow
    "pytest *": allow
    "npm *": ask
    "*": ask
  task:
    "*": allow                          # Can invoke any subagent
---

System prompt / instructions here...
```

#### Agent Modes

| `mode:` value | Behavior |
|---|---|
| `subagent` | Invoked by primary agents or manually via `@name`. Recommended for all zbik-agents roles. |
| `primary` | Top-level agent (like the built-in Build/Plan agents). Appears in agent switcher. |
| `all` | Available as both primary and subagent (default if `mode` is omitted). |

#### Tool Control

Disable tools to restrict what an agent can do:

```yaml
# Read-only agent (reviewer, researcher, BA)
tools:
  write: false
  edit: false
  bash: false

# Full implementation agent (developers, devops)
tools:
  write: true
  edit: true
  bash: true
```

Or use the `permission` field for fine-grained control:

```yaml
permission:
  bash:
    "pytest *": allow
    "git status *": allow
    "git log *": allow
    "*": ask                # Everything else requires approval
```

#### Invoking Agents

In the OpenCode TUI:

```
# Manual invocation via @
@business_analyst Research payment processing frameworks for Python

@architect Design a microservices architecture for the payment system

@reviewer Review the code in src/payments/

@backend_developer Implement the payment service endpoint

@security_developer Create a threat model for the payment API
```

Primary agents can also automatically delegate to subagents via the Task tool
when they determine a task matches a subagent's description.

#### Listing and Managing Agents

```bash
# List all agents (CLI)
opencode agent list

# Create a new agent interactively
opencode agent create

# Inside the TUI
/agents
```

---

### 2. AGENTS.md (project-wide rules)

`AGENTS.md` is OpenCode's equivalent of `CLAUDE.md`. It is automatically loaded
into every interaction.

**`AGENTS.md`** (project root):
```markdown
# My Project

## Team Standards

Follow all rules in zbik-agents/_base.md. Use the Read tool to load it at the
start of every task for constitutional guardrails (typing, testing, docstrings,
security, ethics).

## Team Workflow

Follow the 4-phase workflow in zbik-agents/CLAUDE.md:
1. BA -> [Human Gate 1] -> Architect + Security + DevOps -> [Human Gate 2]
2. PM -> [Human Gate 3] -> Specialists + QA -> [Human Gate 4]
Reviewer challenges all agents at every step.

## Project-Specific

- Python 3.12, PostgreSQL, React frontend
- Always use `.venv` virtual environment
- Run `pytest` before committing
```

#### Including Additional Files

Use the `instructions` array in `opencode.json` to pull in additional files
automatically (OpenCode's equivalent of `@import`):

```json
{
  "instructions": [
    "zbik-agents/_base.md",
    "zbik-agents/CLAUDE.md",
    "CONTRIBUTING.md"
  ]
}
```

This supports file paths, glob patterns, and remote URLs:

```json
{
  "instructions": [
    "zbik-agents/_base.md",
    "zbik-agents/roles/*.md",
    "zbik-agents/specialists/*.md",
    "https://raw.githubusercontent.com/<org>/standards/main/coding-rules.md"
  ]
}
```

#### CLAUDE.md Fallback

If no `AGENTS.md` exists, OpenCode falls back to reading `CLAUDE.md` and
`~/.claude/CLAUDE.md`. This means **your existing Claude Code setup works
out of the box** -- just run `opencode` in a project that already has `CLAUDE.md`.

To disable this fallback:

```bash
export OPENCODE_DISABLE_CLAUDE_CODE=1
```

---

### 3. CLI Flags (one-off sessions)

#### Non-Interactive Mode

```bash
# Run a one-shot task with a specific agent
opencode run "Review the code in src/" --agent reviewer

# Run with a specific model
opencode run "Implement the user service in Python" \
  --agent backend_developer \
  --model anthropic/claude-opus-4-6

# Attach files for context
opencode run "Review this spec" \
  --agent reviewer \
  --file docs/functional-spec.md
```

#### Interactive TUI with Agent Pre-selected

```bash
# Start TUI with a specific agent
opencode --agent backend_developer

# Continue the last session
opencode --continue

# Resume a specific session
opencode --session <session-id>

# Fork a session (branch off from existing conversation)
opencode --fork --session <session-id>
```

#### Headless Server Mode

```bash
# Run as an HTTP API server (for programmatic orchestration)
opencode serve --port 8080

# Connect a TUI to a remote server
opencode attach http://localhost:8080
```

---

### 4. Plugins (lifecycle hooks)

OpenCode's plugin system is the equivalent of Claude Code hooks but significantly
more powerful, with 25+ event types.

#### Creating a Plugin

Place JavaScript/TypeScript files in `.opencode/plugins/` (project) or
`~/.config/opencode/plugins/` (global).

**`.opencode/plugins/coding-standards.js`**
```javascript
export default (ctx) => {
  // Remind about coding standards after every file edit
  ctx.on("file.edited", async (event) => {
    const ext = event.path.split(".").pop();
    if (["py", "ts", "js", "rs", "java"].includes(ext)) {
      ctx.client.toast(
        "Reminder: Ensure type annotations, docstrings with examples, and unit tests.",
        "info"
      );
    }
  });

  // Log session starts
  ctx.on("session.created", async (event) => {
    console.log(`Session started in ${ctx.directory}`);
  });

  // Inject context when agent goes idle
  ctx.on("session.idle", async (event) => {
    // Could trigger a reviewer pass, update todo list, etc.
  });
};
```

#### Available Events

| Event | When it fires | Claude Code equivalent |
|---|---|---|
| `session.created` | New session starts | `SessionStart` |
| `tool.execute.before` | Before any tool runs | `PreToolUse` |
| `tool.execute.after` | After any tool completes | `PostToolUse` |
| `file.edited` | File is modified | `PostToolUse` (Edit/Write matcher) |
| `permission.asked` | Permission prompt shown | No equivalent |
| `session.idle` | Agent finishes a task | No equivalent |
| `session.compacted` | Context window compressed | No equivalent |
| `message.part.updated` | Streaming response chunk | No equivalent |

#### Plugin Context

Plugins receive a context object with:

```javascript
export default (ctx) => {
  ctx.project;    // Project info
  ctx.directory;  // Working directory
  ctx.worktree;   // Git worktree path
  ctx.client;     // OpenCode SDK client
  ctx.$;          // Bun shell API for running commands
};
```

#### NPM Package Plugins

Reference npm packages in `opencode.json`:

```json
{
  "plugin": ["opencode-plugin-lint", "@my-org/opencode-coding-standards"]
}
```

---

### 5. GitHub Integration (autonomous PRs)

OpenCode can be triggered directly from GitHub issues and PR comments --
similar to Copilot's Coding Agent but triggered via comments rather than
issue assignment.

#### Setup

```bash
opencode github install
```

This installs a GitHub App and configures a GitHub Actions workflow.

#### Usage

In any GitHub issue or PR comment:

```
/opencode Fix the authentication bug described in this issue

/oc Review this PR for security vulnerabilities

/opencode Add unit tests for the payment service
```

OpenCode spins up in a GitHub Actions runner, reads the issue/PR context and
your `AGENTS.md` + `opencode.json` configuration, implements the change, and
creates a PR or pushes commits.

**Supported triggers:** `issue_comment`, `pull_request_review_comment`, `issues`,
`pull_request`, `schedule`, `workflow_dispatch`.

---

## Model Configuration

All agents use **Claude Opus 4.6** (`anthropic/claude-opus-4-6`) by default.

### Global Model

Set in `opencode.json`:

```json
{
  "model": "anthropic/claude-opus-4-6",
  "small_model": "anthropic/claude-haiku-4-5"
}
```

`small_model` is used for lightweight tasks (title generation, compaction, summaries).

### Per-Agent Model

Set in each `.opencode/agents/*.md` frontmatter:

```yaml
---
model: anthropic/claude-opus-4-6
---
```

Agents without a `model` field inherit the global model (primary agents) or
the invoking agent's model (subagents).

### Available Models

OpenCode supports 75+ providers. Common options for zbik-agents:

| `model:` value | Provider | Notes |
|---|---|---|
| `anthropic/claude-opus-4-6` | Anthropic | Most capable (default) |
| `anthropic/claude-sonnet-4-6` | Anthropic | Faster, cheaper |
| `anthropic/claude-haiku-4-5` | Anthropic | Fastest Anthropic option |
| `openai/gpt-5.2` | OpenAI | Latest OpenAI |
| `openai/gpt-4.1` | OpenAI | Strong general-purpose |
| `google/gemini-2.5-pro` | Google | Strong reasoning |
| `ollama/llama3` | Ollama | Local, free, private |

### Change All Agents at Once

```bash
# Switch all agents to Claude Sonnet 4.6
sed -i '' 's|model: anthropic/claude-opus-4-6|model: anthropic/claude-sonnet-4-6|g' \
  .opencode/agents/*.md

# Switch to GPT-5.2
sed -i '' 's|model: anthropic/claude-opus-4-6|model: openai/gpt-5.2|g' \
  .opencode/agents/*.md
```

### Mix Models

Edit individual agent files. Example cost-optimization strategy:

| Agent Type | Model | Rationale |
|---|---|---|
| L1 Roles (BA, Architect, PM, Reviewer) | `anthropic/claude-opus-4-6` | Complex reasoning |
| L2 Specialists (Backend, Frontend, etc.) | `anthropic/claude-sonnet-4-6` | Faster for implementation |
| Researcher | `anthropic/claude-opus-4-6` | Needs deep analysis |
| QA | `anthropic/claude-sonnet-4-6` | Test generation is formulaic |

### CLI Override

```bash
# Override model for a single session
opencode --model anthropic/claude-sonnet-4-6

# Override for a one-shot run
opencode run "Review this code" --model openai/gpt-5.2 --agent reviewer
```

---

## Orchestration Flow

The orchestration flow mirrors Claude Code. Primary agents can automatically
delegate to subagents via the Task tool, or you invoke agents manually with `@name`.

```
+--------------------------------------------------------------------+
|                                                                    |
|  1. @business_analyst --> [HUMAN GATE 1: approve spec]             |
|                      |                                             |
|  2. @architect + @security_developer + @devops_engineer            |
|               ------> [HUMAN GATE 2: approve design]               |
|                      |                                             |
|  3. @project_manager --> [HUMAN GATE 3: approve plan + sources]    |
|                      |                                             |
|  4. @backend_developer, @frontend_developer, etc.                  |
|               ------> [HUMAN GATE 4: milestone review]             |
|                                                                    |
|     @reviewer challenges ALL agents at EVERY step                  |
+--------------------------------------------------------------------+
```

### Step-by-step walkthrough

```
# Phase 1: Research & Spec
You: @business_analyst I want to build a payment processing API with...
  -> BA researches, produces functional spec with 3 tiers
You: @reviewer Review the BA's functional spec above
  -> Reviewer challenges findings
You: [review spec] -> "approved, proceed to architecture"

# Phase 2: Architecture
You: @architect Design the system based on the approved spec above
  -> Architect creates 3 design tiers with diagrams
You: @security_developer Review and harden the architecture
  -> Security dev creates threat model
You: @devops_engineer Review the architecture for rollout planning
You: @reviewer Challenge the architecture decisions
You: [review design] -> "approved, proceed to planning"

# Phase 3: Delivery Planning
You: @project_manager Create a delivery plan from the approved design
  -> PM produces 3-tier plan with milestones
You: @reviewer Challenge the delivery plan
You: [approve plan, repos, data sources, tier selection]

# Phase 4: Sprint Execution
You: @devops_engineer Set up git flow, Docker, Makefile, CI/CD
You: @qa_automation_engineer Create test briefs for the first sprint
You: @backend_developer Implement the user service
You: @reviewer Review the implementation
You: [review at milestones]
```

### Session Management

OpenCode sessions are persistent. You can leave and resume:

```bash
# Resume last session
opencode --continue

# List all sessions
opencode session list

# Resume a specific session
opencode --session <id>

# Fork a session to explore alternatives
opencode --fork --session <id>

# Share a session with teammates
/share
```

---

## Quick Reference

| Scenario | Method | Persistence |
|---|---|---|
| Team project, shared via git | Git submodule + agents in `.opencode/agents/` | Persistent, shared |
| Personal project | Copy or symlink + agents | Persistent, private |
| Project-wide coding standards | `AGENTS.md` + `instructions` in `opencode.json` | Persistent, shared |
| One-off agent session | `opencode run "..." --agent <name>` | Per-invocation |
| CI/CD pipeline | `opencode run` or GitHub integration | Per-invocation |
| Headless / programmatic | `opencode serve` HTTP API | Server-based |
| GitHub issue automation | `/opencode` in issue comments | Per-trigger |
| Auto-inject on file edits | Plugin in `.opencode/plugins/` | Persistent |
| Existing Claude Code project | Run `opencode` -- reads `CLAUDE.md` automatically | Fallback |

---

## File Structure

```
your-project/
├── AGENTS.md                                 # Project-wide rules (OpenCode reads this)
├── opencode.json                             # OpenCode config (model, instructions, MCP)
├── .opencode/
│   ├── agents/                               # Custom agent definitions
│   │   ├── business_analyst.md               # Initiates all projects (L1)
│   │   ├── architect.md                      # System design + diagrams (L1)
│   │   ├── project_manager.md                # Sprint delivery planning (L1)
│   │   ├── reviewer.md                       # Persistent critic (L1)
│   │   ├── researcher.md                     # On-demand tech research (L1)
│   │   ├── developer.md                      # Base developer (L1)
│   │   ├── product_owner.md                  # Product vision + backlog (L1)
│   │   ├── backend_developer.md              # Python/Java/Rust/C/C++/SQL (L2)
│   │   ├── frontend_developer.md             # TypeScript/React/Next.js (L2)
│   │   ├── mobile_app_developer.md           # Swift/Kotlin/Flutter (L2)
│   │   ├── devops_engineer.md                # CI/CD/Docker/K8s/Helm (L2)
│   │   ├── qa_automation_engineer.md         # Test frameworks (L2)
│   │   └── security_developer.md             # Auth/encryption/STRIDE (L2)
│   └── plugins/                              # Lifecycle plugins (optional)
│       └── coding-standards.js               # Post-edit reminders, etc.
└── zbik-agents/                              # Git submodule, copy, or symlink
    ├── CLAUDE.md                             # Workflow + standards (referenced by agents)
    ├── _base.md                              # L0 Constitutional guardrails
    ├── roles/                                # L1 Role Archetypes
    │   ├── business_analyst.md
    │   ├── architect.md
    │   ├── project_manager.md
    │   ├── reviewer.md
    │   ├── researcher.md
    │   ├── developer.md
    │   └── product_owner.md
    └── specialists/                          # L2 Technology Specializations
        ├── backend_developer.md
        ├── frontend_developer.md
        ├── mobile_app_developer.md
        ├── devops_engineer.md
        ├── qa_automation_engineer.md
        └── security_developer.md
```
