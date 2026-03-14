# Gastown Integration Guide

How to use `zbik-agents` with [Gastown](https://github.com/steveyegge/gastown).

---

## Table of Contents

- [Setup: Add zbik-agents to Your Town](#setup-add-zbik-agents-to-your-town)
  - [Method A: Simple Copy (quickest)](#method-a-simple-copy-quickest)
  - [Method B: Git Submodule (recommended for teams)](#method-b-git-submodule-recommended-for-teams)
  - [Method C: Symlink or Absolute Path (personal use)](#method-c-symlink-or-absolute-path-personal-use)
- [How to Create a New Project (Step-by-Step)](#how-to-create-a-new-project-step-by-step)
- [1. Register Agents (agents.json)](#1-register-agents-agentsjson)
- [2. Map Roles (town/rig settings)](#2-map-roles-townrig-settings)
- [3. Create a Rig](#3-create-a-rig)
- [4. Assign Work](#4-assign-work)
- [5. Context Injection via Hooks](#5-context-injection-via-hooks)
- [6. Environment Variables](#6-environment-variables)
- [7. Model Configuration](#7-model-configuration)
- [8. Monitoring Your Agents](#8-monitoring-your-agents)
- [9. Role-to-Agent Mapping Reference](#9-role-to-agent-mapping-reference)

---

## Setup: Add zbik-agents to Your Town

Gastown agents reference `zbik-agents/` files via `--append-system-prompt-file`
in their `args` array. The path resolves relative to the rig's working directory.
You need `zbik-agents/` accessible from each rig. Choose one method below.

### Method A: Simple Copy (quickest)

Copy `zbik-agents/` into your Gastown town root or into each rig.

**Town-level (shared by all rigs):**

```bash
cp -r /Users/<your-user>/projects/<your-project>/zbik-agents ~/gt/zbik-agents
```

Then in `agents.json`, use paths relative to the rig working dir. Since rigs
are subdirectories of `~/gt/`, the path `../zbik-agents/roles/reviewer.md`
goes up one level to town root. Alternatively, place a copy inside each rig.

**Rig-level (per-project):**

```bash
cp -r /Users/<your-user>/projects/<your-project>/zbik-agents ~/gt/<rig-name>/zbik-agents
```

Then agents.json paths use `zbik-agents/roles/reviewer.md` directly.

**Add CLAUDE.md to the rig:**

```bash
# In your rig's CLAUDE.md (or the repo's CLAUDE.md):
echo '@zbik-agents/CLAUDE.md' >> ~/gt/<rig-name>/mayor/rig/CLAUDE.md
```

**Pros:** Simple, no external dependencies.
**Cons:** Agents don't auto-update; must re-copy for changes.

---

### Method B: Git Submodule (recommended for teams)

Add `zbik-agents` as a submodule in your project repo. When Gastown clones
the repo as a rig, the submodule comes with it.

**Step 1:** In your project repo (before adding to Gastown):

```bash
cd your-project
git submodule add https://github.com/<your-org>/zbik-agents.git zbik-agents
git commit -m "Add zbik-agents submodule"
git push
```

**Step 2:** Add as a Gastown rig (submodules are cloned automatically):

```bash
gt rig add myproject https://github.com/<your-org>/your-project.git
```

**Step 3:** Add CLAUDE.md import in your repo:

```markdown
# In your repo's CLAUDE.md
@zbik-agents/CLAUDE.md
```

**Step 4:** Update agents to new version:

```bash
cd ~/gt/myproject/mayor/rig/zbik-agents
git pull origin main
cd ..
git add zbik-agents
git commit -m "Update zbik-agents to latest"
```

**Pros:** Versioned, portable, works across team members and rigs.
**Cons:** Requires git submodule init on fresh clones.

---

### Method C: Symlink or Absolute Path (personal use)

**Option C1: Symlink at town level**

```bash
ln -s /Users/<your-user>/projects/<your-project>/zbik-agents ~/gt/zbik-agents
```

All rigs can reference via `../zbik-agents/` paths.

**Option C2: Symlink per rig**

```bash
ln -s /Users/<your-user>/projects/<your-project>/zbik-agents ~/gt/<rig-name>/zbik-agents
```

Agents reference via `zbik-agents/` paths.

**Option C3: Absolute paths in agents.json**

Use absolute paths in the `args` array:

```json
"args": ["--model", "claude-opus-4-6", "--append-system-prompt-file", "/Users/<your-user>/projects/<your-project>/zbik-agents/roles/reviewer.md"]
```

**Pros:** Always up-to-date, no copies.
**Cons:** Not portable; absolute paths are machine-specific; symlinks won't work for other team members.

---

### How Gastown Resolves Agent Paths

```
~/gt/                                  <-- town root
├── settings/
│   ├── agents.json                    <-- agent presets (--append-system-prompt-file paths)
│   └── town-settings.json             <-- role-to-agent mapping
├── zbik-agents/                    <-- copy, submodule, or symlink (town-level)
│   ├── CLAUDE.md
│   ├── _base.md
│   ├── roles/
│   └── specialists/
└── myproject/                         <-- rig
    ├── settings/
    │   └── rig-settings.json          <-- rig-level overrides
    ├── mayor/rig/                     <-- the cloned repo
    │   ├── CLAUDE.md                  <-- "@zbik-agents/CLAUDE.md" or "@../../../zbik-agents/CLAUDE.md"
    │   └── zbik-agents/            <-- submodule (if repo has it)
    ├── polecats/                      <-- transient workers
    ├── crew/                          <-- persistent workers
    └── witness/                       <-- monitor
```

The `--append-system-prompt-file` path in agents.json resolves from the **rig's working directory** (where the agent session runs). Plan your paths accordingly.

---

## How to Create a New Project (Step-by-Step)

Complete walkthrough from empty Gastown town to running team.

### Prerequisites

Install Gastown (pick one):

```bash
# Homebrew (recommended)
brew install gastown

# npm
npm install -g @gastown/gt

# Go
go install github.com/steveyegge/gastown/cmd/gt@latest
```

Also required: Git 2.25+, Dolt 1.82.4+, beads (`bd`) 0.55.4+, tmux 3.0+, sqlite3.

### Step 1: Create your Gastown town (if not already done)

The path is configurable -- `~/gt/` is the default, but you can use any location:

```bash
gt install ~/gt --shell                    # default
gt install ~/projects/gt --shell           # custom location
gt install /opt/gastown --shell            # anywhere you want
```

The path you pass becomes `GT_ROOT`. All rigs, settings, and shared zbik-agents
live under it.

### Step 2: Add zbik-agents to your town (pick one)

```bash
# Option A: Copy to town root (shared by all rigs)
cp -r /Users/<your-user>/projects/<your-project>/zbik-agents ~/gt/zbik-agents

# Option B: Symlink to town root
ln -s /Users/<your-user>/projects/<your-project>/zbik-agents ~/gt/zbik-agents

# Option C: Git submodule in your project repo (see Method B in setup section above)
# Then zbik-agents/ will be inside the rig automatically
```

### Step 3: Register agents and map roles

Copy the agents.json and town-settings.json from sections 1 and 2 below into:

```bash
# Agents
vi ~/gt/settings/agents.json    # Paste from Section 1

# Role mappings
vi ~/gt/settings/town-settings.json  # Paste from Section 2
```

Verify:

```bash
gt config agent list
```

### Step 4: Create your project repo (if new)

```bash
mkdir my-new-project && cd my-new-project
git init

# Add zbik-agents to the repo as submodule (if using Method B)
git submodule add https://github.com/<your-org>/zbik-agents.git zbik-agents

# Add CLAUDE.md
cat > CLAUDE.md << 'EOF'
# My New Project

## Team
@zbik-agents/CLAUDE.md

## Project-Specific
- Describe your project tech stack here
EOF

git add . && git commit -m "Initial project setup with zbik-agents"
git remote add origin https://github.com/<your-org>/my-new-project.git
git push -u origin main
```

### Step 5: Add the project as a Gastown rig

```bash
gt rig add myproject https://github.com/<your-org>/my-new-project.git
gt rig list   # Verify
```

### Step 6: (Optional) Set rig-level overrides

```bash
cat > ~/gt/myproject/settings/rig-settings.json << 'EOF'
{
  "type": "rig-settings",
  "version": 1,
  "role_agents": {
    "polecat": "zbk-backend"
  }
}
EOF
```

### Step 7: Start the team and begin

```bash
# Start with Business Analyst (BA initiates all projects)
gt crew add business-analyst --rig myproject
gt start crew business-analyst --agent zbk-business-analyst
```

```
You: I want to build [describe your project]

-> BA researches and produces functional spec
-> Reviewer (witness) challenges BA automatically
-> You review spec in the crew session -> "approved"

# Start Architect
gt start crew architect --agent zbk-architect

-> Architect designs + Security Dev + DevOps plan rollout
-> You review -> "approved"

# Mayor (PM) takes over delivery planning
-> PM plans delivery with milestones
-> You approve milestones, repos, data sources

# Polecats (specialists) execute sprints
gt sling <bead-id> myproject --agent zbk-backend
gt sling <bead-id> myproject --agent zbk-frontend

-> You review at milestones only
```

### Step 8: Verify health

```bash
gt doctor
gt feed
gt agents
```

---

## 1. Register Agents (agents.json)

Add agent presets to `~/gt/settings/agents.json`:

```json
{
  "version": 1,
  "agents": {
    "zbk-researcher": {
      "name": "zbk-researcher",
      "command": "claude",
      "args": ["--model", "claude-opus-4-6", "--append-system-prompt-file", "zbik-agents/roles/researcher.md"],
      "process_names": ["claude", "node"],
      "session_id_env": "CLAUDE_SESSION_ID",
      "resume_flag": "--resume",
      "resume_style": "flag",
      "prompt_mode": "arg",
      "ready_delay_ms": 3000
    },
    "zbk-developer": {
      "name": "zbk-developer",
      "command": "claude",
      "args": ["--model", "claude-opus-4-6", "--append-system-prompt-file", "zbik-agents/roles/developer.md"],
      "process_names": ["claude", "node"],
      "session_id_env": "CLAUDE_SESSION_ID",
      "resume_flag": "--resume",
      "resume_style": "flag",
      "prompt_mode": "arg",
      "ready_delay_ms": 3000
    },
    "zbk-project-manager": {
      "name": "zbk-project-manager",
      "command": "claude",
      "args": ["--model", "claude-opus-4-6", "--append-system-prompt-file", "zbik-agents/roles/project-manager.md"],
      "process_names": ["claude", "node"],
      "session_id_env": "CLAUDE_SESSION_ID",
      "resume_flag": "--resume",
      "resume_style": "flag",
      "prompt_mode": "arg",
      "ready_delay_ms": 3000
    },
    "zbk-reviewer": {
      "name": "zbk-reviewer",
      "command": "claude",
      "args": ["--model", "claude-opus-4-6", "--append-system-prompt-file", "zbik-agents/roles/reviewer.md"],
      "process_names": ["claude", "node"],
      "session_id_env": "CLAUDE_SESSION_ID",
      "resume_flag": "--resume",
      "resume_style": "flag",
      "prompt_mode": "arg",
      "ready_delay_ms": 3000
    },
    "zbk-business-analyst": {
      "name": "zbk-business-analyst",
      "command": "claude",
      "args": ["--model", "claude-opus-4-6", "--append-system-prompt-file", "zbik-agents/roles/business-analyst.md"],
      "process_names": ["claude", "node"],
      "session_id_env": "CLAUDE_SESSION_ID",
      "resume_flag": "--resume",
      "resume_style": "flag",
      "prompt_mode": "arg",
      "ready_delay_ms": 3000
    },
    "zbk-architect": {
      "name": "zbk-architect",
      "command": "claude",
      "args": ["--model", "claude-opus-4-6", "--append-system-prompt-file", "zbik-agents/roles/architect.md"],
      "process_names": ["claude", "node"],
      "session_id_env": "CLAUDE_SESSION_ID",
      "resume_flag": "--resume",
      "resume_style": "flag",
      "prompt_mode": "arg",
      "ready_delay_ms": 3000
    },
    "zbk-backend": {
      "name": "zbk-backend",
      "command": "claude",
      "args": ["--model", "claude-opus-4-6", "--append-system-prompt-file", "zbik-agents/specialists/backend-developer.md"],
      "process_names": ["claude", "node"],
      "session_id_env": "CLAUDE_SESSION_ID",
      "resume_flag": "--resume",
      "resume_style": "flag",
      "prompt_mode": "arg",
      "ready_delay_ms": 3000
    },
    "zbk-frontend": {
      "name": "zbk-frontend",
      "command": "claude",
      "args": ["--model", "claude-opus-4-6", "--append-system-prompt-file", "zbik-agents/specialists/frontend-developer.md"],
      "process_names": ["claude", "node"],
      "session_id_env": "CLAUDE_SESSION_ID",
      "resume_flag": "--resume",
      "resume_style": "flag",
      "prompt_mode": "arg",
      "ready_delay_ms": 3000
    },
    "zbk-mobile": {
      "name": "zbk-mobile",
      "command": "claude",
      "args": ["--model", "claude-opus-4-6", "--append-system-prompt-file", "zbik-agents/specialists/mobile-app-developer.md"],
      "process_names": ["claude", "node"],
      "session_id_env": "CLAUDE_SESSION_ID",
      "resume_flag": "--resume",
      "resume_style": "flag",
      "prompt_mode": "arg",
      "ready_delay_ms": 3000
    },
    "zbk-devops": {
      "name": "zbk-devops",
      "command": "claude",
      "args": ["--model", "claude-opus-4-6", "--append-system-prompt-file", "zbik-agents/specialists/devops-engineer.md"],
      "process_names": ["claude", "node"],
      "session_id_env": "CLAUDE_SESSION_ID",
      "resume_flag": "--resume",
      "resume_style": "flag",
      "prompt_mode": "arg",
      "ready_delay_ms": 3000
    },
    "zbk-qa": {
      "name": "zbk-qa",
      "command": "claude",
      "args": ["--model", "claude-opus-4-6", "--append-system-prompt-file", "zbik-agents/specialists/qa-automation-engineer.md"],
      "process_names": ["claude", "node"],
      "session_id_env": "CLAUDE_SESSION_ID",
      "resume_flag": "--resume",
      "resume_style": "flag",
      "prompt_mode": "arg",
      "ready_delay_ms": 3000
    },
    "zbk-security": {
      "name": "zbk-security",
      "command": "claude",
      "args": ["--model", "claude-opus-4-6", "--append-system-prompt-file", "zbik-agents/specialists/security-developer.md"],
      "process_names": ["claude", "node"],
      "session_id_env": "CLAUDE_SESSION_ID",
      "resume_flag": "--resume",
      "resume_style": "flag",
      "prompt_mode": "arg",
      "ready_delay_ms": 3000
    }
  }
}
```

Verify registration:

```bash
gt config agent list
```

---

## 2. Map Roles (town/rig settings)

### Town-Level Defaults (`~/gt/settings/town-settings.json`)

```json
{
  "type": "town-settings",
  "version": 1,
  "default_agent": "zbk-developer",
  "role_agents": {
    "mayor":    "zbk-project-manager",
    "deacon":   "zbk-project-manager",
    "witness":  "zbk-reviewer",
    "refinery": "zbk-reviewer",
    "polecat":  "zbk-developer",
    "crew":     "zbk-developer",
    "boot":     "zbk-reviewer",
    "dog":      "zbk-devops"
  }
}
```

### Rig-Level Overrides (`<rig>/settings/rig-settings.json`)

Override per-project. Only include fields you want to change:

```json
{
  "type": "rig-settings",
  "version": 1,
  "role_agents": {
    "polecat": "zbk-backend"
  }
}
```

Example overrides for different project types:

| Project Type | polecat | witness | crew | dog |
|---|---|---|---|---|
| Python backend | zbk-backend | zbk-reviewer | zbk-business-analyst, zbk-architect | zbk-devops |
| React frontend | zbk-frontend | zbk-reviewer | zbk-business-analyst, zbk-architect | zbk-devops |
| Mobile app | zbk-mobile | zbk-reviewer | zbk-business-analyst, zbk-architect | zbk-devops |
| Infrastructure | zbk-devops | zbk-reviewer | zbk-business-analyst, zbk-architect | zbk-devops |
| Security-focused | zbk-security | zbk-reviewer | zbk-business-analyst, zbk-architect | zbk-devops |
| QA/Testing | zbk-qa | zbk-reviewer | zbk-business-analyst | zbk-devops |

---

## 3. Create a Rig

```bash
# Add your project as a rig
gt rig add myproject https://github.com/org/myproject.git

# Verify
gt rig list
```

---

## 4. Assign Work

### Using Polecats (Transient Workers)

```bash
# Assign a bead to a polecat (uses rig's default polecat agent)
gt sling <bead-id> myproject

# Assign with a specific specialist
gt sling <bead-id> myproject --agent zbk-backend
gt sling <bead-id> myproject --agent zbk-frontend
gt sling <bead-id> myproject --agent zbk-mobile
gt sling <bead-id> myproject --agent zbk-qa
gt sling <bead-id> myproject --agent zbk-security

# Assign a formula-based workflow
gt sling <formula-name> --on <bead-id> myproject
```

### Using Crew (Persistent Workers)

```bash
# Create persistent workspaces
gt crew add backend-dev --rig myproject
gt crew add frontend-dev --rig myproject
gt crew add business-analyst --rig myproject
gt crew add architect --rig myproject

# Start sessions with specific agents
gt start crew backend-dev --agent zbk-backend
gt start crew frontend-dev --agent zbk-frontend
gt start crew business-analyst --agent zbk-business-analyst
gt start crew architect --agent zbk-architect
```

---

## 5. Context Injection via Hooks

For deeper integration, inject agent context via SessionStart hooks.

### Base Hooks (`~/.gt/hooks-base.json`)

```json
{
  "SessionStart": [
    {
      "matcher": "",
      "hooks": [
        {
          "type": "command",
          "command": "gt prime --hook"
        },
        {
          "type": "command",
          "command": "cat zbik-agents/_base.md"
        }
      ]
    }
  ]
}
```

### Role-Specific Overrides

```bash
# Polecat override -- inject developer base
gt hooks override polecat
```

```json
{
  "SessionStart": [
    {
      "matcher": "",
      "hooks": [
        {
          "type": "command",
          "command": "cat zbik-agents/roles/developer.md"
        }
      ]
    }
  ]
}
```

```bash
# Witness override -- inject reviewer
gt hooks override witness
```

```json
{
  "SessionStart": [
    {
      "matcher": "",
      "hooks": [
        {
          "type": "command",
          "command": "cat zbik-agents/roles/reviewer.md"
        }
      ]
    }
  ]
}
```

Apply changes:

```bash
gt hooks sync
```

---

## 6. Environment Variables

Gastown automatically sets these per agent session:

| Variable | Example | Description |
|---|---|---|
| `GT_ROLE` | `polecat` | Gastown role (mayor, witness, polecat, crew, etc.) |
| `GT_RIG` | `myproject` | Which rig the agent belongs to |
| `GT_ROOT` | `~/gt` | Town root directory |
| `GT_POLECAT` | `worker-1` | Polecat name (polecat-only) |
| `BD_ACTOR` | `myproject/polecats/worker-1` | Beads identity for attribution |
| `GIT_AUTHOR_NAME` | `myproject/polecats/worker-1` | Git commit attribution |

---

## 7. Model Configuration

All agents use **Claude Opus 4.6** (`claude-opus-4-6`) by default.

The model is set in `~/gt/settings/agents.json` via the `--model` argument in each agent's `args` array:

```json
"zbk-backend": {
  "args": ["--model", "claude-opus-4-6", "--append-system-prompt-file", "zbik-agents/specialists/backend-developer.md"],
  ...
}
```

### Available Models

| `--model` value | Model | Notes |
|---|---|---|
| `claude-opus-4-6` | Claude Opus 4.6 | Most capable (default for all agents) |
| `claude-sonnet-4-6` | Claude Sonnet 4.6 | Faster, cheaper |
| `claude-haiku-4-5-20251001` | Claude Haiku 4.5 | Fastest, cheapest |

### Change All Agents at Once

```bash
# Switch all agents to sonnet
sed -i '' 's/claude-opus-4-6/claude-sonnet-4-6/g' ~/gt/settings/agents.json

# Switch back to opus
sed -i '' 's/claude-sonnet-4-6/claude-opus-4-6/g' ~/gt/settings/agents.json
```

### Mix Models

Edit individual entries in `agents.json`. There is no central config -- the model
is set per-agent in the `args` array.

---

## 8. Monitoring Your Agents

```bash
gt feed                          # Real-time activity dashboard
gt agents                        # View active workers
gt peek zbk-backend              # Look at a specific agent's session
gt seance                        # Query previous sessions

# Health checks
gt deacon health-check zbk-backend
gt deacon health-state
```

---

## 9. Role-to-Agent Mapping Reference

| Gastown Role | Purpose | zbik-agents Agent | Why |
|---|---|---|---|
| mayor | Global coordinator | zbk-project-manager | PM decomposes tasks, tracks progress |
| deacon | Background supervisor | zbk-project-manager | PM monitors health, escalates |
| witness | Per-rig lifecycle manager | zbk-reviewer | Reviewer verifies quality before merge |
| refinery | Merge queue processor | zbk-reviewer | Reviewer enforces quality gates |
| polecat | Transient workers | zbk-backend/frontend/mobile/devops/qa/security | Specialists execute implementation tasks |
| crew | Persistent workers | zbk-business-analyst, zbk-architect, zbk-researcher | Long-running research, design, and analysis sessions |
| boot | Ephemeral health triage | zbk-reviewer | Quick diagnostic assessment |
| dog | Infrastructure helpers | zbk-devops | CI/CD and infrastructure tasks |
