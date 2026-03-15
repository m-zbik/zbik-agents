# Claude Code vs Gastown: When to Use Which

Comparison of the two orchestration approaches for zbik-agents.

---

## Table of Contents

- [How Each Works](#how-each-works)
- [Side-by-Side Comparison](#side-by-side-comparison)
- [Agent Definition Scope: Central vs Per-Project](#agent-definition-scope-central-vs-per-project)
- [Which to Choose for Your Workflow](#which-to-choose-for-your-workflow)
- [Migration Path](#migration-path)

---

## How Each Works

### Claude Code Native Agents (`.claude/agents/`)

Subagents run inside a single Claude session. One conversation, Claude delegates
to specialized subagents as needed.

| Aspect | Reality |
|---|---|
| Setup | 5 minutes -- markdown files, done |
| Parallelism | None -- subagents run one at a time within your session |
| Persistence | None -- context lost when session ends |
| Orchestration | Manual -- you tell Claude which agent to engage next |
| Infrastructure | Zero -- just Claude Code |
| Scaling | 1 human, 1 conversation, agents take turns |

### Gastown

Each agent runs in its own tmux session with its own Claude instance. A Go binary
orchestrates everything. Git-backed work tracking.

| Aspect | Reality |
|---|---|
| Setup | Hours -- Go binary, Dolt SQL, tmux, beads, hooks, town/rig config |
| Parallelism | Real -- 20-30 concurrent agents, each with own context |
| Persistence | Full -- survives restarts, session handoffs, git-backed state |
| Orchestration | Automated -- mayor assigns work, witness monitors, refinery merges |
| Infrastructure | Heavy -- Dolt DB, tmux sessions, git worktrees per agent |
| Scaling | Multiple parallel polecats coding different features simultaneously |

---

## Side-by-Side Comparison

| | Claude Code | Gastown |
|---|---|---|
| Agent definitions | Per-project (`project/zbik-agents/`) | Central (`~/gt/zbik-agents/`) |
| Agent presets | Per-project (`.claude/agents/`) | Central (`agents.json`) |
| Customization | Per-project `CLAUDE.md` additions | Per-rig overrides (`rig-settings.json`) |
| Adding a new project | Copy/submodule zbik-agents + create `.claude/agents/` | `gt rig add` -- instantly uses existing agents |
| Parallel agents | No -- one at a time in a single session | Yes -- 20-30 concurrent tmux sessions |
| Session persistence | No -- lost on restart | Yes -- git-backed, survives crashes |
| Merge queue | No -- manual PRs | Yes -- automated refinery |
| Health monitoring | No | Yes -- witness + deacon |
| Work tracking | No built-in tracking | Yes -- beads (git-backed issues) |
| Cost per session | 1 Claude session | N Claude sessions (one per agent) |

---

## Agent Definition Scope: Central vs Per-Project

### Gastown: Central (`~/gt/zbik-agents/`)

One copy of agent definitions shared by all projects (rigs):

```
~/gt/                              <-- one town
├── zbik-agents/                   <-- shared agent definitions (one copy)
├── settings/
│   ├── agents.json                <-- all agent presets (reference zbik-agents/ paths)
│   └── town-settings.json         <-- default role mappings
├── project-alpha/                 <-- rig 1 (uses shared zbik-agents/)
├── project-beta/                  <-- rig 2 (uses shared zbik-agents/)
└── project-gamma/                 <-- rig 3 (uses shared zbik-agents/)
```

- `agents.json` defines presets once (`backend`, `reviewer`, etc.)
- Every rig uses them automatically
- Each rig can override which preset goes to which role via `rig-settings.json`
- Agent definitions themselves are shared -- update once, all rigs get the change

### Claude Code: Per-Project

Each project needs its own copy of agent definitions:

```
project-alpha/
├── zbik-agents/                   <-- copy/submodule per project
├── .claude/agents/                <-- subagent files per project
└── CLAUDE.md

project-beta/
├── zbik-agents/                   <-- another copy/submodule
├── .claude/agents/                <-- another set of subagent files
└── CLAUDE.md
```

- Claude Code resolves paths relative to the project directory -- no central registry
- Each project needs its own `zbik-agents/` and `.claude/agents/`
- Git submodules help: `git submodule add` + one setup script per project
- Updates require `git submodule update` in each project

### Summary

| | Gastown | Claude Code |
|---|---|---|
| Agent definitions | Central (`~/gt/zbik-agents/`) | Per-project (`project/zbik-agents/`) |
| Agent presets | Central (`agents.json`) | Per-project (`.claude/agents/`) |
| Customization | Per-rig overrides (`rig-settings.json`) | Per-project `CLAUDE.md` additions |
| Adding a new project | `gt rig add` -- instantly uses existing agents | Copy/submodule zbik-agents + create .claude/agents/ |
| Updating agents | Update `~/gt/zbik-agents/` once | Update submodule in each project |

---

## Which to Choose for Your Workflow

The zbik-agents orchestration flow is inherently sequential with human gates:

```
BA --> [you review] --> Architect --> [you review] --> PM --> [you review] --> Specialists --> [you review]
```

### Start with Claude Code agents when:

- You are a solo developer or small team
- The sequential human-gated flow is your primary pattern
- You want something working in 5 minutes
- You don't need agents running overnight or across sessions
- Only phase 4 (sprint execution) would benefit from parallelism
- You want minimal infrastructure
- You want to work fully local without a GitHub remote

### Move to Gastown when:

- Phase 4 gets big -- you want 5+ polecats coding 5+ features simultaneously across git branches
- You need persistent tracking across multiple days/sessions (agents surviving restarts)
- You want automated merge queue (refinery) and health monitoring (witness)
- You run multiple projects (rigs) concurrently and want centralized agent management
- You need audit trails and git-backed work tracking (beads)
- Multiple humans are managing different parts of the same project
- You have GitHub repos for all projects (Gastown requires a remote URL per rig)

---

## Migration Path

The zbik-agents structure was designed to work with both systems. The same `.md`
agent definition files are used by Claude Code subagents and Gastown agent presets.
Only the orchestration layer changes.

### Claude Code --> Gastown

1. Install Gastown: `go install github.com/steveyegge/gastown/cmd/gt@latest`
2. Create town: `gt install ~/gt --shell`
3. Copy zbik-agents to town: `cp -r zbik-agents ~/gt/zbik-agents`
4. Register agents: paste `agents.json` from `INTEGRATION-GASTOWN.md`
5. Map roles: paste `town-settings.json` from `INTEGRATION-GASTOWN.md`
6. Add existing project as rig: `gt rig add my-project <repo-url>`
7. Done -- your agent definitions are unchanged, only the runner is different

### Gastown --> Claude Code

1. Copy zbik-agents from town into project: `cp -r ~/gt/zbik-agents project/zbik-agents`
2. Create `.claude/agents/` subagent files (see `INTEGRATION-CLAUDE-CODE.md`)
3. Add `@zbik-agents/CLAUDE.md` to project's `CLAUDE.md`
4. Done -- same agent definitions, just running inside a single Claude session

### What stays the same across both

- All agent `.md` files (`_base.md`, `roles/*`, `specialists/*`)
- The orchestration flow (BA --> Architect --> Security --> PM --> Specialists)
- Human review gates (4 gates)
- Coding standards (typing, tests, docstrings)
- The reviewer's role as persistent critic
