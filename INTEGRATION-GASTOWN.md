# Gastown Integration Guide

How to use `zbik-agents` with [Gastown](https://github.com/steveyegge/gastown).

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [Create a New Project (Step-by-Step)](#create-a-new-project-step-by-step)
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

Gastown requires a GitHub (or other remote) URL for every rig. Local-only repos
without a remote are not supported -- `gt crew add` and `gt sling` need a URL to
clone from. If you want local-only development without GitHub, use Claude Code
instead (see `INTEGRATION-CLAUDE-CODE.md`).

### Step 1: Create your town

The path is configurable -- `~/gt/` is the default but you can use any location.
The `--git` flag runs `gt git-init` automatically:

```bash
gt install ~/projects/gt --git
cd ~/projects/gt
```

If you used `--shell` instead, run `gt git-init` separately:

```bash
gt install ~/projects/gt --shell
cd ~/projects/gt
gt git-init
```

### Step 2: Add zbik-agents to the town

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

### Step 3: Install the agent wrapper

The Claude CLI has `--append-system-prompt` (inline text) but not
`--append-system-prompt-file` (read from file). The `bin/claude-agent` wrapper
translates `--append-system-prompt-file <path>` into
`--append-system-prompt "$(cat <path>)"` so agents.json can reference role files
by path.

```bash
mkdir -p bin
cat > bin/claude-agent << 'WRAPPER'
#!/bin/bash
# Wrapper: translates --append-system-prompt-file <path> into
# --append-system-prompt "$(cat <path>)" so Gastown agents.json works
# with the real Claude CLI (which only has --append-system-prompt).

# Resolve file paths relative to GT_ROOT if set
resolve_path() {
  local p="$1"
  if [[ "$p" != /* && -n "$GT_ROOT" && ! -f "$p" && -f "$GT_ROOT/$p" ]]; then
    echo "$GT_ROOT/$p"
  else
    echo "$p"
  fi
}

args=()
while [[ $# -gt 0 ]]; do
  case "$1" in
    --append-system-prompt-file)
      shift
      local_path="$(resolve_path "$1")"
      if [[ -f "$local_path" ]]; then
        args+=(--append-system-prompt "$(cat "$local_path")")
      else
        echo "claude-agent: file not found: $1 (resolved: $local_path)" >&2
        exit 1
      fi
      ;;
    *)
      args+=("$1")
      ;;
  esac
  shift
done

exec claude "${args[@]}"
WRAPPER
chmod +x bin/claude-agent
git add bin/ && git commit -m "Add claude-agent wrapper for system prompt files"
```

### Step 4: Register agents and map roles

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

### Updating zbik-agents

After changes are pushed to the `zbik-agents` repo, update the submodule in your town:

```bash
cd ~/projects/gt
git submodule update --remote zbik-agents
git add zbik-agents
git commit -m "Update zbik-agents to latest"
git push   # if backed up to GitHub
```

When teammates pull the town and need the updated submodule:

```bash
git pull
git submodule update --init --recursive
```

---

### Step 5: (Optional) Back up your town to GitHub

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

### Step 6: Create your project repo on GitHub

```bash
# For a brand new project
gh repo create <your-org>/my-project --private --clone=false
```

Skip this if you already have a repo on GitHub.

### Step 7: Add the project as a rig

```bash
cd ~/projects/gt
gt rig add my-project git@github.com:<your-org>/my-project.git
gt rig list   # verify
```

### Step 8: (Optional) Set rig-level overrides

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

### Step 9: Start the team

There are two ways to kick off a project — **direct agent sessions** or **orchestrated coordination**:

#### Option A: Direct Agent Session (`gt crew`)

```bash
gt crew add business_analyst --rig my-project
gt start crew business_analyst --rig my-project --agent business_analyst
```

The crew worker runs in the background. To interact with it, attach to its tmux session:

```bash
gt crew at business_analyst --rig my-project
```

This opens the Claude session where you type your requests. Use `Ctrl+B D` to detach from tmux and leave the session running in the background.

Starts a **single, persistent agent session**. You interact with that one agent directly and manually hand off between agents as each phase completes (e.g., when BA finishes the spec, you start the Architect yourself). Best for focused, phase-by-phase work with full control over the workflow.

#### Option B: Orchestrated Coordination (`gt mayor`)

```bash
gt mayor attach
```

Starts the **Project Manager (Mayor)** who acts as the global coordinator. The Mayor decomposes tasks, delegates to the right agents automatically, manages the full workflow (BA → Architect → Security → PM planning → Specialists), tracks progress across milestones, and enforces the 4 human review gates. Best for end-to-end project execution where you want the PM to drive.

| | `gt crew` (direct) | `gt mayor attach` (orchestrated) |
|---|---|---|
| **Control** | You drive | Mayor drives |
| **Scope** | One agent at a time | Full team coordination |
| **Handoffs** | Manual | Automatic |
| **Best for** | Focused work on a single phase | End-to-end project execution |

The flow once started:

```
You: I want to build [describe your project]

-> BA researches and produces functional spec
-> Reviewer (witness) challenges BA automatically
-> You review spec -> "approved"

# Start Architect
gt start crew architect --rig my-project --agent architect

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

### Step 10: Verify health

```bash
gt doctor
gt feed
gt agents
```

---

## Reference

Configuration files and settings referenced by the steps above.

### agents.json (agent presets)

Paste into `~/projects/gt/settings/agents.json` (or wherever your `GT_ROOT/settings/` is).

> **Note:** Gastown runs agent commands from the rig directory, not `GT_ROOT`.
> The `bash -c` wrapper resolves `bin/claude-agent` via `$GT_ROOT` so the
> wrapper is found regardless of the working directory. The `exec` ensures the
> final process is `claude`, so `process_names` detection still works.

```json
{
  "version": 1,
  "agents": {
    "researcher": {
      "name": "researcher",
      "command": "bash",
      "args": ["-c", "exec \"$GT_ROOT/bin/claude-agent\" \"$@\"", "--", "--model", "claude-opus-4-6", "--append-system-prompt-file", "zbik-agents/roles/researcher.md"],
      "process_names": ["claude", "node"],
      "session_id_env": "CLAUDE_SESSION_ID",
      "resume_flag": "--resume",
      "resume_style": "flag",
      "prompt_mode": "arg",
      "ready_delay_ms": 3000
    },
    "developer": {
      "name": "developer",
      "command": "bash",
      "args": ["-c", "exec \"$GT_ROOT/bin/claude-agent\" \"$@\"", "--", "--model", "claude-opus-4-6", "--append-system-prompt-file", "zbik-agents/roles/developer.md"],
      "process_names": ["claude", "node"],
      "session_id_env": "CLAUDE_SESSION_ID",
      "resume_flag": "--resume",
      "resume_style": "flag",
      "prompt_mode": "arg",
      "ready_delay_ms": 3000
    },
    "project_manager": {
      "name": "project_manager",
      "command": "bash",
      "args": ["-c", "exec \"$GT_ROOT/bin/claude-agent\" \"$@\"", "--", "--model", "claude-opus-4-6", "--append-system-prompt-file", "zbik-agents/roles/project_manager.md"],
      "process_names": ["claude", "node"],
      "session_id_env": "CLAUDE_SESSION_ID",
      "resume_flag": "--resume",
      "resume_style": "flag",
      "prompt_mode": "arg",
      "ready_delay_ms": 3000
    },
    "reviewer": {
      "name": "reviewer",
      "command": "bash",
      "args": ["-c", "exec \"$GT_ROOT/bin/claude-agent\" \"$@\"", "--", "--model", "claude-opus-4-6", "--append-system-prompt-file", "zbik-agents/roles/reviewer.md"],
      "process_names": ["claude", "node"],
      "session_id_env": "CLAUDE_SESSION_ID",
      "resume_flag": "--resume",
      "resume_style": "flag",
      "prompt_mode": "arg",
      "ready_delay_ms": 3000
    },
    "business_analyst": {
      "name": "business_analyst",
      "command": "bash",
      "args": ["-c", "exec \"$GT_ROOT/bin/claude-agent\" \"$@\"", "--", "--model", "claude-opus-4-6", "--append-system-prompt-file", "zbik-agents/roles/business_analyst.md"],
      "process_names": ["claude", "node"],
      "session_id_env": "CLAUDE_SESSION_ID",
      "resume_flag": "--resume",
      "resume_style": "flag",
      "prompt_mode": "arg",
      "ready_delay_ms": 3000
    },
    "architect": {
      "name": "architect",
      "command": "bash",
      "args": ["-c", "exec \"$GT_ROOT/bin/claude-agent\" \"$@\"", "--", "--model", "claude-opus-4-6", "--append-system-prompt-file", "zbik-agents/roles/architect.md"],
      "process_names": ["claude", "node"],
      "session_id_env": "CLAUDE_SESSION_ID",
      "resume_flag": "--resume",
      "resume_style": "flag",
      "prompt_mode": "arg",
      "ready_delay_ms": 3000
    },
    "backend": {
      "name": "backend",
      "command": "bash",
      "args": ["-c", "exec \"$GT_ROOT/bin/claude-agent\" \"$@\"", "--", "--model", "claude-opus-4-6", "--append-system-prompt-file", "zbik-agents/specialists/backend_developer.md"],
      "process_names": ["claude", "node"],
      "session_id_env": "CLAUDE_SESSION_ID",
      "resume_flag": "--resume",
      "resume_style": "flag",
      "prompt_mode": "arg",
      "ready_delay_ms": 3000
    },
    "frontend": {
      "name": "frontend",
      "command": "bash",
      "args": ["-c", "exec \"$GT_ROOT/bin/claude-agent\" \"$@\"", "--", "--model", "claude-opus-4-6", "--append-system-prompt-file", "zbik-agents/specialists/frontend_developer.md"],
      "process_names": ["claude", "node"],
      "session_id_env": "CLAUDE_SESSION_ID",
      "resume_flag": "--resume",
      "resume_style": "flag",
      "prompt_mode": "arg",
      "ready_delay_ms": 3000
    },
    "mobile": {
      "name": "mobile",
      "command": "bash",
      "args": ["-c", "exec \"$GT_ROOT/bin/claude-agent\" \"$@\"", "--", "--model", "claude-opus-4-6", "--append-system-prompt-file", "zbik-agents/specialists/mobile_app_developer.md"],
      "process_names": ["claude", "node"],
      "session_id_env": "CLAUDE_SESSION_ID",
      "resume_flag": "--resume",
      "resume_style": "flag",
      "prompt_mode": "arg",
      "ready_delay_ms": 3000
    },
    "devops": {
      "name": "devops",
      "command": "bash",
      "args": ["-c", "exec \"$GT_ROOT/bin/claude-agent\" \"$@\"", "--", "--model", "claude-opus-4-6", "--append-system-prompt-file", "zbik-agents/specialists/devops_engineer.md"],
      "process_names": ["claude", "node"],
      "session_id_env": "CLAUDE_SESSION_ID",
      "resume_flag": "--resume",
      "resume_style": "flag",
      "prompt_mode": "arg",
      "ready_delay_ms": 3000
    },
    "qa": {
      "name": "qa",
      "command": "bash",
      "args": ["-c", "exec \"$GT_ROOT/bin/claude-agent\" \"$@\"", "--", "--model", "claude-opus-4-6", "--append-system-prompt-file", "zbik-agents/specialists/qa_automation_engineer.md"],
      "process_names": ["claude", "node"],
      "session_id_env": "CLAUDE_SESSION_ID",
      "resume_flag": "--resume",
      "resume_style": "flag",
      "prompt_mode": "arg",
      "ready_delay_ms": 3000
    },
    "security": {
      "name": "security",
      "command": "bash",
      "args": ["-c", "exec \"$GT_ROOT/bin/claude-agent\" \"$@\"", "--", "--model", "claude-opus-4-6", "--append-system-prompt-file", "zbik-agents/specialists/security_developer.md"],
      "process_names": ["claude", "node"],
      "session_id_env": "CLAUDE_SESSION_ID",
      "resume_flag": "--resume",
      "resume_style": "flag",
      "prompt_mode": "arg",
      "ready_delay_ms": 3000
    },
    "product_owner": {
      "name": "product_owner",
      "command": "bash",
      "args": ["-c", "exec \"$GT_ROOT/bin/claude-agent\" \"$@\"", "--", "--model", "claude-opus-4-6", "--append-system-prompt-file", "zbik-agents/roles/product_owner.md"],
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
    "mayor":    "product_owner",
    "deacon":   "project_manager",
    "witness":  "qa",
    "refinery": "reviewer",
    "polecat":  "developer",
    "crew":     "business_analyst",
    "boot":     "security",
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

The model is set in `agents.json` via the `--model` argument inside the args array:

```json
"args": ["-c", "exec \"$GT_ROOT/bin/claude-agent\" \"$@\"", "--", "--model", "claude-opus-4-6", "--append-system-prompt-file", "..."]
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

| Gastown Role | Purpose | Default Agent | Available via `--agent` | Why |
|---|---|---|---|---|
| mayor | Global coordinator | product_owner | project_manager | PO drives product vision, prioritizes backlog |
| deacon | Background supervisor | project_manager | — | PM monitors health, coordinates, escalates |
| witness | Per-rig lifecycle manager | qa | reviewer | QA validates quality gates before merge |
| refinery | Merge queue processor | reviewer | qa | Reviewer enforces code review standards |
| polecat | Transient workers | developer | backend, frontend, mobile, devops, qa, security | Specialists execute tasks (override per rig) |
| crew | Persistent workers | business_analyst | architect, researcher, developer | Long-running sessions for research & analysis |
| boot | Ephemeral health triage | security | reviewer | Security-first diagnostic assessment |
| dog | Infrastructure helpers | devops | — | CI/CD and infrastructure tasks |

### Assigning work

```bash
# Polecats (transient workers)
gt sling <bead-id> my-project --agent backend
gt sling <bead-id> my-project --agent frontend

# Crew (persistent workers)
gt crew add backend_dev --rig my-project
gt start crew backend_dev --rig my-project --agent backend
```
