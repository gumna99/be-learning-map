# be-learning

Claude Code skill — BE engineer learning companion. Cross-references a BE Engineer Roadmap against speckit implementation tasks to identify learning opportunities during feature development.

## Installation

Copy the skill into your Claude Code skills directory:

```bash
cp -r . ~/.claude/skills/be-learning
```

## Prerequisites

- Claude Code with superpowers skill system
- A project using the speckit workflow (`/speckit.plan` -> `/speckit.tasks` -> `/speckit.implement`)

## Sub-commands

| Command | Purpose |
|---------|---------|
| `/be-learning map` | Predict which BE concepts a feature involves |
| `/be-learning explain {concept-id}` | Teach a concept using real codebase examples |
| `/be-learning quiz {concept-id}` | Self-test understanding with codebase questions |
| `/be-learning calibrate` | Compare predictions vs actual code, improve matching |
| `/be-learning progress` | View learning progress and mark concepts as learned |

## Workflow

```
Step 1: PM spec.md                              [Spec-Hub]
Step 2: Architect sequence + API contracts       [mint-tea]
Step 3: specify init . --ai claude --force       [Code Repo]
Step 4: /speckit.plan    -> plan.md              [Code Repo]
Step 5: /speckit.tasks   -> tasks.md             [Code Repo]
        /be-learning map    -> be-learning-map.json (predictions + auto-record encounters)
        /be-learning explain X -> deep-dive         (auto-record encounter)
Step 6: /speckit.implement                       [Code Repo]
        /be-learning calibrate                      (post-implementation accuracy)
        /be-learning progress                       (review & mark learned)
```

## Optional: Implement-time prompts

Add this snippet to your project's `CLAUDE.md` for brief learning hints during `/speckit.implement`:

```markdown
## BE Learning Prompts (opt-in)

When implementing a feature via `/speckit.implement`, if a `be-learning-map.json` file exists in the current feature's `specs/{feature}/` directory, show a brief learning hint at the start of each task that matches a BE concept. Format:

Learning hint: This task involves [concept name] — [one-sentence why from roadmap].
Run `/be-learning explain {concept-id}` to learn more.

Rules:
- Only show for HIGH confidence matches (not medium/low)
- One hint per task, max
- Never show hints if `be-learning-map.json` does not exist
- To disable: delete this section from CLAUDE.md
```

## File Structure

```
be-learning/
├── SKILL.md                       # Skill definition (5 sub-commands)
├── references/
│   ├── be-roadmap.json            # 41 BE concepts with matching rules
│   └── progress.json              # Personal learning progress (auto-created)
└── README.md
```

## Roadmap JSON

`references/be-roadmap.json` contains 41 backend engineering concepts across 5 pillars:

- **基本功** (7): Clean Code, SOLID, Design Patterns, Testing, Version Control, Type System, Programming Paradigms, DI
- **計算** (4): Execution Model, Memory Model, Algorithm Complexity, Numerical Precision
- **通訊** (10): Network Layers, API Design, Auth, Request Lifecycle, Web Security, Async Communication, Fault Tolerance, Service Communication, Observability, API Contract, Rate Limiting, Realtime, Event-Driven
- **狀態** (8): Domain Modeling, Relational Model, Query Execution, Transaction & ACID, Cross-Service Consistency, Idempotency, Schema Evolution, Caching, Distributed Consistency, Large-Scale Data, Pagination
- **基礎設施** (4): Containerization, Container Orchestration, CI/CD, Config & Secrets, Performance Testing

Each concept includes keywords, file patterns, code signatures, and context triggers for 3-layer matching.
