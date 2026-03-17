# GitHub Copilot Integration Guide

How to use `zbik-agents` with GitHub Copilot in VS Code (prompt files, instruction files, agent mode, and MCP).

---

## Table of Contents

- [Feasibility Analysis: Claude Code vs Copilot](#feasibility-analysis-claude-code-vs-copilot)
- [Prerequisites](#prerequisites)
- [Setup: Add zbik-agents to Your Project](#setup-add-zbik-agents-to-your-project)
- [How to Create a New Project (Step-by-Step)](#how-to-create-a-new-project-step-by-step)
- [Using Agents in GitHub Copilot](#using-agents-in-github-copilot)
  - [1. Prompt Files (recommended)](#1-prompt-files-recommended)
  - [2. Project Instructions (project-wide rules)](#2-project-instructions-project-wide-rules)
  - [3. Path-Specific Instruction Files (conditional rules)](#3-path-specific-instruction-files-conditional-rules)
  - [4. MCP Servers (external tool integration)](#4-mcp-servers-external-tool-integration)
  - [5. Copilot Coding Agent (autonomous PRs)](#5-copilot-coding-agent-autonomous-prs)
- [Model Configuration](#model-configuration)
- [Orchestration Flow](#orchestration-flow)
- [VS Code Settings Reference](#vs-code-settings-reference)
- [Quick Reference](#quick-reference)
- [File Structure](#file-structure)
- [What Cannot Be Replicated](#what-cannot-be-replicated)

---

## Feasibility Analysis: Claude Code vs Copilot

The zbik-agents approach relies on these core mechanisms:

| Mechanism | Claude Code | GitHub Copilot | Parity |
|---|---|---|---|
| Project-wide instructions | `CLAUDE.md` with `@import` | `.github/copilot-instructions.md` (or `CLAUDE.md` via setting) | Full |
| Specialized agent prompts | `.claude/agents/*.md` subagents | `.github/prompts/*.prompt.md` prompt files | Partial |
| Per-agent model selection | `model: opus` in frontmatter | `model: 'Claude Opus 4.6'` in frontmatter | Full |
| Per-agent tool control | `tools:` in frontmatter | `tools:` in frontmatter | Full |
| Agent mode (file edits + terminal) | Always on | `agent: 'agent'` in frontmatter | Full |
| File import / inheritance | `@../../path/file.md` | `[label](../../path/file.md)` markdown links | Partial |
| Conditional rules (by file type) | Hooks (`PostToolUse` with matcher) | `.instructions.md` files with `applyTo` glob | Comparable |
| Automatic subagent delegation | Claude picks the right subagent | Manual -- you invoke `/prompt_name` | None |
| CLI / scripting | `claude -p "..."` with flags | Not available (IDE-only) | None |
| Agent SDK (programmatic) | `claude_agent_sdk` / `claude-agent-sdk` | Not available | None |
| Lifecycle hooks | `SessionStart`, `PostToolUse`, `SubagentStart` | Not available | None |
| Parallel agents | Sequential subagents in one session | One prompt at a time | None |
| Autonomous PR creation | Not built-in | Copilot Coding Agent (assign issue to `@copilot`) | Copilot advantage |

### Bottom Line

**What works well:** The 12 agent roles, coding standards, three-tier frameworks, and
orchestration flow all port cleanly. Each role becomes a prompt file. The constitutional
guardrails (`_base.md`) go into project instructions. Per-agent model selection and tool
control work natively.

**What changes:** You manually invoke agents by typing `/business_analyst`, `/architect`,
etc. in Copilot Chat instead of Claude automatically delegating. The `@import` chain
becomes markdown links. There are no CLI flags or SDK -- everything is IDE-based.

**What is lost:** Automatic subagent delegation, lifecycle hooks, CLI scripting, and
programmatic orchestration via the Agent SDK. These are Claude Code-specific features
with no Copilot equivalent.

---

## Prerequisites

- **VS Code** 1.99+ with the GitHub Copilot extension
- **GitHub Copilot** subscription (Free, Pro, Business, or Enterprise)
- **Git** 2.25+

### Required VS Code Settings

Enable these in your VS Code settings (`.vscode/settings.json` or user settings):

```json
{
  "github.copilot.chat.codeGeneration.useInstructionFiles": true,
  "chat.agent.enabled": true,
  "chat.promptFilesLocations": {
    ".github/prompts": true
  },
  "chat.instructionsFilesLocations": {
    ".github/instructions": true
  }
}
```

Optional -- if you want Copilot to also read `CLAUDE.md` files (useful if you
already have a Claude Code setup and want both tools to share instructions):

```json
{
  "chat.useClaudeMdFile": true
}
```

---

## Setup: Add zbik-agents to Your Project

Same three methods as Claude Code. The agent definition files (`_base.md`, `roles/*`,
`specialists/*`) are identical -- only the integration layer (prompt files instead of
subagent files) changes.

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

### Step 1: Create your project directory

```bash
mkdir my-new-project && cd my-new-project
git init
```

### Step 2: Add zbik-agents (pick one method from above)

```bash
# Example: copy
cp -r /path/to/zbik-agents .
```

### Step 3: Create project instructions

```bash
mkdir -p .github
cat > .github/copilot-instructions.md << 'EOF'
# Project: My New Project

## Team Standards

The following file defines constitutional guardrails for all AI agents working
on this project. These rules (typing, testing, docstrings, security, ethics)
apply to every interaction.

[Base Agent Rules](../zbik-agents/_base.md)

## Team Workflow

[Team Workflow and Agents](../zbik-agents/CLAUDE.md)

## Project-Specific

- Describe your project tech stack here
- Example: Python 3.12, PostgreSQL, React frontend
- Always use `.venv` virtual environment for Python
- Run `pytest` before committing
EOF
```

### Step 4: Create prompt files for all 12 agents

```bash
mkdir -p .github/prompts

# --- L1 Roles (coordination agents) ---

cat > .github/prompts/business_analyst.prompt.md << 'PROMPT'
---
name: business_analyst
description: >
  Initiates all projects. Deep research from scientific papers, official docs,
  GitHub repos, verified blogs. Produces functional specs with three solution
  tiers and cost analysis.
agent: 'agent'
model: 'Claude Opus 4.6'
tools: ['search/codebase', 'githubRepo', 'fetch']
---

You are the Business Analyst. Follow these rules:

[Constitutional Guardrails](../../zbik-agents/_base.md)

[Business Analyst Role](../../zbik-agents/roles/business_analyst.md)

Your task: research the topic provided, verify sources, and produce a functional
specification with three solution tiers (simplified, transitional, full-flagged)
and cost analysis.
PROMPT

cat > .github/prompts/architect.prompt.md << 'PROMPT'
---
name: architect
description: >
  System architecture design with mermaid and drawio diagrams. Three design tiers
  with technology justifications.
agent: 'agent'
model: 'Claude Opus 4.6'
tools: ['search/codebase', 'githubRepo', 'editFiles', 'terminalCommand']
---

You are the Architect. Follow these rules:

[Constitutional Guardrails](../../zbik-agents/_base.md)

[Architect Role](../../zbik-agents/roles/architect.md)

Your task: design the system architecture based on the approved functional spec.
Produce three design tiers with diagrams, technology justifications, and cost
analysis.
PROMPT

cat > .github/prompts/project_manager.prompt.md << 'PROMPT'
---
name: project_manager
description: >
  Project coordination, sprint delivery planning, and progress tracking.
  Prepares three-tier delivery plans.
agent: 'agent'
model: 'Claude Opus 4.6'
tools: ['search/codebase', 'githubRepo']
---

You are the Project Manager. Follow these rules:

[Constitutional Guardrails](../../zbik-agents/_base.md)

[Project Manager Role](../../zbik-agents/roles/project_manager.md)

Your task: create a delivery plan with milestones, sprint breakdown, and
specialist assignments based on the approved architecture.
PROMPT

cat > .github/prompts/reviewer.prompt.md << 'PROMPT'
---
name: reviewer
description: >
  Persistent critic and quality gate. Challenges all agents at every phase.
  Verifies types, tests, docs, and security.
agent: 'agent'
model: 'Claude Opus 4.6'
tools: ['search/codebase']
---

You are the Reviewer (Critic). Follow these rules:

[Constitutional Guardrails](../../zbik-agents/_base.md)

[Reviewer Role](../../zbik-agents/roles/reviewer.md)

Your task: critically evaluate the artifact provided. Check for correctness,
completeness, security, types, tests, and documentation. Produce a structured
review report.
PROMPT

cat > .github/prompts/researcher.prompt.md << 'PROMPT'
---
name: researcher
description: >
  Technology research, evaluation, and architecture recommendations.
agent: 'agent'
model: 'Claude Opus 4.6'
tools: ['search/codebase', 'githubRepo', 'fetch']
---

You are the Researcher. Follow these rules:

[Constitutional Guardrails](../../zbik-agents/_base.md)

[Researcher Role](../../zbik-agents/roles/researcher.md)

Your task: research the technology or topic provided. Evaluate options with
evidence, compare alternatives, and produce a recommendation.
PROMPT

cat > .github/prompts/developer.prompt.md << 'PROMPT'
---
name: developer
description: >
  General software development. Use for implementation tasks that don't need
  a specific specialist.
agent: 'agent'
model: 'Claude Opus 4.6'
tools: ['search/codebase', 'editFiles', 'terminalCommand']
---

You are the Developer. Follow these rules:

[Constitutional Guardrails](../../zbik-agents/_base.md)

[Developer Role](../../zbik-agents/roles/developer.md)

Your task: implement the feature or fix described. Follow all coding standards
(typing, tests, docstrings).
PROMPT

cat > .github/prompts/product_owner.prompt.md << 'PROMPT'
---
name: product_owner
description: >
  Product vision, backlog prioritization, stakeholder alignment. Bridges
  business goals with technical execution.
agent: 'agent'
model: 'Claude Opus 4.6'
tools: ['search/codebase', 'githubRepo']
---

You are the Product Owner. Follow these rules:

[Constitutional Guardrails](../../zbik-agents/_base.md)

[Product Owner Role](../../zbik-agents/roles/product_owner.md)

Your task: define the product backlog, prioritize by business value, write user
stories with acceptance criteria, and make scope decisions.
PROMPT

# --- L2 Specialists (implementation agents) ---

cat > .github/prompts/backend_developer.prompt.md << 'PROMPT'
---
name: backend_developer
description: >
  Backend development in Python (venv), Java, Rust, C, C++, SQL, and databases.
  Use for server-side code, APIs, database schemas, and data processing.
agent: 'agent'
model: 'Claude Opus 4.6'
tools: ['search/codebase', 'editFiles', 'terminalCommand']
---

You are the Backend Developer. Follow these rules:

[Constitutional Guardrails](../../zbik-agents/_base.md)

[Backend Developer Specialization](../../zbik-agents/specialists/backend_developer.md)

Your task: implement the backend feature or fix described. Python must use .venv.
All functions must have type annotations, docstrings, and unit tests.
PROMPT

cat > .github/prompts/frontend_developer.prompt.md << 'PROMPT'
---
name: frontend_developer
description: >
  Frontend development with TypeScript, React, and Next.js. Use for UI
  components, web apps, and accessibility work.
agent: 'agent'
model: 'Claude Opus 4.6'
tools: ['search/codebase', 'editFiles', 'terminalCommand']
---

You are the Frontend Developer. Follow these rules:

[Constitutional Guardrails](../../zbik-agents/_base.md)

[Frontend Developer Specialization](../../zbik-agents/specialists/frontend_developer.md)

Your task: implement the frontend feature described. Use TypeScript with strict
mode. All components must be accessible and tested.
PROMPT

cat > .github/prompts/mobile_app_developer.prompt.md << 'PROMPT'
---
name: mobile_app_developer
description: >
  Mobile development for iOS (Swift), Android (Kotlin), React Native, and
  Flutter. Use for native and cross-platform mobile apps.
agent: 'agent'
model: 'Claude Opus 4.6'
tools: ['search/codebase', 'editFiles', 'terminalCommand']
---

You are the Mobile App Developer. Follow these rules:

[Constitutional Guardrails](../../zbik-agents/_base.md)

[Mobile App Developer Specialization](../../zbik-agents/specialists/mobile_app_developer.md)

Your task: implement the mobile feature described with proper architecture,
testing, and platform conventions.
PROMPT

cat > .github/prompts/devops_engineer.prompt.md << 'PROMPT'
---
name: devops_engineer
description: >
  CI/CD pipelines, Docker, Kubernetes, Helm, GitHub Actions, git flow, and
  local dev environment setup.
agent: 'agent'
model: 'Claude Opus 4.6'
tools: ['search/codebase', 'editFiles', 'terminalCommand', 'githubRepo']
---

You are the DevOps Engineer. Follow these rules:

[Constitutional Guardrails](../../zbik-agents/_base.md)

[DevOps Engineer Specialization](../../zbik-agents/specialists/devops_engineer.md)

Your task: set up or modify the infrastructure, CI/CD, Docker, Kubernetes,
or deployment configuration as described.
PROMPT

cat > .github/prompts/qa_automation_engineer.prompt.md << 'PROMPT'
---
name: qa_automation_engineer
description: >
  QA test framework design and automation. Provides test briefs, patterns,
  and templates to developers.
agent: 'agent'
model: 'Claude Opus 4.6'
tools: ['search/codebase', 'editFiles', 'terminalCommand']
---

You are the QA Automation Engineer. Follow these rules:

[Constitutional Guardrails](../../zbik-agents/_base.md)

[QA Automation Engineer Specialization](../../zbik-agents/specialists/qa_automation_engineer.md)

Your task: design test frameworks, write test briefs, create test patterns,
and ensure every feature has automated tests.
PROMPT

cat > .github/prompts/security_developer.prompt.md << 'PROMPT'
---
name: security_developer
description: >
  Security design and implementation. Reviews architecture for vulnerabilities,
  creates threat models (STRIDE), implements auth and encryption.
agent: 'agent'
model: 'Claude Opus 4.6'
tools: ['search/codebase', 'editFiles', 'terminalCommand']
---

You are the Security Developer. Follow these rules:

[Constitutional Guardrails](../../zbik-agents/_base.md)

[Security Developer Specialization](../../zbik-agents/specialists/security_developer.md)

Your task: review the design or code for security vulnerabilities, create
threat models, and implement or recommend security controls.
PROMPT

echo "Created all 13 prompt files in .github/prompts/"
```

### Step 5: (Optional) Create path-specific instruction files

These are the Copilot equivalent of Claude Code hooks -- rules that trigger
based on file patterns:

```bash
mkdir -p .github/instructions

cat > .github/instructions/python.instructions.md << 'EOF'
---
name: 'Python Standards'
description: 'Coding standards for Python files'
applyTo: '**/*.py'
---

- Always use `.venv` virtual environment
- Type annotations on all function signatures (params + return types)
- Docstrings with Args, Returns, Raises, and Example sections
- Unit tests for every function (place in tests/ directory)
- No bare `except:` -- always catch specific exceptions
- No hardcoded secrets or magic numbers
- Use `pathlib` over `os.path`
EOF

cat > .github/instructions/typescript.instructions.md << 'EOF'
---
name: 'TypeScript Standards'
description: 'Coding standards for TypeScript files'
applyTo: '**/*.ts,**/*.tsx'
---

- Strict mode enabled (`"strict": true` in tsconfig.json)
- Explicit return types on all exported functions
- JSDoc with `@param`, `@returns`, `@throws`, `@example` on public functions
- Unit tests for every function
- No `any` type -- use `unknown` and narrow
- No bare `catch(e)` -- always type the error
EOF
```

### Step 6: Create VS Code workspace settings

```bash
mkdir -p .vscode
cat > .vscode/settings.json << 'EOF'
{
  "github.copilot.chat.codeGeneration.useInstructionFiles": true,
  "chat.agent.enabled": true,
  "chat.promptFilesLocations": {
    ".github/prompts": true
  },
  "chat.instructionsFilesLocations": {
    ".github/instructions": true
  },
  "chat.includeReferencedInstructions": true
}
EOF
```

### Step 7: Commit the setup

```bash
git add .github/ .vscode/ zbik-agents/
git commit -m "Add zbik-agents team setup for GitHub Copilot"
```

### Step 8: Open in VS Code and start

```bash
code .
```

Open Copilot Chat (`Ctrl+L` / `Cmd+L`) and invoke the Business Analyst:

```
/business_analyst I want to build a payment processing API with...
```

---

## Using Agents in GitHub Copilot

### 1. Prompt Files (recommended)

Prompt files in `.github/prompts/` are the primary mechanism for using zbik-agents
with Copilot. Each agent is a `.prompt.md` file with YAML frontmatter.

#### Frontmatter Reference

```yaml
---
name: backend_developer              # Display name after / in chat
description: >                        # Short description
  Backend development in Python, Java, Rust, C, C++, SQL.
agent: 'agent'                        # Run in agent mode (can edit files, run terminal)
model: 'Claude Opus 4.6'             # Model selection (see Model Configuration)
tools: ['search/codebase', 'editFiles', 'terminalCommand']  # Available tools
argument-hint: 'describe the feature' # Hint text in chat input
---
```

#### Available Agent Modes

| `agent:` value | Behavior |
|---|---|
| `'agent'` | Full agent mode -- can edit files, run terminal commands, iterate |
| `'ask'` | Chat-only -- answers questions but doesn't modify files |
| `'plan'` | Creates an implementation plan without executing |

**Recommended settings per agent type:**

| Agent Type | `agent:` | Why |
|---|---|---|
| L1 Roles (BA, Architect, PM, Reviewer, Researcher, PO) | `'agent'` | Need to create docs, search repos |
| L2 Specialists (Backend, Frontend, Mobile, DevOps, QA, Security) | `'agent'` | Need to edit code, run tests |
| Reviewer (alternative) | `'ask'` | If you only want review feedback without edits |

#### Invoking Agents

In Copilot Chat:

```
/business_analyst Research payment processing frameworks for Python

/architect Design a microservices architecture for the payment system

/reviewer Review the code in src/payments/

/backend_developer Implement the payment service endpoint

/security_developer Create a threat model for the payment API
```

Or press `/` to see all available prompts and select one.

#### File References in Prompt Bodies

Prompt files reference zbik-agents definitions via markdown links:

```markdown
[Constitutional Guardrails](../../zbik-agents/_base.md)
[Backend Developer Specialization](../../zbik-agents/specialists/backend_developer.md)
```

Paths are relative to the prompt file's location. Since prompt files live in
`.github/prompts/`, use `../../zbik-agents/` to reach the project root.

---

### 2. Project Instructions (project-wide rules)

`.github/copilot-instructions.md` is the Copilot equivalent of Claude Code's
`CLAUDE.md`. It is automatically loaded into every Copilot interaction.

```markdown
# Project: My App

## Team Standards

[Base Agent Rules](../zbik-agents/_base.md)

## Team Workflow

[Team Workflow and Agents](../zbik-agents/CLAUDE.md)

## Project-Specific

- Python 3.12, PostgreSQL, React frontend
- Always use `.venv` virtual environment
- Run `pytest` before committing
```

**Key differences from CLAUDE.md:**

| Feature | CLAUDE.md | copilot-instructions.md |
|---|---|---|
| File import syntax | `@zbik-agents/_base.md` | `[label](../zbik-agents/_base.md)` |
| Import depth | 5 hops max | Requires `chat.includeReferencedInstructions: true` |
| Auto-loaded | Always | Controlled by `useInstructionFiles` setting |
| Location | Project root | `.github/` directory |

**Note:** If you enable `chat.useClaudeMdFile: true` in VS Code settings, Copilot
will also read your existing `CLAUDE.md` file. This lets you share one instruction
file between Claude Code and Copilot.

---

### 3. Path-Specific Instruction Files (conditional rules)

Files in `.github/instructions/*.instructions.md` apply only when working with
files matching the `applyTo` glob pattern. This is the Copilot equivalent of
Claude Code's `PostToolUse` hooks.

```yaml
---
name: 'Python Standards'
description: 'Coding standards for Python files'
applyTo: '**/*.py'
---

- Type annotations on all function signatures
- Docstrings with Args, Returns, Raises, and Example sections
- Unit tests for every function
```

The `applyTo` field accepts comma-separated glob patterns:

```yaml
applyTo: '**/*.ts,**/*.tsx,**/*.js'
```

You can also exclude specific agents from seeing the instruction:

```yaml
excludeAgent: 'code-review'
```

---

### 4. MCP Servers (external tool integration)

Copilot supports MCP (Model Context Protocol) for integrating external tools.
This is useful for giving agents access to GitHub, databases, APIs, etc.

#### GitHub MCP Server (built-in)

GitHub provides an official MCP server for repository operations:

```json
// .vscode/settings.json
{
  "mcp": {
    "servers": {
      "github": {
        "type": "remote",
        "url": "https://api.githubcopilot.com/mcp/"
      }
    }
  }
}
```

#### Custom MCP Servers

```json
// .vscode/mcp.json
{
  "servers": {
    "my-tools": {
      "command": "node",
      "args": ["./mcp-server/index.js"],
      "env": {
        "API_KEY": "${env:MY_API_KEY}"
      }
    }
  }
}
```

Reference MCP tools in prompt files:

```yaml
---
tools: ['my-tools/*', 'github/*']
---
```

---

### 5. Copilot Coding Agent (autonomous PRs)

The Copilot Coding Agent can autonomously create pull requests from GitHub Issues.
This has no Claude Code equivalent and is a Copilot-specific advantage.

**How it works:**

1. Create a GitHub Issue describing the task
2. Assign the issue to `@copilot`
3. Copilot spins up a cloud sandbox, analyzes the codebase, implements the change
4. Creates a PR with the implementation
5. Self-heals on build/test failures
6. Responds to review feedback

**It reads `.github/copilot-instructions.md`**, so your zbik-agents coding standards
apply automatically.

**Best for:** Phase 4 (sprint execution) tasks that are well-defined with clear
acceptance criteria from the Project Manager's delivery plan.

---

## Model Configuration

All prompt files use `model: 'Claude Opus 4.6'` by default.

The model is set in each `.github/prompts/*.prompt.md` file via YAML frontmatter:

```yaml
---
name: backend_developer
model: 'Claude Opus 4.6'    # <-- change model here
---
```

### Available Models (Copilot)

Copilot supports multiple model providers. Common options:

| `model:` value | Provider | Notes |
|---|---|---|
| `Claude Opus 4.6` | Anthropic | Most capable (default for all agents) |
| `Claude Sonnet 4.6` | Anthropic | Faster, cheaper |
| `Claude Haiku 4.5` | Anthropic | Fastest Anthropic option |
| `GPT-4.1` | OpenAI | Strong general-purpose |
| `GPT-5.2` | OpenAI | Latest OpenAI |
| `Gemini 2.5 Pro` | Google | Strong reasoning |

**Note:** The exact model name strings depend on what appears in the VS Code model
picker. Available models depend on your Copilot subscription tier. Different models
have different premium request multipliers.

### Change All Agents at Once

```bash
# Switch all agents to Claude Sonnet 4.6
sed -i '' "s/model: 'Claude Opus 4.6'/model: 'Claude Sonnet 4.6'/g" .github/prompts/*.prompt.md

# Switch to GPT-4.1
sed -i '' "s/model: 'Claude Opus 4.6'/model: 'GPT-4.1'/g" .github/prompts/*.prompt.md
```

### Mix Models (e.g., Opus for roles, Sonnet for specialists)

Edit individual prompt files. The model is set per-agent in each `.github/prompts/*.prompt.md`.

---

## Orchestration Flow

The orchestration flow is the same as Claude Code, but **you drive it manually**
by invoking prompt files in sequence. Copilot does not automatically delegate to
the right agent -- you choose which agent to engage.

```
+--------------------------------------------------------------------+
|                                                                    |
|  1. /business_analyst --> [HUMAN GATE 1: approve spec]             |
|                      |                                             |
|  2. /architect + /security_developer + /devops_engineer            |
|               ------> [HUMAN GATE 2: approve design]               |
|                      |                                             |
|  3. /project_manager --> [HUMAN GATE 3: approve plan + sources]    |
|                      |                                             |
|  4. /backend_developer, /frontend_developer, etc.                  |
|               ------> [HUMAN GATE 4: milestone review]             |
|                                                                    |
|     /reviewer challenges ALL agents at EVERY step (you invoke it)  |
+--------------------------------------------------------------------+
```

### Step-by-step walkthrough

```
# Phase 1: Research & Spec
You: /business_analyst I want to build a payment processing API with...
  -> BA researches, produces functional spec with 3 tiers
You: /reviewer Review the BA's functional spec above
  -> Reviewer challenges findings
You: [review spec] -> "approved, proceed to architecture"

# Phase 2: Architecture
You: /architect Design the system based on the approved spec above
  -> Architect creates 3 design tiers with diagrams
You: /security_developer Review and harden the architecture
  -> Security dev iterates with you on threat model
You: /devops_engineer Review the architecture for rollout planning
You: /reviewer Challenge the architecture decisions
You: [review design] -> "approved, proceed to planning"

# Phase 3: Delivery Planning
You: /project_manager Create a delivery plan from the approved design
  -> PM produces 3-tier plan with milestones
You: /reviewer Challenge the delivery plan
You: [approve plan, repos, data sources, tier selection]

# Phase 4: Sprint Execution
You: /devops_engineer Set up git flow, Docker, Makefile, CI/CD
You: /qa_automation_engineer Create test briefs for the first sprint
You: /backend_developer Implement the user service
You: /reviewer Review the implementation
You: [review at milestones]
```

**Tip:** Keep Copilot Chat history within a single session to maintain context
across agent invocations. Each `/agent_name` prompt starts with its role
definition but can see the conversation history above it.

---

## VS Code Settings Reference

### Core Settings

```json
{
  // Load .github/copilot-instructions.md automatically
  "github.copilot.chat.codeGeneration.useInstructionFiles": true,

  // Enable agent mode (file edits + terminal)
  "chat.agent.enabled": true,

  // Where to find prompt files
  "chat.promptFilesLocations": {
    ".github/prompts": true
  },

  // Where to find instruction files
  "chat.instructionsFilesLocations": {
    ".github/instructions": true
  },

  // Follow markdown links in instructions to pull in referenced files
  "chat.includeReferencedInstructions": true,

  // Auto-include instruction files matching applyTo patterns
  "chat.includeApplyingInstructions": true,

  // Max agent requests per session (increase for complex tasks)
  "chat.agent.maxRequests": 50,

  // Also read CLAUDE.md if you share instructions with Claude Code
  "chat.useClaudeMdFile": false
}
```

### Terminal Auto-Approval (optional)

```json
{
  // Auto-approve specific terminal commands
  "chat.tools.terminal.enableAutoApprove": true,
  "chat.tools.terminal.autoApprove": [
    "pytest",
    "npm test",
    "npm run lint",
    "make test",
    "git status",
    "git diff"
  ]
}
```

---

## Quick Reference

| Scenario | Method | Notes |
|---|---|---|
| Project-wide coding standards | `.github/copilot-instructions.md` | Auto-loaded |
| Invoke a specific agent role | `/agent_name` in Copilot Chat | From `.github/prompts/` |
| Python-only rules | `.github/instructions/python.instructions.md` with `applyTo` | Conditional |
| External tool integration | MCP servers in `.vscode/mcp.json` | GitHub, DBs, APIs |
| Autonomous PR from issue | Assign issue to `@copilot` on GitHub | Coding Agent |
| Share instructions with Claude Code | Enable `chat.useClaudeMdFile: true` | Reads `CLAUDE.md` |
| Change model for all agents | `sed` on `.github/prompts/*.prompt.md` | Per-file override |

---

## File Structure

```
your-project/
├── .github/
│   ├── copilot-instructions.md           # Project-wide rules (references zbik-agents/)
│   ├── prompts/                          # Prompt files (one per agent)
│   │   ├── business_analyst.prompt.md    # Initiates all projects (L1)
│   │   ├── architect.prompt.md           # System design + diagrams (L1)
│   │   ├── project_manager.prompt.md     # Sprint delivery planning (L1)
│   │   ├── reviewer.prompt.md            # Persistent critic (L1)
│   │   ├── researcher.prompt.md          # On-demand tech research (L1)
│   │   ├── developer.prompt.md           # Base developer (L1)
│   │   ├── product_owner.prompt.md       # Product vision + backlog (L1)
│   │   ├── backend_developer.prompt.md   # Python/Java/Rust/C/C++/SQL (L2)
│   │   ├── frontend_developer.prompt.md  # TypeScript/React/Next.js (L2)
│   │   ├── mobile_app_developer.prompt.md # Swift/Kotlin/Flutter (L2)
│   │   ├── devops_engineer.prompt.md     # CI/CD/Docker/K8s/Helm (L2)
│   │   ├── qa_automation_engineer.prompt.md # Test frameworks (L2)
│   │   └── security_developer.prompt.md  # Auth/encryption/STRIDE (L2)
│   └── instructions/                     # Path-specific rules (conditional)
│       ├── python.instructions.md        # Applied to **/*.py
│       └── typescript.instructions.md    # Applied to **/*.ts,**/*.tsx
├── .vscode/
│   └── settings.json                     # VS Code + Copilot settings
└── zbik-agents/                          # Git submodule, copy, or symlink
    ├── CLAUDE.md                         # Workflow + standards (referenced by prompts)
    ├── _base.md                          # L0 Constitutional guardrails
    ├── roles/                            # L1 Role Archetypes
    │   ├── business_analyst.md
    │   ├── architect.md
    │   ├── project_manager.md
    │   ├── reviewer.md
    │   ├── researcher.md
    │   ├── developer.md
    │   └── product_owner.md
    └── specialists/                      # L2 Technology Specializations
        ├── backend_developer.md
        ├── frontend_developer.md
        ├── mobile_app_developer.md
        ├── devops_engineer.md
        ├── qa_automation_engineer.md
        └── security_developer.md
```

---

## What Cannot Be Replicated

These Claude Code features have no Copilot equivalent. If you depend on them,
use Claude Code instead (or alongside Copilot).

| Feature | Why it matters | Workaround |
|---|---|---|
| Automatic subagent delegation | Claude picks the right agent for you | Manually invoke `/agent_name` |
| `@import` chaining (5 hops) | Deep file inheritance | Markdown links (requires `includeReferencedInstructions`) |
| Lifecycle hooks | Auto-inject context at session start, post-edit, etc. | Use instruction files with `applyTo` (partial) |
| CLI flags (`--append-system-prompt-file`) | One-off agent sessions from terminal | Not available -- IDE only |
| Agent SDK (Python/TypeScript) | Programmatic orchestration | Not available |
| Non-interactive mode (`claude -p`) | CI/CD integration | Use Copilot Coding Agent for GitHub Issues |
| Parallel subagents | Multiple agents in one session | One prompt at a time |
| Session hooks (`SubagentStart`) | Inject context when agents spawn | Not available |

### Using Both Tools Together

You can use Claude Code and Copilot on the same project. The zbik-agents definitions
are shared -- only the integration files differ:

- Claude Code reads: `CLAUDE.md`, `.claude/agents/`, `.claude/settings.json`
- Copilot reads: `.github/copilot-instructions.md`, `.github/prompts/`, `.github/instructions/`
- Both read: `zbik-agents/` (the agent definitions themselves)

Enable `chat.useClaudeMdFile: true` in VS Code to let Copilot also read `CLAUDE.md`,
giving both tools the same project-wide context.
