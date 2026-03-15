# zbik-agents

A portable, layered agent team for AI-assisted software development. Works with
Claude Code and Gastown. Defines 12 agents across 3 levels with an orchestrated
workflow and 4 human review gates.

---

## What is this?

A set of markdown files that define AI agent roles, responsibilities, guardrails,
and workflows. When imported into Claude Code or Gastown, they create a full
software development team that follows a structured process:

```
BA researches -> Architect designs -> Security hardens -> PM plans -> Specialists build
                    Reviewer challenges everyone at every step
                    Human reviews at 4 gates
```

## File Map

```
zbik-agents/
|
|-- README.md                        <-- You are here
|-- CLAUDE.md                        <-- Runtime: imported by Claude Code via @zbik-agents/CLAUDE.md
|                                        Contains workflow, agents, coding standards.
|                                        This is what the AI reads. Not a setup guide.
|
|-- _base.md                         <-- L0 Constitutional: guardrails inherited by ALL agents
|                                        (typing, testing, docstrings, security, ethics)
|
|-- roles/                           <-- L1 Role Archetypes (coordination, design, governance)
|   |-- business_analyst.md              Initiates all projects. Research-first. 3 solution tiers.
|   |-- architect.md                     System design. Diagrams (mermaid/drawio). 3 design tiers.
|   |-- project_manager.md              Sprint delivery. 3-tier plans. QA-dev sync. Risk management.
|   |-- reviewer.md                     Persistent critic. Challenges all agents at every phase.
|   |-- researcher.md                   On-demand tech research and evaluation.
|   |-- developer.md                    Base developer role (extended by specialists).
|
|-- specialists/                     <-- L2 Technology Specializations (extend developer.md)
|   |-- backend_developer.md             Python (venv), Java, Rust, C, C++, SQL, databases, APIs.
|   |-- frontend_developer.md            TypeScript, React, Next.js, accessibility.
|   |-- mobile_app_developer.md          Swift, Kotlin, React Native, Flutter.
|   |-- devops_engineer.md               Docker, K8s, Helm, GitHub Actions, git flow, Makefile.
|   |-- qa_automation_engineer.md        Test framework design. Feeds devs with test patterns.
|   |-- security_developer.md            Auth, encryption, STRIDE threat modeling.
|
|-- INTEGRATION-CLAUDE-CODE.md       <-- Setup guide: how to add zbik-agents to Claude Code
|-- INTEGRATION-GASTOWN.md           <-- Setup guide: how to add zbik-agents to Gastown
|-- CLAUDE-vs-GASTOWN.md             <-- Comparison: when to use which platform
```

## How to Read

**If you want to start using zbik-agents right now:**
1. Read `INTEGRATION-CLAUDE-CODE.md` (for Claude Code) or `INTEGRATION-GASTOWN.md` (for Gastown)
2. Follow the "How to Create a New Project" section
3. Start Claude Code and describe your project

**If you want to understand how the agents work:**
1. Read `_base.md` -- the constitutional layer all agents inherit
2. Read `roles/developer.md` -- the base developer pattern
3. Read any specialist (e.g., `specialists/backend_developer.md`) to see how L2 extends L1
4. Read `CLAUDE.md` -- the orchestration flow and coding standards

**If you want to decide between Claude Code and Gastown:**
1. Read `CLAUDE-vs-GASTOWN.md`

**If you want to modify or add agents:**
1. Follow the L0 -> L1 -> L2 inheritance pattern
2. For a new role: create `roles/your-role.md` with `extends: _base.md`
3. For a new specialist: create `specialists/your-spec.md` with `extends: roles/developer.md`

## Agent Hierarchy

```
L0  _base.md (Constitutional)
    |
    |-- All agents inherit guardrails: typing, testing, docstrings, security, ethics
    |
L1  roles/ (Archetypes)
    |
    |-- business_analyst     Coordination, not code
    |-- architect             Coordination, not code
    |-- project_manager       Coordination, not code
    |-- reviewer              Coordination, not code
    |-- researcher            Coordination, not code
    |-- developer             Base for all specialists
    |
L2  specialists/ (extend developer.md)
    |
    |-- backend_developer     Python, Java, Rust, C, C++, SQL
    |-- frontend_developer    TypeScript, React, Next.js
    |-- mobile_app_developer  Swift, Kotlin, Flutter
    |-- devops_engineer       Docker, K8s, Helm, GitHub Actions
    |-- qa_automation_engineer Test frameworks, CI integration
    |-- security_developer    Auth, encryption, threat modeling
```

L1 roles coordinate, design, and govern. L2 specialists write code and extend
the developer role with technology-specific guardrails.

## Orchestration Flow

```
 1. HUMAN + BA ----------> [HUMAN GATE 1: approve spec]
                      |
 2. Architect + Security + DevOps (rollout)
               ----------> [HUMAN GATE 2: approve design]
                      |
 3. PM (plan) ----------> [HUMAN GATE 3: approve plan + sources]
                      |
 4. Specialists + QA ---> [HUMAN GATE 4: milestone review]

    REVIEWER challenges ALL agents at EVERY step (independently)
```

| Gate | What you review | What you approve |
|---|---|---|
| 1 | Functional spec + documentation | Proceed to architecture |
| 2 | Architecture + security + rollout plan | Proceed to planning |
| 3 | Delivery plan, milestones, repos, data sources | Which tier to build |
| 4 | Milestone deliverables + demo | Not individual PRs |

## Key Design Decisions

- **Same agent files work with both Claude Code and Gastown** -- only the orchestration layer changes
- **Reviewer is always in the loop** -- challenges independently at every phase, not just at the end
- **Three-tier solutions everywhere** -- BA, Architect, and PM all produce simplified, transitional, and full-flagged options
- **DevOps before development** -- git flow, Docker, Makefile, CI/CD, Helm are set up before the first line of code
- **QA before coding** -- test briefs and frameworks are ready before developers start implementing
- **All agents use Claude Opus 4.6** -- see Model Configuration in the integration guides to change
