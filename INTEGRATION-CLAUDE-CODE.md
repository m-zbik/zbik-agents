# Claude Code Integration Guide

How to use `zbik-agents` with Claude Code (CLI, subagents, Agent SDK, and hooks).

---

## Table of Contents

- [Setup: Add zbik-agents to Your Project](#setup-add-zbik-agents-to-your-project)
  - [Method A: Simple Copy (quickest)](#method-a-simple-copy-quickest)
  - [Method B: Git Submodule (recommended for teams)](#method-b-git-submodule-recommended-for-teams)
  - [Method C: Symlink or Absolute Path (personal use)](#method-c-symlink-or-absolute-path-personal-use)
- [How to Create a New Project (Step-by-Step)](#how-to-create-a-new-project-step-by-step)
  - [Version A: Local Only (no remote)](#version-a-local-only-no-remote-no-push)
  - [Version B: With GitHub Remote](#version-b-with-github-remote)
- [Using Agents in Claude Code](#using-agents-in-claude-code)
  - [1. Subagents (recommended)](#1-subagents-recommended)
  - [2. Agent Teams (experimental -- multi-session)](#2-agent-teams-experimental----multi-session)
  - [3. CLAUDE.md (project-wide rules)](#3-claudemd-project-wide-rules)
  - [4. CLI Flags (one-off sessions)](#4-cli-flags-one-off-sessions)
  - [5. Hooks (context injection)](#5-hooks-context-injection)
  - [6. Agent SDK (programmatic)](#6-agent-sdk-programmatic)
- [Model Configuration](#model-configuration)
- [Orchestration Flow](#orchestration-flow)
- [Quick Reference](#quick-reference)
- [File Structure](#file-structure)

---

## Setup: Add zbik-agents to Your Project

Claude Code resolves `@` imports **relative to the file containing the `@` directive**.
There is no global registry -- the files must be reachable on disk from your project.
Choose one of three setup methods below.

### Method A: Simple Copy (quickest)

Copy `zbik-agents/` into your project and let Claude set up `.claude/agents/` for you.

**Step 1:** Copy the agents into your project root:

```bash
cp -r /Users/<your-user>/projects/zbik-agents your-project/zbik-agents
```

**Step 2:** Create your project's `CLAUDE.md` at the project root:

```markdown
# Project: My App

## Team

@zbik-agents/CLAUDE.md

## Project-Specific Instructions

- This is a Python 3.12 project using PostgreSQL
- Run `pytest` before committing
```

**Step 3:** Create the `.claude/agents/` subagent files. You can either:

(a) Copy them manually (see the [Subagent Files](#subagent-files) section below), or

(b) Ask Claude to do it for you in your first session:

```
> Set up the zbik-agents subagents. Create .claude/agents/ files for all 12 agents
  from zbik-agents/roles/ and zbik-agents/specialists/. Each subagent file should import
  @../../zbik-agents/_base.md and the corresponding role/specialist .md file.
```

**Step 4:** Start Claude Code and describe your project. The BA engages first:

```
> I want to build a payment processing API with...
```

**Pros:** Fully self-contained, no external dependencies, works offline.
**Cons:** Agents don't auto-update; you must re-copy to get changes.

---

### Method B: Git Submodule (recommended for teams)

Add `zbik-agents` as a git submodule. All team members get the same version,
and updates are a `git submodule update` away.

**Step 1:** Add the submodule:

```bash
cd your-project
git submodule add https://github.com/<your-org>/zbik-agents.git zbik-agents
git commit -m "Add zbik-agents submodule"
```

**Step 2:** Create your project's `CLAUDE.md` at the project root:

```markdown
# Project: My App

## Team

@zbik-agents/CLAUDE.md

## Project-Specific Instructions

- This is a Python 3.12 project using PostgreSQL
```

**Step 3:** Create `.claude/agents/` subagent files (same as Method A, Step 3).

**Step 4:** When cloning the repo elsewhere, init submodules:

```bash
git clone --recurse-submodules https://github.com/<your-org>/your-project.git
# Or if already cloned:
git submodule update --init --recursive
```

**Step 5:** To update agents to a new version:

```bash
cd your-project
git submodule update --remote zbik-agents
git add zbik-agents
git commit -m "Update zbik-agents to latest"
```

**Step 6:** When teammates pull and need the updated submodule:

```bash
git pull
git submodule update --init --recursive
```

**Pros:** Versioned, shared across team, easy to update.
**Cons:** Requires git submodule knowledge; new clones must init submodules.

---

### Method C: Symlink or Absolute Path (personal use)

For personal projects or quick experiments, symlink or use absolute paths.

**Option C1: Symlink**

```bash
cd your-project
ln -s /Users/<your-user>/projects/<your-project>/zbik-agents zbik-agents
```

Then use `@zbik-agents/CLAUDE.md` in your project's CLAUDE.md (same as Method A).

**Option C2: Absolute path in CLAUDE.md**

No setup needed -- just reference the absolute path directly:

```markdown
# In your project's CLAUDE.md
@/Users/<your-user>/projects/<your-project>/zbik-agents/CLAUDE.md
```

Or with `~`:

```markdown
@~/projects/<your-project>/zbik-agents/CLAUDE.md
```

**Option C3: Global (all projects)**

Add to `~/.claude/CLAUDE.md` to make the team available in every session:

```markdown
@~/projects/<your-project>/zbik-agents/CLAUDE.md
```

**Pros:** Zero copy, always up-to-date, works instantly.
**Cons:** Not portable; symlinks don't work for teammates; absolute paths are machine-specific.

---

### How @import Path Resolution Works

```
your-project/                          <-- project root
├── CLAUDE.md                          <-- "@zbik-agents/CLAUDE.md" resolves from here
│                                          (zbik-agents/ is a sibling directory)
├── .claude/
│   └── agents/
│       └── researcher.md              <-- "@../../zbik-agents/_base.md" resolves from here
│                                          (../../ goes up to project root)
└── zbik-agents/                    <-- submodule, copy, or symlink
    ├── CLAUDE.md                      <-- "@./_base.md" resolves from here (same dir)
    ├── _base.md
    ├── roles/
    └── specialists/
```

Rules:
- `@zbik-agents/file.md` -- relative to the file containing the `@`
- `@../../zbik-agents/file.md` -- relative path with `../` traversal
- `@~/path/to/file.md` -- resolves `~` to home directory
- `@/absolute/path/file.md` -- absolute path
- Maximum import depth: 5 hops

---

## How to Create a New Project (Step-by-Step)

Two versions: local-only (no GitHub needed) and with GitHub remote.

### Version A: Local Only (no remote, no push)

Everything stays on your machine. No GitHub account needed.

```bash
# 1. Create project
mkdir my-new-project && cd my-new-project
git init

# 2. Copy zbik-agents into project
cp -r /Users/<your-user>/projects/<your-project>/zbik-agents .

# 3. Create CLAUDE.md
cat > CLAUDE.md << 'EOF'
# My Project

## Team
@zbik-agents/CLAUDE.md

## Project-Specific
- Describe your project tech stack here
EOF

# 4. Create .claude/agents/ subagent files (see Step 4 in Version B below)
mkdir -p .claude/agents
for agent in business_analyst architect project_manager reviewer researcher developer; do
  cat > .claude/agents/${agent}.md << EOF
---
name: ${agent}
description: See zbik-agents/roles/${agent}.md
tools: Read, Grep, Glob, WebSearch, WebFetch, Bash
model: opus
---

@../../zbik-agents/_base.md
@../../zbik-agents/roles/${agent}.md
EOF
done

for agent in backend_developer frontend_developer mobile_app_developer devops_engineer qa_automation_engineer security_developer; do
  cat > .claude/agents/${agent}.md << EOF
---
name: ${agent}
description: See zbik-agents/specialists/${agent}.md
tools: Read, Edit, Write, Bash, Grep, Glob
model: opus
---

@../../zbik-agents/_base.md
@../../zbik-agents/specialists/${agent}.md
EOF
done

# 5. Commit
git add . && git commit -m "Add zbik-agents team setup"

# 6. Start Claude Code
claude
```

Everything is local. No remote, no push. You can add a remote later if you decide
to back things up:

```bash
git remote add origin git@github.com:<your-org>/my-project.git
git push -u origin main
```

---

### Version B: With GitHub Remote

Complete walkthrough from empty directory to running team with GitHub backup.

#### Step 1: Create your project directory

```bash
mkdir my-new-project && cd my-new-project
git init
```

#### Step 2: Add zbik-agents (pick one)

```bash
# Option A: Copy (simplest, self-contained)
cp -r /Users/<your-user>/projects/<your-project>/zbik-agents .

# Option B: Git submodule (versioned, shared with team)
git submodule add https://github.com/<your-org>/zbik-agents.git zbik-agents

# Option C: Symlink (always up-to-date, not portable)
ln -s /Users/<your-user>/projects/<your-project>/zbik-agents zbik-agents
```

#### Step 3: Create CLAUDE.md at project root

```bash
cat > CLAUDE.md << 'EOF'
# My New Project

## Team
@zbik-agents/CLAUDE.md

## Project-Specific
- Describe your project tech stack here
- Example: Python 3.12, PostgreSQL, React frontend
EOF
```

#### Step 4: Create .claude/agents/ subagent files

```bash
mkdir -p .claude/agents

# Create all 12 subagent files in one go
for agent in business_analyst architect project_manager reviewer researcher developer; do
  cat > .claude/agents/${agent}.md << EOF
---
name: ${agent}
description: See zbik-agents/roles/${agent}.md
tools: Read, Grep, Glob, WebSearch, WebFetch, Bash
model: opus
---

@../../zbik-agents/_base.md
@../../zbik-agents/roles/${agent}.md
EOF
done

for agent in backend_developer frontend_developer mobile_app_developer devops_engineer qa_automation_engineer security_developer; do
  cat > .claude/agents/${agent}.md << EOF
---
name: ${agent}
description: See zbik-agents/specialists/${agent}.md
tools: Read, Edit, Write, Bash, Grep, Glob
model: opus
---

@../../zbik-agents/_base.md
@../../zbik-agents/specialists/${agent}.md
EOF
done
```

#### Step 5: (Optional) Add hooks for auto-injection

```bash
mkdir -p .claude
cat > .claude/settings.json << 'EOF'
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "cat zbik-agents/_base.md"
          }
        ]
      }
    ]
  }
}
EOF
```

#### Step 6: Commit the setup

```bash
git add CLAUDE.md .claude/ zbik-agents/
git commit -m "Add zbik-agents team setup"
```

#### Step 7: Start Claude Code and begin

```bash
claude
```

```
You: I want to build [describe your project]

-> BA engages first, researches, produces functional spec
-> Reviewer challenges BA independently
-> You review spec -> "approved, proceed to architecture"
-> Architect designs -> Security hardens -> DevOps plans rollout
-> You review design -> "approved, proceed to planning"
-> PM creates delivery plan with milestones
-> You approve plan, repos, data sources, tier selection
-> DevOps sets up git flow, Docker, Makefile, CI/CD, Helm
-> Specialists implement in sprints
-> You review at milestones only
```

---

## Using Agents in Claude Code

### 1. Subagents (recommended)

Place agent definitions in `.claude/agents/` (project-level, shared via git) or `~/.claude/agents/` (user-level).

#### Subagent Files

Each file uses YAML frontmatter + markdown body. The body is the system prompt; the frontmatter controls config.

**`.claude/agents/business_analyst.md`**
```markdown
---
name: business_analyst
description: Initiates all projects. Deep research from scientific papers, official docs, GitHub repos, verified blogs. Produces functional specs with three solution tiers and cost analysis.
tools: Read, Grep, Glob, WebSearch, WebFetch, Bash
model: opus
---

@../../zbik-agents/_base.md
@../../zbik-agents/roles/business_analyst.md
```

**`.claude/agents/architect.md`**
```markdown
---
name: architect
description: System architecture design with mermaid and drawio diagrams. Three design tiers with technology justifications. Use when designing systems or selecting technologies.
tools: Read, Edit, Write, Bash, Grep, Glob
model: opus
---

@../../zbik-agents/_base.md
@../../zbik-agents/roles/architect.md
```

**`.claude/agents/project_manager.md`**
```markdown
---
name: project_manager
description: Project coordination, sprint delivery planning, and progress tracking. Prepares three-tier delivery plans. Keeps QA synchronized with developers.
tools: Read, Grep, Glob, Bash
model: opus
---

@../../zbik-agents/_base.md
@../../zbik-agents/roles/project_manager.md
```

**`.claude/agents/reviewer.md`**
```markdown
---
name: reviewer
description: Persistent critic and quality gate. Challenges all agents at every phase. Verifies types, tests, docs, and security.
tools: Read, Grep, Glob, Bash
model: opus
---

@../../zbik-agents/_base.md
@../../zbik-agents/roles/reviewer.md
```

**`.claude/agents/researcher.md`**
```markdown
---
name: researcher
description: Technology research, evaluation, and architecture recommendations. Use when evaluating tech choices, comparing frameworks, or investigating solutions.
tools: Read, Grep, Glob, WebSearch, WebFetch, Bash
model: opus
---

@../../zbik-agents/_base.md
@../../zbik-agents/roles/researcher.md
```

**`.claude/agents/developer.md`**
```markdown
---
name: developer
description: General software development. Use for implementation tasks that don't need a specific specialist.
tools: Read, Edit, Write, Bash, Grep, Glob
model: opus
---

@../../zbik-agents/_base.md
@../../zbik-agents/roles/developer.md
```

**`.claude/agents/backend_developer.md`**
```markdown
---
name: backend_developer
description: Backend development in Python (venv), Java, Rust, C, C++, SQL, and databases. Use for server-side code, APIs, database schemas, and data processing.
tools: Read, Edit, Write, Bash, Grep, Glob
model: opus
---

@../../zbik-agents/_base.md
@../../zbik-agents/specialists/backend_developer.md
```

**`.claude/agents/frontend_developer.md`**
```markdown
---
name: frontend_developer
description: Frontend development with TypeScript, React, and Next.js. Use for UI components, web apps, and accessibility work.
tools: Read, Edit, Write, Bash, Grep, Glob
model: opus
---

@../../zbik-agents/_base.md
@../../zbik-agents/specialists/frontend_developer.md
```

**`.claude/agents/mobile_app_developer.md`**
```markdown
---
name: mobile_app_developer
description: Mobile development for iOS (Swift), Android (Kotlin), React Native, and Flutter. Use for native and cross-platform mobile apps.
tools: Read, Edit, Write, Bash, Grep, Glob
model: opus
---

@../../zbik-agents/_base.md
@../../zbik-agents/specialists/mobile_app_developer.md
```

**`.claude/agents/devops_engineer.md`**
```markdown
---
name: devops_engineer
description: CI/CD pipelines, Docker, Kubernetes, Helm, GitHub Actions, git flow, and local dev environment setup. Use for deployment, infrastructure, and pipeline work.
tools: Read, Edit, Write, Bash, Grep, Glob
model: opus
---

@../../zbik-agents/_base.md
@../../zbik-agents/specialists/devops_engineer.md
```

**`.claude/agents/qa_automation_engineer.md`**
```markdown
---
name: qa_automation_engineer
description: QA test framework design and automation. Provides test briefs, patterns, and templates to developers. Every feature must have automated tests.
tools: Read, Edit, Write, Bash, Grep, Glob
model: opus
---

@../../zbik-agents/_base.md
@../../zbik-agents/specialists/qa_automation_engineer.md
```

**`.claude/agents/security_developer.md`**
```markdown
---
name: security_developer
description: Security design and implementation. Reviews architecture for vulnerabilities, creates threat models (STRIDE), implements auth and encryption. Iterates with architect until design is secure.
tools: Read, Edit, Write, Bash, Grep, Glob
model: opus
---

@../../zbik-agents/_base.md
@../../zbik-agents/specialists/security_developer.md
```

#### Using Subagents in Conversation

Claude Code automatically delegates to subagents based on the task, or you can invoke them explicitly following the orchestration flow:

```
> Start a new project to build a payment processing system
  -> business_analyst initiates: researches, produces functional spec with 3 tiers

> Have the architect design the system based on the BA's spec
  -> architect creates 3 design tiers with diagrams

> Ask the security_developer to review and harden the architecture
  -> security_developer + architect iterate until secure

> Have the project_manager plan the delivery
  -> project_manager produces 3-tier delivery plan, selects specialists

> Use the reviewer to challenge the BA's findings
  -> reviewer challenges with "why?", "how much?", "what if it fails?"
```

Manage agents interactively with `/agents`.

---

### 2. Agent Teams (experimental -- multi-session)

> **Requires:** Claude Code v2.1.32+, experimental flag enabled.

Agent Teams are fundamentally different from subagents. While subagents run within
a single session and report results back to the caller, Agent Teams spawn **independent
Claude Code instances** that coordinate through a shared task list and communicate
directly with each other.

| | Subagents | Agent Teams |
|---|---|---|
| **Context** | Own context; results return to caller | Own context; fully independent |
| **Communication** | Report back to main agent only | Teammates message each other directly |
| **Coordination** | Main agent manages all work | Shared task list with self-coordination |
| **Best for** | Focused tasks where only the result matters | Complex work requiring discussion and collaboration |
| **Token cost** | Lower: results summarized back | Higher: each teammate is a separate instance |

**Use subagents** (section 1 above) for the sequential, gated workflow defined in CLAUDE.md.
**Use Agent Teams** when teammates need to share findings, challenge each other, and
coordinate on their own -- especially for parallel research, review, and implementation.

#### Setup: Enable Agent Teams

The experimental flag and quality-gate hooks are already configured in this repository's
`.claude/settings.json`. If you are setting up a **new project**, copy the config below
into your project's `.claude/settings.json`:

```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  },
  "hooks": {
    "TaskCompleted": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "echo 'QUALITY GATE -- before marking complete, verify: (1) strong typing on ALL function signatures (params + return types), (2) unit tests for every function, (3) docstrings with Args, Returns, Raises, and Example sections, (4) no bare except/catch -- specific exceptions only, (5) no hardcoded secrets or magic numbers.'"
          }
        ]
      }
    ],
    "TeammateIdle": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "echo 'IDLE CHECK -- before going idle, confirm: (1) all tests pass, (2) all functions have type annotations and docstrings, (3) no TODO/FIXME left unaddressed, (4) code reviewed against _base.md constitutional guardrails.'"
          }
        ]
      }
    ]
  }
}
```

#### Setup CLI Command

Run these two commands in your project directory. The first saves the prompt to a
file, the second passes it to Claude Code:

**Step 1:** Create the setup prompt file:

```bash
cat > /tmp/zbik-teams-setup.txt << 'PROMPT'
Set up Agent Teams for this project using zbik-agents. Do the following:

1. Verify .claude/settings.json has CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1 in env, plus TaskCompleted and TeammateIdle hooks enforcing zbik-agents coding standards (strong typing, unit tests, docstrings, no bare except, no hardcoded secrets). If the settings file is missing or incomplete, create/update it.

2. Ensure .claude/agents/ subagent files exist for all 13 agents (product_owner, business_analyst, researcher, reviewer, architect, security_developer, devops_engineer, project_manager, backend_developer, frontend_developer, mobile_app_developer, qa_automation_engineer, developer). Each file must import @../../zbik-agents/_base.md and its role/specialist .md.

3. Confirm the setup by listing all agent files and the settings.json contents.

The team is organized into 4 phase-based compositions:
- Phase 1 Research & Spec (3 teammates): business_analyst, researcher, reviewer
- Phase 2 Architecture & Security (4 teammates): architect, security_developer, devops_engineer, reviewer
- Phase 3 Delivery Planning (2 teammates): project_manager, reviewer
- Phase 4 Implementation (5 teammates, varies by project): backend_developer, frontend_developer, qa_automation_engineer, devops_engineer, reviewer

product_owner maps to the team lead (the human-interactive session).
reviewer is spawned in every phase as the persistent critic.
PROMPT
```

**Step 2:** Run Claude with the prompt file:

```bash
claude -p "$(cat /tmp/zbik-teams-setup.txt)"
```

#### zbik-agents Team Mapping

The 13 zbik-agents map to Agent Teams roles as follows. The human-interactive session
acts as the team lead; teammates are spawned per phase.

| zbik-agent | Team Role | Phase(s) | Notes |
|---|---|---|---|
| **product_owner** | Team lead (human session) | All | Vision, backlog, stakeholder alignment -- the human + lead session |
| **business_analyst** | Teammate | 1: Research | Deep research, functional spec, 3 solution tiers |
| **researcher** | Teammate | 1: Research | Tech research, framework evaluation |
| **reviewer** | Teammate | 1-4: All | Persistent critic -- spawned in every phase team |
| **architect** | Teammate | 2: Design | System design, diagrams, tech justification |
| **security_developer** | Teammate | 2: Design | Threat modeling (STRIDE), auth, encryption |
| **devops_engineer** | Teammate | 2: Design, 4: Implement | Rollout planning, CI/CD, Docker, K8s, git flow |
| **project_manager** | Teammate | 3: Planning | Delivery plans, sprint breakdown, team coordination |
| **backend_developer** | Teammate | 4: Implement | Python, Java, Rust, C, C++, SQL |
| **frontend_developer** | Teammate | 4: Implement | TypeScript, React, Next.js |
| **mobile_app_developer** | Teammate | 4: Implement | Swift, Kotlin, Flutter, React Native |
| **qa_automation_engineer** | Teammate | 4: Implement | Test frameworks, automation, CI integration |
| **developer** | Teammate | 4: Implement | General-purpose, used when no specialist fits |

#### Phase-Based Team Compositions

Spawn a new team per phase. Each team is small (3-5 teammates) to keep coordination
manageable and token cost reasonable. Clean up the previous phase's team before
starting the next.

**Phase 1 -- Research & Spec:**

```text
Create an agent team for research. Spawn 3 teammates:
- business_analyst: research the problem space, produce functional spec with 3 tiers
- researcher: evaluate technical options, frameworks, and prior art
- reviewer: challenge all findings, verify sources, push back on assumptions
Have them share findings and challenge each other. Require plan approval.
```

**Phase 2 -- Architecture & Security:**

```text
Create an agent team for architecture design. Spawn 4 teammates:
- architect: create 3 design tiers with diagrams from the approved spec
- security_developer: threat model (STRIDE) and harden each tier
- devops_engineer: plan rollout strategy, CI/CD, infrastructure
- reviewer: challenge all design decisions
Architect and security_developer should iterate until the design is secure.
Require plan approval before any changes.
```

**Phase 3 -- Delivery Planning:**

```text
Create an agent team for delivery planning. Spawn 2 teammates:
- project_manager: create 3-tier delivery plan with milestones and sprint breakdown
- reviewer: challenge the plan, verify resource estimates, check for gaps
PM should document all external sources: repos, datasets (with sizes), APIs.
```

**Phase 4 -- Implementation (example: full-stack web app):**

```text
Create an agent team for sprint implementation. Spawn 5 teammates:
- backend_developer: implement server-side code, APIs, database schemas
- frontend_developer: implement UI components and pages
- qa_automation_engineer: write test briefs before devs start, then automate tests
- devops_engineer: set up git flow, Dockerfiles, docker-compose, Makefile, CI/CD
- reviewer: review all code for types, tests, docstrings, security
Each teammate owns separate files to avoid conflicts. Use sonnet for specialists.
```

#### Quality Gates with Hooks

The `TaskCompleted` and `TeammateIdle` hooks are configured in `.claude/settings.json`
and enforce zbik-agents coding standards automatically:

- **`TaskCompleted`** -- fires when any teammate marks a task complete. Reminds them to
  verify: strong typing, unit tests, docstrings (Args/Returns/Raises/Example),
  specific exception handling, no hardcoded secrets or magic numbers.

- **`TeammateIdle`** -- fires when a teammate is about to go idle. Reminds them to
  confirm: all tests pass, all functions typed and documented, no unaddressed
  TODO/FIXME, code reviewed against `_base.md` constitutional guardrails.

If the hook exits with code 2, the task is **not** marked complete and the teammate
is sent back to fix the issues. This enforces the quality gates without human
intervention during sprints.

#### Display Modes

| Mode | Setting | Best for |
|---|---|---|
| In-process (default) | `"teammateMode": "in-process"` | Any terminal; use Shift+Down to cycle teammates |
| Split panes | `"teammateMode": "tmux"` | tmux or iTerm2; see all teammates simultaneously |

To use split panes, add to `.claude/settings.json`:

```json
{
  "teammateMode": "tmux"
}
```

Or pass as a flag for a single session:

```bash
claude --teammate-mode tmux
```

#### Agent Teams vs Subagents: When to Use Which

| Scenario | Use |
|---|---|
| Sequential gated workflow (Phase 1 -> 2 -> 3 -> 4) | Subagents |
| BA writes spec, reviewer challenges it, you approve | Subagents |
| 3 reviewers examining a PR from different angles simultaneously | Agent Teams |
| Backend + frontend + QA implementing a feature in parallel | Agent Teams |
| Debugging with competing hypotheses | Agent Teams |
| Quick one-off research or code review | Subagents |

#### Limitations (as of v2.1.32)

- Experimental: enable with `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` flag
- No session resumption for in-process teammates
- One team per session; clean up before starting a new team
- No nested teams (teammates cannot spawn their own teams)
- All teammates inherit the lead's permission mode at spawn time
- Split panes require tmux or iTerm2 (not VS Code terminal, Windows Terminal, or Ghostty)
- Task status can lag; monitor and nudge if tasks appear stuck

---

### 3. CLAUDE.md (project-wide rules)

Embed shared coding standards from `_base.md` directly in your project's CLAUDE.md.

**`CLAUDE.md`** (project root):
```markdown
# Team Standards

@zbik-agents/_base.md

# Project-Specific

- This is a Python 3.12 project
- Use PostgreSQL for all database work
- Run `pytest` before committing
- Use `.venv` virtual environment (create with `python3 -m venv .venv`)
```

The `@zbik-agents/_base.md` import pulls in the constitutional guardrails (typing, testing, docstrings) for every session.

---

### 4. CLI Flags (one-off sessions)

#### Append Agent Context to Default Prompt

```bash
# Start a session with backend_developer context appended
claude --append-system-prompt-file zbik-agents/specialists/backend_developer.md

# Start a reviewer session
claude --append-system-prompt-file zbik-agents/roles/reviewer.md

# Combine base + specialist
claude --append-system-prompt-file zbik-agents/_base.md \
       --append-system-prompt-file zbik-agents/specialists/backend_developer.md
```

#### Non-Interactive (Scripting/CI)

```bash
# Run a one-shot review
claude -p "Review the code in src/" \
  --append-system-prompt-file zbik-agents/roles/reviewer.md

# Run backend implementation
claude -p "Implement the user service in Python" \
  --append-system-prompt-file zbik-agents/specialists/backend_developer.md
```

---

### 5. Hooks (context injection)

Use hooks to automatically inject agent context at specific lifecycle points.

#### Project Settings (`.claude/settings.json`)

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "cat zbik-agents/_base.md"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "echo 'Reminder: Ensure this file has type annotations, docstrings with examples, and corresponding unit tests.'"
          }
        ]
      }
    ]
  }
}
```

#### Subagent-Specific Hooks

```json
{
  "hooks": {
    "SubagentStart": [
      {
        "matcher": "backend_developer",
        "hooks": [
          {
            "type": "command",
            "command": "echo 'Python: always use .venv. Java: always use Maven with pinned versions. Rust: no .unwrap() in production.'"
          }
        ]
      }
    ]
  }
}
```

---

### 6. Agent SDK (programmatic)

Use the Claude Agent SDK for custom orchestration.

#### Python

```python
import asyncio
from pathlib import Path
from claude_agent_sdk import query, ClaudeAgentOptions, AgentDefinition

def load_agent(base_path: Path, *files: str) -> str:
    """Load and concatenate agent definition files."""
    parts: list[str] = []
    for f in files:
        parts.append((base_path / f).read_text())
    return "\n\n".join(parts)

AGENTS_DIR = Path("zbik-agents")

async def main() -> None:
    async for message in query(
        prompt="Build a REST API for user management with tests",
        options=ClaudeAgentOptions(
            allowed_tools=["Read", "Edit", "Write", "Bash", "Grep", "Glob", "Agent"],
            agents={
                "project_manager": AgentDefinition(
                    description="Coordinates work, decomposes tasks, tracks progress.",
                    prompt=load_agent(AGENTS_DIR, "_base.md", "roles/project_manager.md"),
                    tools=["Read", "Grep", "Glob", "Bash"],
                    model="opus",
                ),
                "backend_developer": AgentDefinition(
                    description="Backend development in Python, Java, Rust, C, C++, SQL.",
                    prompt=load_agent(AGENTS_DIR, "_base.md", "specialists/backend_developer.md"),
                    tools=["Read", "Edit", "Write", "Bash", "Grep", "Glob"],
                    model="opus",
                ),
                "reviewer": AgentDefinition(
                    description="Critical code review -- types, tests, docs, security.",
                    prompt=load_agent(AGENTS_DIR, "_base.md", "roles/reviewer.md"),
                    tools=["Read", "Grep", "Glob", "Bash"],
                    model="opus",
                ),
            },
        ),
    ):
        if hasattr(message, "result"):
            print(message.result)

asyncio.run(main())
```

#### TypeScript

```typescript
import { query, type AgentDefinition } from "claude-agent-sdk";
import { readFileSync } from "fs";
import { join } from "path";

function loadAgent(...files: string[]): string {
  const base = "zbik-agents";
  return files.map((f) => readFileSync(join(base, f), "utf-8")).join("\n\n");
}

const agents: Record<string, AgentDefinition> = {
  "backend_developer": {
    description: "Backend development in Python, Java, Rust, C, C++, SQL.",
    prompt: loadAgent("_base.md", "specialists/backend_developer.md"),
    tools: ["Read", "Edit", "Write", "Bash", "Grep", "Glob"],
    model: "opus",
  },
  reviewer: {
    description: "Critical code review -- types, tests, docs, security.",
    prompt: loadAgent("_base.md", "roles/reviewer.md"),
    tools: ["Read", "Grep", "Glob", "Bash"],
    model: "opus",
  },
};

for await (const message of query({
  prompt: "Build a REST API for user management with tests",
  options: { allowedTools: ["Read", "Edit", "Write", "Bash", "Agent"], agents },
})) {
  if ("result" in message) console.log(message.result);
}
```

---

## Model Configuration

All agents use **Claude Opus 4.6** (`opus`) by default.

The model is set in each `.claude/agents/*.md` subagent file via YAML frontmatter:

```yaml
---
name: backend_developer
model: opus          # <-- change model here
---
```

### Available Models

| `model:` value | Model ID | Notes |
|---|---|---|
| `opus` | `claude-opus-4-6` | Most capable (default for all agents) |
| `sonnet` | `claude-sonnet-4-6` | Faster, cheaper |
| `haiku` | `claude-haiku-4-5-20251001` | Fastest, cheapest |

### Change All Agents at Once

```bash
# Switch all agents to sonnet
sed -i '' 's/model: opus/model: sonnet/g' .claude/agents/*.md

# Switch back to opus
sed -i '' 's/model: sonnet/model: opus/g' .claude/agents/*.md
```

### Mix Models (e.g., opus for roles, sonnet for specialists)

Edit individual files. There is no central config -- the model is set per-agent
in each `.claude/agents/*.md` frontmatter.

---

## Orchestration Flow

When using the full squad, this is the conversation flow with human review gates:

```
+--------------------------------------------------------------------+
|                                                                    |
|  1. HUMAN + BA ------> [HUMAN GATE 1: approve spec]               |
|                   |                                                |
|  2. Architect + Security Dev + DevOps (rollout)                    |
|                ------> [HUMAN GATE 2: approve design]              |
|                   |                                                |
|  3. PM (plan) ------> [HUMAN GATE 3: approve plan + sources]      |
|                   |                                                |
|  4. Specialists + QA --> [HUMAN GATE 4: milestone review]          |
|                                                                    |
|     REVIEWER challenges ALL agents at EVERY step (independently)   |
+--------------------------------------------------------------------+
```

| Phase | Agent | Input | Output |
|---|---|---|---|
| 1. Initiate | business_analyst | Problem statement | Functional spec + 3 solution tiers + cost |
| 2. Design | architect + security_developer + devops_engineer | Approved spec | Architecture + security + rollout plan |
| 3. Plan | project_manager | Approved design | 3-tier delivery plan + sprint breakdown |
| 4. Implement | specialists + qa | Sprint tasks | Code + tests + CI/CD |
| All phases | reviewer | Each phase output | Challenges, approvals, blocking findings |

---

## Quick Reference

| Scenario | Method | Persistence |
|---|---|---|
| Team project, shared via git | Git submodule + subagents in `.claude/agents/` | Persistent, shared |
| Personal project | Copy or symlink + subagents | Persistent, private |
| All projects globally | Absolute path in `~/.claude/CLAUDE.md` | Persistent, global |
| Project-wide coding standards | CLAUDE.md with `@import` | Persistent, shared |
| One-off specialist session | `--append-system-prompt-file` | Per-invocation |
| CI/CD pipeline | CLI flags or Agent SDK | Per-invocation |
| Custom orchestration app | Agent SDK | Programmatic |
| Auto-inject on every session | Hooks in `.claude/settings.json` | Persistent |

---

## File Structure

```
your-project/
├── CLAUDE.md                              # @zbik-agents/CLAUDE.md import
├── .claude/
│   ├── settings.json                      # Hooks, permissions, tool config
│   └── agents/                            # Subagent definitions (shared via git)
│       ├── business_analyst.md            # Initiates all projects (L1)
│       ├── architect.md                   # System design + diagrams (L1)
│       ├── project_manager.md             # Sprint delivery planning (L1)
│       ├── reviewer.md                    # Persistent critic at every phase (L1)
│       ├── researcher.md                  # On-demand tech research (L1)
│       ├── developer.md                   # Base developer (L1)
│       ├── backend_developer.md           # Python/Java/Rust/C/C++/SQL (L2)
│       ├── frontend_developer.md          # TypeScript/React/Next.js (L2)
│       ├── mobile_app_developer.md        # Swift/Kotlin/Flutter (L2)
│       ├── devops_engineer.md             # CI/CD/Docker/K8s/Helm/git flow (L2)
│       ├── qa_automation_engineer.md      # Test frameworks + automation (L2)
│       └── security_developer.md          # Auth/encryption/threat modeling (L2)
└── zbik-agents/                        # Git submodule, copy, or symlink
    ├── CLAUDE.md                          # Runtime: workflow + standards (AI reads this)
    ├── _base.md                           # L0 Constitutional guardrails
    ├── roles/                             # L1 Role Archetypes
    │   ├── business_analyst.md
    │   ├── architect.md
    │   ├── project_manager.md
    │   ├── reviewer.md
    │   ├── researcher.md
    │   └── developer.md
    └── specialists/                       # L2 Technology Specializations
        ├── backend_developer.md
        ├── frontend_developer.md
        ├── mobile_app_developer.md
        ├── devops_engineer.md
        ├── qa_automation_engineer.md
        └── security_developer.md
```
