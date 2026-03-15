# Gastown Integration Guide

How to use `zbik-agents` with [Gastown](https://github.com/steveyegge/gastown).

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [Create a New Project](#create-a-new-project)
  - [Version A: Local Only (no remote)](#version-a-local-only-no-remote-no-push)
  - [Version B: With GitHub Remote](#version-b-with-github-remote)
- [Reference](#reference)
  - [agents.json (agent presets)](#agentsjson-agent-presets)
  - [town-settings.json (role mapping)](#town-settingsjson-role-mapping)
  - [Rig-level overrides](#rig-level-overrides)
  - [Context injection via hooks](#context-injection-via-hooks)
  - [Environment variables](#environment-variables)
  - [Model configuration](#model-configuration)
  - [Monitoring](#monitoring)
  - [Role-to-agent mapping](#role-to-agent-mapping)

---

## Prerequisites

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

---

## Create a New Project

Two versions: local-only (everything on your machine) and with GitHub remote.

### Version A: Local Only (no remote, no push)

No GitHub account needed. Everything stays on your machine.

The town path is configurable -- `~/gt/` is the default but you can use any location.
The `--git` flag runs `gt git-init` automatically. If you used `--shell` instead,
run `gt git-init` separately after install.

```bash
# 1. Create your town
gt install ~/projects/gt --git             # or: gt install ~/gt --git
cd ~/projects/gt

# 2. Add zbik-agents to the town (shared by all rigs) -- pick one:

# 2A. Copy from local
cp -r /Users/<your-user>/projects/<your-project>/zbik-agents .
git add zbik-agents/ && git commit -m "Add zbik-agents"

# 2B. Or git submodule (even without pushing the town itself)
git submodule add git@github.com:m-zbik/zbik-agents.git zbik-agents
git commit -m "Add zbik-agents submodule"

# 3. Register agents and map roles
#    Paste agents.json content from the Reference section below:
vi settings/agents.json
#    Paste town-settings.json content from the Reference section below:
vi settings/town-settings.json
git add settings/ && git commit -m "Configure agent presets and role mappings"

# Verify agents are registered
gt config agent list

# 4. Create your project inside the town and register as a rig
mkdir -p ~/projects/gt/my-project && cd ~/projects/gt/my-project
git init && git commit --allow-empty -m "Initial commit"
gt rig add my-project --adopt --force
gt rig list   # verify

# 6. Start working
gt crew add business_analyst --rig my-project
gt start crew business_analyst --agent business_analyst

# Or start the Mayor to coordinate everything
gt mayor attach
```

Everything is local. No remote, no push. You can add a remote later:

```bash
# Back up your town
gh repo create <your-org>/gt-town --private --clone=false
cd ~/projects/gt
git remote add origin git@github.com:<your-org>/gt-town.git
git push -u origin main

# Back up your project
gh repo create <your-org>/my-project --private --clone=false
cd ~/projects/my-project
git remote add origin git@github.com:<your-org>/my-project.git
git push -u origin main
```

---

### Version B: With GitHub Remote

Full setup with git submodules, GitHub backup, and team sharing.

#### Step 1: Create your town

The path is configurable -- `~/gt/` is the default but you can use any location:

```bash
gt install ~/projects/gt --git
cd ~/projects/gt
```

The `--git` flag runs `gt git-init` automatically. If you used `--shell` instead:

```bash
gt install ~/projects/gt --shell
cd ~/projects/gt
gt git-init                                # initialize git tracking separately
```

#### Step 2: Add zbik-agents as a git submodule to the town

```bash
cd ~/projects/gt
git submodule add git@github.com:<your-org>/zbik-agents.git zbik-agents
git commit -m "Add zbik-agents submodule"
```

This makes `zbik-agents/` available at the town root, shared by all rigs.

If you don't have zbik-agents in a remote repo yet, copy instead:

```bash
cp -r /Users/<your-user>/projects/<your-project>/zbik-agents .
git add zbik-agents/ && git commit -m "Add zbik-agents"
```

#### Step 3: Register agents and map roles

Paste the JSON from the [Reference](#reference) section below:

```bash
vi settings/agents.json            # paste from agents.json section
vi settings/town-settings.json     # paste from town-settings.json section
git add settings/ && git commit -m "Configure agent presets and role mappings"
```

Verify:

```bash
gt config agent list
```

#### Step 4: (Optional) Back up your town to GitHub

```bash
cd ~/projects/gt
gh repo create <your-org>/gt-town --private --clone=false
git remote add origin git@github.com:<your-org>/gt-town.git
git push -u origin main
```

On another machine:

```bash
git clone --recurse-submodules git@github.com:<your-org>/gt-town.git ~/projects/gt
```

#### Step 5: Create your project repo on GitHub

```bash
# For a brand new project
gh repo create <your-org>/my-project --private --clone=false
```

Skip this if you already have a repo on GitHub.

#### Step 6: Add the project as a rig

```bash
cd ~/projects/gt
gt rig add my-project git@github.com:<your-org>/my-project.git
gt rig list   # verify
```

#### Step 7: (Optional) Set rig-level overrides

```bash
cat > ~/projects/gt/my-project/settings/rig-settings.json << 'EOF'
{
  "type": "rig-settings",
  "version": 1,
  "role_agents": {
    "polecat": "backend"
  }
}
EOF
```

#### Step 8: Start the team

```bash
# Start with Business Analyst (BA initiates all projects)
gt crew add business_analyst --rig my-project
gt start crew business_analyst --agent business_analyst

# Or start the Mayor to coordinate everything
gt mayor attach
```

The flow once started:

```
You: I want to build [describe your project]

-> BA researches and produces functional spec
-> Reviewer (witness) challenges BA automatically
-> You review spec -> "approved"

# Start Architect
gt start crew architect --agent architect

-> Architect designs + Security Dev + DevOps plan rollout
-> You review -> "approved"

# Mayor (PM) takes over delivery planning
-> PM plans delivery with milestones
-> You approve milestones, repos, data sources

# Polecats (specialists) execute sprints
gt sling <bead-id> my-project --agent backend
gt sling <bead-id> my-project --agent frontend

-> You review at milestones only
```

#### Step 9: Verify health

```bash
gt doctor
gt feed
gt agents
```

---

## Reference

Configuration files and settings referenced by the steps above.

### agents.json (agent presets)

Paste into `~/projects/gt/settings/agents.json` (or wherever your `GT_ROOT/settings/` is):

```json
{
  "version": 1,
  "agents": {
    "researcher": {
      "name": "researcher",
      "command": "claude",
      "args": ["--model", "claude-opus-4-6", "--append-system-prompt-file", "zbik-agents/roles/researcher.md"],
      "process_names": ["claude", "node"],
      "session_id_env": "CLAUDE_SESSION_ID",
      "resume_flag": "--resume",
      "resume_style": "flag",
      "prompt_mode": "arg",
      "ready_delay_ms": 3000
    },
    "developer": {
      "name": "developer",
      "command": "claude",
      "args": ["--model", "claude-opus-4-6", "--append-system-prompt-file", "zbik-agents/roles/developer.md"],
      "process_names": ["claude", "node"],
      "session_id_env": "CLAUDE_SESSION_ID",
      "resume_flag": "--resume",
      "resume_style": "flag",
      "prompt_mode": "arg",
      "ready_delay_ms": 3000
    },
    "project_manager": {
      "name": "project_manager",
      "command": "claude",
      "args": ["--model", "claude-opus-4-6", "--append-system-prompt-file", "zbik-agents/roles/project_manager.md"],
      "process_names": ["claude", "node"],
      "session_id_env": "CLAUDE_SESSION_ID",
      "resume_flag": "--resume",
      "resume_style": "flag",
      "prompt_mode": "arg",
      "ready_delay_ms": 3000
    },
    "reviewer": {
      "name": "reviewer",
      "command": "claude",
      "args": ["--model", "claude-opus-4-6", "--append-system-prompt-file", "zbik-agents/roles/reviewer.md"],
      "process_names": ["claude", "node"],
      "session_id_env": "CLAUDE_SESSION_ID",
      "resume_flag": "--resume",
      "resume_style": "flag",
      "prompt_mode": "arg",
      "ready_delay_ms": 3000
    },
    "business_analyst": {
      "name": "business_analyst",
      "command": "claude",
      "args": ["--model", "claude-opus-4-6", "--append-system-prompt-file", "zbik-agents/roles/business_analyst.md"],
      "process_names": ["claude", "node"],
      "session_id_env": "CLAUDE_SESSION_ID",
      "resume_flag": "--resume",
      "resume_style": "flag",
      "prompt_mode": "arg",
      "ready_delay_ms": 3000
    },
    "architect": {
      "name": "architect",
      "command": "claude",
      "args": ["--model", "claude-opus-4-6", "--append-system-prompt-file", "zbik-agents/roles/architect.md"],
      "process_names": ["claude", "node"],
      "session_id_env": "CLAUDE_SESSION_ID",
      "resume_flag": "--resume",
      "resume_style": "flag",
      "prompt_mode": "arg",
      "ready_delay_ms": 3000
    },
    "backend": {
      "name": "backend",
      "command": "claude",
      "args": ["--model", "claude-opus-4-6", "--append-system-prompt-file", "zbik-agents/specialists/backend_developer.md"],
      "process_names": ["claude", "node"],
      "session_id_env": "CLAUDE_SESSION_ID",
      "resume_flag": "--resume",
      "resume_style": "flag",
      "prompt_mode": "arg",
      "ready_delay_ms": 3000
    },
    "frontend": {
      "name": "frontend",
      "command": "claude",
      "args": ["--model", "claude-opus-4-6", "--append-system-prompt-file", "zbik-agents/specialists/frontend_developer.md"],
      "process_names": ["claude", "node"],
      "session_id_env": "CLAUDE_SESSION_ID",
      "resume_flag": "--resume",
      "resume_style": "flag",
      "prompt_mode": "arg",
      "ready_delay_ms": 3000
    },
    "mobile": {
      "name": "mobile",
      "command": "claude",
      "args": ["--model", "claude-opus-4-6", "--append-system-prompt-file", "zbik-agents/specialists/mobile_app_developer.md"],
      "process_names": ["claude", "node"],
      "session_id_env": "CLAUDE_SESSION_ID",
      "resume_flag": "--resume",
      "resume_style": "flag",
      "prompt_mode": "arg",
      "ready_delay_ms": 3000
    },
    "devops": {
      "name": "devops",
      "command": "claude",
      "args": ["--model", "claude-opus-4-6", "--append-system-prompt-file", "zbik-agents/specialists/devops_engineer.md"],
      "process_names": ["claude", "node"],
      "session_id_env": "CLAUDE_SESSION_ID",
      "resume_flag": "--resume",
      "resume_style": "flag",
      "prompt_mode": "arg",
      "ready_delay_ms": 3000
    },
    "qa": {
      "name": "qa",
      "command": "claude",
      "args": ["--model", "claude-opus-4-6", "--append-system-prompt-file", "zbik-agents/specialists/qa_automation_engineer.md"],
      "process_names": ["claude", "node"],
      "session_id_env": "CLAUDE_SESSION_ID",
      "resume_flag": "--resume",
      "resume_style": "flag",
      "prompt_mode": "arg",
      "ready_delay_ms": 3000
    },
    "security": {
      "name": "security",
      "command": "claude",
      "args": ["--model", "claude-opus-4-6", "--append-system-prompt-file", "zbik-agents/specialists/security_developer.md"],
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

---

### town-settings.json (role mapping)

Paste into `~/projects/gt/settings/town-settings.json`:

```json
{
  "type": "town-settings",
  "version": 1,
  "default_agent": "developer",
  "role_agents": {
    "mayor":    "project_manager",
    "deacon":   "project_manager",
    "witness":  "reviewer",
    "refinery": "reviewer",
    "polecat":  "developer",
    "crew":     "developer",
    "boot":     "reviewer",
    "dog":      "devops"
  }
}
```

---

### Rig-level overrides

Override per-project in `<rig>/settings/rig-settings.json`. Only include fields you want to change:

```json
{
  "type": "rig-settings",
  "version": 1,
  "role_agents": {
    "polecat": "backend"
  }
}
```

Example overrides for different project types:

| Project Type | polecat | witness | crew | dog |
|---|---|---|---|---|
| Python backend | backend | reviewer | business_analyst, architect | devops |
| React frontend | frontend | reviewer | business_analyst, architect | devops |
| Mobile app | mobile | reviewer | business_analyst, architect | devops |
| Infrastructure | devops | reviewer | business_analyst, architect | devops |

---

### Context injection via hooks

Inject agent context via SessionStart hooks for deeper integration.

**Base hooks** (`~/.gt/hooks-base.json`):

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

**Role-specific overrides:**

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
gt hooks sync   # apply changes
```

---

### Environment variables

Gastown automatically sets these per agent session:

| Variable | Example | Description |
|---|---|---|
| `GT_ROLE` | `polecat` | Gastown role (mayor, witness, polecat, crew, etc.) |
| `GT_RIG` | `my-project` | Which rig the agent belongs to |
| `GT_ROOT` | `~/projects/gt` | Town root directory |
| `GT_POLECAT` | `worker-1` | Polecat name (polecat-only) |
| `BD_ACTOR` | `my-project/polecats/worker-1` | Beads identity for attribution |
| `GIT_AUTHOR_NAME` | `my-project/polecats/worker-1` | Git commit attribution |

---

### Model configuration

All agents use **Claude Opus 4.6** (`claude-opus-4-6`) by default.

The model is set in `agents.json` via the `--model` argument:

```json
"args": ["--model", "claude-opus-4-6", "--append-system-prompt-file", "..."]
```

| `--model` value | Model | Notes |
|---|---|---|
| `claude-opus-4-6` | Claude Opus 4.6 | Most capable (default) |
| `claude-sonnet-4-6` | Claude Sonnet 4.6 | Faster, cheaper |
| `claude-haiku-4-5-20251001` | Claude Haiku 4.5 | Fastest, cheapest |

Change all at once:

```bash
sed -i '' 's/claude-opus-4-6/claude-sonnet-4-6/g' ~/projects/gt/settings/agents.json
```

---

### Monitoring

```bash
gt feed                          # Real-time activity dashboard
gt agents                        # View active workers
gt peek backend              # Look at a specific agent's session
gt seance                        # Query previous sessions
gt doctor                        # Health check
gt deacon health-state           # Overall health
```

---

### Role-to-agent mapping

| Gastown Role | Purpose | zbik-agents Agent | Why |
|---|---|---|---|
| mayor | Global coordinator | project_manager | PM decomposes tasks, tracks progress |
| deacon | Background supervisor | project_manager | PM monitors health, escalates |
| witness | Per-rig lifecycle manager | reviewer | Reviewer verifies quality before merge |
| refinery | Merge queue processor | reviewer | Reviewer enforces quality gates |
| polecat | Transient workers | backend/frontend/mobile/devops/qa/security | Specialists execute tasks |
| crew | Persistent workers | business_analyst, architect, researcher | Long-running sessions |
| boot | Ephemeral health triage | reviewer | Quick diagnostic assessment |
| dog | Infrastructure helpers | devops | CI/CD and infrastructure tasks |

### Assigning work

```bash
# Polecats (transient workers)
gt sling <bead-id> my-project --agent backend
gt sling <bead-id> my-project --agent frontend

# Crew (persistent workers)
gt crew add backend_dev --rig my-project
gt start crew backend_dev --agent backend
```
