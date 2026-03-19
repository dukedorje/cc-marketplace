# sprint-plan

Multi-phase sprint planning workflow that transforms product ideas into implementation-ready user stories.

## Quick Start

```
/sprint-plan "your product idea here"
```

Or point it at an existing PRD file:

```
/sprint-plan path/to/prd.md
```

Or generate a structured PRD before planning:

```
/prd "your product idea here"
/sprint-plan path/to/prd-my-product.md
```

For help and usage info:

```
/sprint-plan:help
```

## What It Does

Runs a 6-phase workflow powered by specialized AI agents:

| Phase | What happens |
|-------|-------------|
| **0 - Discovery** | Scans your codebase, detects greenfield vs brownfield, finds existing artifacts |
| **1 - Requirements** | Expands idea into functional/non-functional requirements (parallel analyst + architect agents) |
| **1.5 - UX Design (optional)** | Generates component specs, user flows, and wireframes when frontend is detected |
| **2A - Architecture** | Makes architecture decisions in ADR-lite format with RALPLAN-DR consensus |
| **2B - Epic Design** | Groups requirements into user-value-focused epics with FR coverage map |
| **3 - Story Decomposition** | Breaks epics into dev-agent-sized stories with BDD acceptance criteria |
| **4 - Story Enrichment** | Adds technical details, file lists, testing requirements, anti-patterns |
| **5 - Validation** | Critic + verifier check FR coverage, dependency ordering, story quality |

## Modes

**Thorough** (default): Full RALPLAN-DR consensus loops in architecture and epic design phases. Decision Steering asks you about important decisions (HIGH/CRITICAL significance). Requirements-Architecture refinement loop catches circular dependencies.

**Fast** (`--fast`): Single-pass through all phases. No consensus loops. All decisions auto-made and logged. Use when speed matters more than exhaustive review.

```
/sprint-plan --fast "quick prototype idea"
```

## Flags

| Flag | Effect |
|------|--------|
| `--fast` | Single-pass, no consensus loops, all decisions auto-made |
| `--thorough` | Explicit thorough mode (the default, useful for clarity) |
| `--restart-from=<phase>` | Resume from a specific phase |
| `--skip-ux` | Skip the optional UX Design Phase (1.5) even when frontend is detected |

Valid restart phases: `discovery`, `requirements`, `ux-design`, `architecture`, `epic-design`, `story-decomposition`, `story-enrichment`, `validation`

## Output

Everything lands in `.omc/sprint-plan/sprint-NNN/`:

```
.omc/sprint-plan/
  current -> sprint-001/          # symlink to active sprint
  sprint-001/
    discovery.md                  # Phase 0: project context
    requirements.md               # Phase 1: FRs, NFRs, constraints
    ux-design.md                  # Phase 1.5: component specs and user flows (optional)
    architecture-decisions.md     # Phase 2A: ADR-lite decisions
    epics.md                      # Phase 2B: epic structure + stories
    stories/                      # Phase 4: enriched story files
      1-1-user-auth.md
      1-2-profile-setup.md
      2-1-dashboard-layout.md
    readiness-report.md           # Phase 5: validation summary
    retrospective.md              # /retro: sprint retrospective and next-sprint prep
    phase-state.json              # workflow state
  decisions/
    decision-log.md               # cross-sprint decision history
    active-decisions.md           # current non-superseded decisions
```

## Decision Steering

During thorough mode, the workflow presents important decisions for your input:

- **CRITICAL/HIGH** significance decisions are shown with options and recommendations
- You can choose an option, say "choose for me", or "choose for me and automate the rest"
- **MEDIUM/LOW** decisions are auto-made and logged

Say "stop and ask me" at any time to re-enter guided mode.

## Refining Results

Use `/ral` to trigger a RALPLAN-DR refinement pass (Planner → Architect → Critic consensus) on any phase output:

```
/ral architecture
/ral requirements
/ral epics
/ral retro
/ral prd
```

Each phase can be refined up to 2 times (use `--force` to override).

## After Planning

Execute the validated stories with `/sprint-exec`:

```
/sprint-exec
```

This dispatches executor agents for each story — epics run sequentially, stories within each epic run in parallel.

```
/sprint-exec --epic=2          # run only epic 2
/sprint-exec --story=1.3       # run (or retry) a single story
/sprint-exec --dry-run         # preview execution plan without running
```

Or feed stories into a manual implementation workflow:

```
/team ralph
```

## After Implementation

Generate a retrospective from the completed sprint:

```
/retro
```

Analyzes git history, story Dev Agent Records, and architecture decision adherence to produce a structured retrospective at `.omc/sprint-plan/current/retrospective.md`.

The retrospective is automatically consumed by Phase 0 of the next `/sprint-plan` run.

## Brownfield Support

For existing codebases, the workflow automatically:
- Produces an Existing Codebase Inventory in discovery
- Accounts for existing features in requirements (no duplication)
- Notes alignment/divergence with existing patterns in architecture decisions
- Includes Codebase Context sections in enriched stories

## Cross-Sprint Intelligence

Run `/retro` after each sprint to generate a retrospective. Sprint 2+ automatically loads:
- Previous sprint retrospective learnings (from `/retro` output)
- Unfinished work and promoted deferred requirements
- Architecture evolution proposals from execution experience
- Active (non-superseded) architecture decisions
- Patterns and problems from prior story implementations

## Requirements

This plugin uses agents from the [oh-my-claudecode](https://github.com/nicobailon/oh-my-claudecode) ecosystem: `analyst`, `architect`, `planner`, `critic`, `verifier`, `executor`, `writer`, `explore`, and `document-specialist`.

## License

MIT
