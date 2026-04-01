# Phase 0: Intake & Discovery

## Purpose

Load all available context, detect project type and input quality, and establish a baseline for the sprint planning workflow. For sprint N>1, load previous sprint intelligence and regenerate `active-decisions.md` from the full decision log.

This phase is fully automated — no user interaction required.

---

## Agent

**Agent**: `explore`
**Model**: `haiku`
**Duration**: ~30 seconds

---

## Inputs

- User-provided product idea or brief (if any)
- `.omc/sprint-plan/` directory tree
- Project working directory (codebase root)
- `decisions/active-decisions.md` (sprint N>1)
- `sprint-{N-1}/retrospective.md` (sprint N>1)

---

## Step-by-Step Process

### Step 1: Scan for Existing Planning Artifacts

Search `.omc/sprint-plan/` for any previously created planning artifacts:

- PRD or product requirement documents (`prd.md`, `requirements.md`, `product-brief.md`, or similar)
- Architecture docs (`architecture*.md`, `architecture-decisions.md`)
- Research notes (`.omc/research/`)
- Existing epics (`epics.md`)
- UX specs (`ux-design.md`, `ux*.md`, any Figma/design references)
- Sprint retrospectives (`sprint-*/retrospective.md`)
- Active decisions (`decisions/active-decisions.md`)

Record each found artifact and its path in the output under `## Available Artifacts`.

### Step 2: Detect New Repository

Scan the project working directory to determine if this is a brand-new repository:

**New repo indicators** (ALL must be true):
- Fewer than 10 source files in the project
- No build manifests (`package.json`, `pyproject.toml`, `Cargo.toml`, `go.mod`, `pom.xml`, etc.)
- No `src/`, `lib/`, `app/`, or `packages/` directories with code

**Classification rule**:
- All indicators true → `new_repo: true`
- Otherwise → `new_repo: false`

Record the classification. This flag is informational — the codebase inventory (Step 4) always runs regardless.

### Step 3: Detect Input Quality

Classify the user's provided product idea or brief:

| Classification | Criteria |
|---|---|
| `raw-idea` | 1–3 paragraphs, informal language, no numbered requirements, no personas |
| `structured-brief` | Has distinct sections (e.g., goals, features, constraints), but no formal FR numbering |
| `existing-prd` | Has numbered functional requirements (FR1, FR2, ...) or formal acceptance criteria |

If no input was provided, record `input_quality: raw-idea` and note that the user must provide context before Phase 1 can proceed (flag in Recommendations).

### Step 4: Extract Codebase Inventory

Scan the codebase and produce an "Existing Codebase Inventory". This always runs — even for new repos, capturing whatever exists provides useful context for downstream phases.

**Tech Stack**: Identify languages, frameworks, major libraries (read `package.json` dependencies, `pyproject.toml`, `Cargo.toml`, etc.).

**Project Structure**: List top-level directories and their purpose (infer from directory names and file contents if needed).

**Existing Patterns**: Identify architectural patterns in use:
- Monorepo vs single-package
- Frontend framework (React, Vue, Svelte, etc.)
- Backend framework (Express, FastAPI, Axum, etc.)
- ORM or database access pattern
- Auth patterns (if any)
- API style (REST, GraphQL, tRPC, etc.)
- Test framework and patterns

**Module Boundaries**: Identify distinct modules or domains in the codebase (e.g., `auth`, `billing`, `dashboard`).

This inventory is referenced by ALL downstream phases. Be thorough here.

### Step 5: Check for UX Artifacts

Determine whether the project has frontend requirements:

**Frontend indicators**: presence of a frontend framework in tech stack, UI-related directories (`components/`, `pages/`, `views/`), or UI-related terms in the product brief ("screen", "interface", "UI", "dashboard", "form").

**UX artifact indicators**: `ux-design.md`, Figma links, wireframe files, design system references.

**Decision logic**:
- `has_frontend: false` → UX check not applicable
- `has_frontend: true` AND UX artifacts found → `has_ux_artifacts: true`
- `has_frontend: true` AND no UX artifacts → `has_ux_artifacts: false`
  - Add to Recommendations: "No UX artifacts found. Consider creating UX specs before proceeding to Phase 4 story enrichment. Phase 4 references `ux-design.md` when present."

### Step 6: Load User's Product Idea

If the user provided a product idea or brief, read and summarize it under `## Project Overview` and `## Input Analysis`.

If nothing was provided, note this in `## Input Analysis` as a blocker for Phase 1.

### Step 7: Sprint N>1 — Load Previous Sprint Intelligence

**Only if this is not the first sprint** (i.e., `sprint-{N-1}/` directory exists).

Perform the following two sub-steps:

**7a. Read prior retrospective.**
Read `sprint-{N-1}/retrospective.md` and extract:
- Key learnings that should influence this sprint
- Technical debt or unresolved problems noted
- Velocity/scope lessons

**7b. Regenerate `active-decisions.md`.**
Read the full `decisions/decision-log.md`. For each decision:
- If status is `accepted` and not superseded by a later decision → include in active-decisions
- If status is `superseded by D-{NNN}` → exclude (the superseding decision replaces it)
- Write the compacted, current-only view to `decisions/active-decisions.md`

This regeneration ensures `active-decisions.md` is always accurate as of the current sprint.

### Step 8: Write Output

Write all findings to `SPRINT_DIR/discovery.md` using the schema below.

---

## Output Schema: `current/discovery.md`

```markdown
---
project: [name]
sprint: sprint-[NNN]
created: [date]
new_repo: true | false
input_quality: raw-idea | structured-brief | existing-prd
has_ux_artifacts: true | false
has_frontend: true | false
previous_sprint: sprint-[NNN-1] | null
---

## Project Overview
[Brief description of the product idea — 2-5 sentences. If no input provided, state that.]

## Input Analysis
[What was provided, quality assessment, and any gaps. Note if input is too sparse for Phase 1 to proceed without clarification.]

## New Repo Detection
[New repo or existing codebase. Note source file count and presence of build manifests/source directories.]

## Existing Codebase Inventory
### Tech Stack
[Languages, frameworks, libraries with versions where available]

### Project Structure
[Top-level directory listing with inferred purpose]

### Existing Patterns
[Architectural patterns identified: API style, auth pattern, ORM, test framework, etc.]

### Module Boundaries
[Distinct domains or modules in the codebase]

## Available Artifacts
[Bulleted list of found planning documents with paths. If none, state "No prior planning artifacts found."]

## UX Status
[Either: "UX artifacts found: [list]" or "No UX artifacts found. Consider creating UX specs before proceeding."]

## Previous Sprint Intelligence (sprint N>1 only)
### Key Learnings from Retrospective
[Extracted from sprint-{N-1}/retrospective.md]

### Active Decisions Carried Forward
[Summary of non-superseded decisions from regenerated active-decisions.md]

## Recommendations
[Flags or suggestions for subsequent phases. Examples:
- "Input quality is raw-idea — Phase 1 analyst agent will expand to include persona/metrics extraction."
- "No UX artifacts found. Consider providing UX specs before Phase 4."
- "No product brief provided — Phase 1 cannot proceed until user provides context."
- "Existing codebase detected. Codebase inventory populated above."
]
```

---

## Agent Dispatch

Dispatch to the explore agent using the `Agent` tool:

```python
Agent(
    subagent_type="oh-my-claudecode:explore",
    model="haiku",
    prompt="""
You are running Phase 0: Intake & Discovery for the sprint-plan workflow.

Working directory: {working_directory}
Sprint directory: SPRINT_DIR/
Sprint number: {sprint_number}
User input: {user_input_or_none}

Follow the Phase 0 instructions in full:
.omc/sprint-plan/ → plugins/sprint-plan/skills/sprint-plan/phases/phase-0-discovery.md

Execute all 8 steps in order. Write output to SPRINT_DIR/discovery.md.
Do not ask for confirmation. Complete the phase and return when done.
""",
)
```

---

## Phase Completion

Phase 0 is complete when `current/discovery.md` exists and contains all required sections.

Update `current/phase-state.json`:
```json
{
  "current_phase": "requirements",
  "stale_phases": []
}
```

Proceed automatically to Phase 1: Requirements Expansion.
