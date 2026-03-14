# zbik-agents

@./_base.md

## Team Workflow

When starting a new project, follow this exact flow. Each phase has a human
review gate -- nothing proceeds without explicit human approval at each gate.

### Phase 1: Research & Functional Spec
- **You talk to Business Analyst first** to describe the project
- BA conducts deep research (papers, official docs, GitHub, verified blogs)
- BA produces functional spec with 3 solution tiers and cost analysis
- **Reviewer challenges BA independently** -- you don't need to prompt it
- **HUMAN GATE 1**: You review the spec. Say "approved" to proceed to architecture.

### Phase 2: Architecture Design
- Architect creates 3 design tiers from your approved spec
- Security Developer reviews and hardens the design (iterates with Architect)
- Architect reviews with DevOps Engineer to plan rollout strategy
- Developers consulted if feasibility questions arise
- **Reviewer challenges all agents independently**
- **HUMAN GATE 2**: You review the full package (design + security + rollout). Approve to proceed.

### Phase 3: Delivery Planning
- Project Manager prepares 3-tier delivery plan with milestones
- PM documents ALL external sources: GitHub repos, data sets (with sizes), APIs
- PM selects specialists and creates sprint breakdown
- **Reviewer challenges the plan independently**
- **HUMAN GATE 3**: You approve milestones, repos, data sources, and which tier to build.

### Phase 4: Project Setup & Sprint Execution
- DevOps Engineer sets up: git flow, Dockerfiles, docker-compose, Makefile, GitHub Actions, Helm charts
- PM orchestrates specialists through sprints
- QA provides test briefs BEFORE developers start coding
- Git flow: feature/* -> develop -> release/* -> main
- **Reviewer challenges all code (types, tests, docstrings, security)**
- **HUMAN GATE 4**: You review at milestones only. You do NOT review individual PRs.

### Agents

| Agent | Role |
|---|---|
| Business Analyst | Initiates projects, research, functional specs |
| Architect | System design, diagrams, tech justification |
| Project Manager | Delivery planning, sprints, team coordination |
| Reviewer | Persistent critic at every phase |
| Researcher | On-demand tech research |
| Backend Developer | Python (venv), Java, Rust, C, C++, SQL |
| Frontend Developer | TypeScript, React, Next.js |
| Mobile App Developer | Swift, Kotlin, Flutter, React Native |
| DevOps Engineer | Docker, K8s, Helm, GitHub Actions, git flow |
| QA Automation Engineer | Test frameworks, automation, CI integration |
| Security Developer | Auth, encryption, threat modeling, STRIDE |

### Coding Standards
- Strong typing on ALL function signatures (params + return types)
- Unit tests for every function
- Docstrings with Args, Returns, Raises, and Example sections
- Python always uses `.venv` virtual environment
- No bare except/catch -- specific exceptions only
- No hardcoded secrets or magic numbers
