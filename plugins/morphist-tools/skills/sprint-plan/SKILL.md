---
name: sprint-plan
description: Multi-phase sprint planning workflow that transforms product ideas into implementation-ready user stories with architecture decisions, requirements expansion, and adversarial validation
user-invocable: true
argument-hint: "[product-idea-or-prd-path] [--fast] [--thorough] [--auto] [--step] [--continue[=phase]] [--restart-from=phase]"
---

# Sprint Plan Orchestrator

You are the orchestrator for a multi-phase sprint planning workflow. Follow these instructions precisely, phase by phase. You coordinate OMC agents to transform a product idea into implementation-ready user stories.

---

## 1. Argument Parsing

Parse `$ARGUMENTS` for the following:

1. **Product input**: The remaining text after flags are extracted. This may be:
   - A file path (check if it exists) pointing to a product idea, PRD, or brief
   - Inline text describing the product idea
   - Empty (prompt the user: "What would you like to build?")

2. **Flags**:
   - `--fast`: Single-pass mode. No RALPLAN-DR consensus loops. Decision Steering starts in AUTONOMOUS mode. Refinement loops disabled. Implies `--auto` (no inter-phase pauses).
   - `--thorough`: Full consensus mode. RALPLAN-DR active in Phases 2A/2B. Decision Steering starts in GUIDED mode. Refinement loops enabled.
   - `--auto`: Run all phases without pausing between them. Pauses only for Decision Steering elicitations in GUIDED mode.
   - `--step`: Pause after EVERY phase with a summary for user review. Maximum control.
   - `--skip-ux`: Skip Phase 1.5 (UX Design) even if frontend requirements are detected and no UX artifacts exist.
   - `--continue`: Resume the current sprint from the next incomplete phase. Reads `phase-state.json` to auto-detect where to pick up. Optionally accepts a phase name (`--continue=<phase>`) to resume from a specific phase without re-running it.
   - `--restart-from=<phase>`: Re-run a specific phase and everything downstream. Use this when you want to redo a completed phase. Valid values: `discovery`, `requirements`, `ux-design`, `architecture`, `epic-design`, `story-decomposition`, `story-enrichment`, `validation`. Marks the specified phase and all downstream phases as stale.

If both `--fast` and `--thorough` are provided, `--thorough` wins.
If both `--continue` and `--restart-from` are provided, `--restart-from` wins (it's the more specific intent).
`--fast` implies `--auto`. `--step` overrides `--auto` if both are provided.

Store the parsed input for use throughout the workflow.

---

## 2. Initialization

### 2a. Determine Sprint Number

**Skip this step if `--continue` or `--restart-from` is set** — those flags resume an existing sprint (see 2g/2h).

1. Check `.omc/sprint-plan/` for existing `sprint-NNN/` directories.
2. If none exist, this is sprint 1 (`sprint-001`).
3. If directories exist, increment: find the highest NNN and use NNN+1.
4. Store the sprint number (zero-padded 3 digits).

### 2b. Create Sprint Directory

**Skip this step if `--continue` or `--restart-from` is set.**

```bash
mkdir -p .omc/sprint-plan/sprint-{NNN}/stories
mkdir -p .omc/sprint-plan/decisions
```

### 2c. Update Symlink

**Skip this step if `--continue` or `--restart-from` is set** — handled in 2g/2h.

```bash
rm -f .omc/sprint-plan/current
ln -s sprint-{NNN} .omc/sprint-plan/current
```

### 2c2. Write AGENTS.md

Write `.omc/sprint-plan/AGENTS.md` (create or overwrite). This is a static pointer so any agent working in the repo knows where sprint artifacts live:

```markdown
# Sprint Plan

Sprint planning artifacts are in `.omc/sprint-plan/current/` (symlink to active sprint).
Refer to files there for architecture decisions, requirements, story specs, and execution context.
```

This file is written once and does not need to be kept in sync — the `current` symlink handles resolution.

### 2d. Determine Mode

- All sprints default to `thorough` mode.
- Use `--fast` to opt into single-pass mode (no RALPLAN-DR, autonomous steering).
- Flags always override the default.

### 2e. Initialize phase-state.json

Write `.omc/sprint-plan/current/phase-state.json`:

```json
{
  "sprint": "sprint-{NNN}",
  "mode": "thorough|fast",
  "active": true,
  "current_phase": "discovery",
  "steering_mode": "GUIDED|AUTONOMOUS",
  "significance_calibration": [],
  "decisions_log": [],
  "refinement_loops": {
    "requirements_architecture": { "count": 0, "max": 3 }
  },
  "ral_passes": {
    "requirements": 0,
    "architecture": 0,
    "epics": 0,
    "stories": 0,
    "enrichment": 0
  },
  "stale_phases": [],
  "epics_count": 0,
  "stories_total": 0,
  "stories_enriched": 0,
  "validation_status": "pending"
}
```

Set `steering_mode` to `GUIDED` if thorough, `AUTONOMOUS` if fast.

### 2f. Register with OMC State (optional)

If OMC state tools (`state_write`) are available, register the session. If unavailable, skip — the workflow uses `phase-state.json` as the primary state store.
```json
{
  "mode": "sprint-plan",
  "active": true,
  "current_sprint": "sprint-{NNN}",
  "session_id": "<generate unique id>"
}
```

### 2g. Handle --continue

If `--continue` is specified (with or without a phase argument):

1. Find the most recent sprint directory (highest `sprint-NNN/`). Do NOT create a new sprint directory — this resumes an existing one.
2. Read `phase-state.json` from that sprint.
3. **Without a phase argument** (`--continue`):
   - Read `current_phase` from `phase-state.json`.
   - Determine the next phase in the chain after `current_phase`. That is the resume point.
   - If `current_phase` is `"validation"`, report: "Sprint {NNN} is already complete."
4. **With a phase argument** (`--continue=<phase>`):
   - Verify the artifact for the specified phase already exists (it was completed previously).
   - The resume point is the next phase after the specified one.
5. Update `current_phase` in `phase-state.json` to the resume point.
6. Update the `current` symlink to point to this sprint directory.
7. Skip directly to the resume phase in the orchestration loop below.

Report to user: "Resuming sprint {NNN} from Phase {X}: {phase name}."

### 2h. Handle --restart-from

If `--restart-from=<phase>` is specified:
1. Find the most recent sprint directory (highest `sprint-NNN/`). Do NOT create a new sprint directory.
2. Read `phase-state.json` from that sprint.
3. Verify all artifact files for phases BEFORE the restart phase exist.
4. Update `current_phase` in `phase-state.json` to the restart phase.
5. Mark the restart phase and all downstream phases as stale in `stale_phases`.
6. Update the `current` symlink to point to this sprint directory.
7. Skip directly to the specified phase in the orchestration loop below.

Report to user: "Restarting sprint {NNN} from Phase {X}: {phase name}. Downstream phases marked stale."

---

## 3. Phase Orchestration

Execute phases sequentially. For each phase:
1. Read the phase instruction file from `${CLAUDE_SKILL_DIR}/phases/` for detailed agent prompts.
2. Load the previous phase's output artifact (context shedding -- load the file, do not rely on memory).
3. Dispatch to the appropriate agent(s).
4. Write the phase output to the correct file path.
5. Update `current_phase` and metrics in `phase-state.json`.
6. Run Decision Steering if the phase is in an active steering zone.
7. **Inter-Phase Summary & Pause** (see section 3a below).

### 3a. Inter-Phase Summary & Pause

After each phase completes (steps 1-6), determine whether to pause based on the pause mode:

**Pause modes**:

| Mode | Pauses after |
|------|-------------|
| Default | **Decision points only**: Requirements (Phase 1), Architecture (Phase 2A), Validation (Phase 5) |
| `--step` | Every phase |
| `--auto` / `--fast` | Never (proceeds immediately; only Decision Steering elicitations in GUIDED mode can pause) |

**Decision point phases** are where the user's input has the most impact — scope (requirements), technology choices (architecture), and go/no-go (validation). Other phases are downstream work that flows from those decisions.

If the current phase is NOT a pause point for the active mode, skip the summary and proceed immediately.

**Summary format**:

```
═══════════════════════════════════════════════════
  Phase Complete: {phase_name}
═══════════════════════════════════════════════════

  Artifact: {artifact_path}

  What was produced:
    {2-4 bullet summary of the key outputs — e.g., "12 functional requirements",
     "5 architecture decisions (2 CRITICAL)", "3 epics with 11 stories"}

  Key decisions made:
    {list significant decisions from this phase, if any}
    {or "No significant decisions in this phase."}

  Things to consider:
    {1-3 items the user might want to review, steer, or discuss}
    {e.g., "FR7 (real-time sync) has high complexity — worth reviewing scope"}
    {e.g., "D-003 chose SSE over WebSocket — affects Stories 2.3, 3.1"}
    {e.g., "Epic 2 has 8 stories — close to the split threshold"}

  ─────────────────────────────────────────────────
  Next: {next_phase_name}

  Options:
    continue       — proceed to {next_phase_name}
    review         — open the artifact for detailed review
    edit           — make changes before proceeding
    /ral {phase}   — run a refinement pass on this phase
    /sprint-plan --restart-from={phase} — redo this phase
═══════════════════════════════════════════════════
```

Wait for user input before proceeding to the next phase.

**Phase-specific summary content**:

| Phase | "What was produced" highlights |
|-------|-------------------------------|
| Discovery | Tech stack, project type, existing artifacts found, new_repo flag |
| Requirements | FR count, NFR count, constraint count, open questions count |
| UX Design | Component count, screen count, interaction patterns |
| Architecture | Decision count by significance, consensus iterations used |
| Epic Design | Epic count, FR coverage %, dependency chain shape |
| Story Decomposition | Story count per epic, total stories, health flags |
| Story Enrichment | Stories enriched, avg file count per story, tech stack referenced |
| Validation | Pass/fail status, critical findings, auto-fixed count |

**"Things to consider"** — generate these by scanning the phase output for:
- High-significance decisions (CRITICAL/HIGH)
- Open questions or assumptions
- Health flags (epic too large, orphan FRs, uncovered decisions)
- Complexity hotspots
- Anything that changed from previous phases (if resuming)

If the user says "continue" (or equivalent), proceed to the next phase.
If the user provides feedback or edits, incorporate them before proceeding.

---

### Phase 0: Intake & Discovery (~30s)

**Read phase instructions from `${CLAUDE_SKILL_DIR}/phases/phase-0-discovery.md`**

**Agent**: `explore` (haiku)

**Steps**:
1. Prepare the context payload:
   - The product idea/brief (from argument parsing)
   - Project root path for codebase scanning
   - For sprint N>1: path to `sprint-{N-1}/retrospective.md` and `decisions/active-decisions.md`
2. Dispatch to `explore` agent with the phase instruction prompt and context.
3. The agent must:
   - Scan for existing planning artifacts (PRD, architecture docs, research, existing epics, UX specs)
   - Detect new repo vs existing codebase
   - Detect input quality: raw idea (1-3 paragraphs) vs structured brief vs existing PRD with FRs
   - Extract project structure, tech stack, existing patterns into "Existing Codebase Inventory"
   - Check for UX artifacts. If frontend requirements detected but no UX artifacts, note: "No UX artifacts found. Consider creating UX specs before proceeding."
   - For sprint N>1: read previous sprint retrospective and active decisions. Regenerate `decisions/active-decisions.md` from `decisions/decision-log.md`.
4. Write output to `.omc/sprint-plan/current/discovery.md`
5. Update `phase-state.json`: set `current_phase` to `"discovery"` (completed).

**Inter-Phase Summary** (section 3a): Discovery is not a decision point — default mode proceeds immediately to Phase 1. In `--step` mode, pauses here for review.

---

### Phase 1: Requirements Expansion

**Read phase instructions from `${CLAUDE_SKILL_DIR}/phases/phase-1-requirements.md`**

**Agents** (dispatch in parallel):
- `analyst` (opus): Extract testable requirements, identify missing guardrails, scope risks, unvalidated assumptions. For raw idea input (detected in Phase 0): also extract user personas and success metrics.
- `architect` (opus): Identify technical constraints, integration requirements, infrastructure needs, security requirements.
- `explore` (haiku): Scan codebase for existing implementations that constrain requirements. Skip only for new repos with no source files.

**PRD-Status-Aware behavior** (when input is an existing PRD file):
- Read the PRD's `status` frontmatter field:
  - `status: validated` → **light-touch mode**: focus on expanding FRs into sprint-ready detail, skip re-validation of goals/personas (treat them as approved). Note in requirements.md that this sprint is based on a validated PRD.
  - `status: draft` or `status: draft-with-issues` → **full validation mode**: run complete requirements expansion including validating goals, assumptions, personas, and identifying gaps or contradictions in the PRD.
  - No `status` field or `status: unknown` → default to full validation mode.
- Read the PRD's `scope_tier` frontmatter field (if present):
  - Filter FRs to only those tagged `tier: mvp` (or equivalent) for this sprint's scope.
  - FRs tagged `tier: future` or `tier: v2` are noted in Scope Boundaries → Out of Scope.
  - If no `scope_tier` tagging exists, include all FRs and note in Open Questions that tier prioritization was not present.

**Steps**:
1. Read `.omc/sprint-plan/current/discovery.md` (context shedding).
2. Read the product idea/brief.
3. Dispatch all agents in parallel. Each receives: product idea + discovery results.
4. Collect all agent outputs.
5. Merge results into a unified `requirements.md` following this structure:
   - YAML frontmatter (project, sprint, created date, steering_mode, previous_sprint, input_quality)
   - Product Vision
   - User Personas (if raw idea input)
   - Success Metrics (if raw idea input)
   - Functional Requirements (FR1, FR2, ...)
   - Non-Functional Requirements (NFR1, NFR2, ...)
   - Technical Constraints (TC1, TC2, ...)
   - Open Questions
   - Scope Boundaries (In Scope / Out of Scope)
   - Assumptions (with validation method + impact if wrong)
   - Previous Sprint Intelligence (for sprint N>1)
   - Existing Codebase Inventory
6. Write to `.omc/sprint-plan/current/requirements.md`
7. Update `phase-state.json`: set `current_phase` to `"requirements"`.
8. **Decision Steering**: Evaluate the merged requirements for significant decisions (see Section 4). If any HIGH or CRITICAL decisions are found and steering is GUIDED, present them to the user before proceeding.

---

### Phase 1.5: UX Design (Optional)

**Read phase instructions from `${CLAUDE_SKILL_DIR}/phases/phase-1.5-ux-design.md`**

**Trigger logic** (evaluate after Phase 1 completes):
1. Read `current/discovery.md` frontmatter.
2. Check `has_frontend` and `has_ux_artifacts` fields.
3. **Run Phase 1.5 if ALL of the following are true**:
   - `has_frontend: true`
   - `has_ux_artifacts: false`
   - `--skip-ux` flag is NOT set
   - Mode is NOT `fast` (AUTONOMOUS steering)
4. **Skip Phase 1.5** (set `ux_design_phase: "skipped"` in `phase-state.json`) if any condition fails.

**In GUIDED mode**: Present decision point to user before running (see phase instruction file for format). Wait for user response.

**In AUTONOMOUS mode or `--fast`**: Skip Phase 1.5 automatically. UX design is opt-in in fast/autonomous mode.

**Agent**: `designer` (sonnet)

**Output**: `.omc/sprint-plan/current/ux-design.md`

**State update**: Set `ux_design_phase: "complete"` (or `"skipped"`) in `phase-state.json`. Set `current_phase: "ux-design"` when run.

---

### Phase 2A: Architecture Decisions

**Read phase instructions from `${CLAUDE_SKILL_DIR}/phases/phase-2a-architecture.md`**

**Steps**:
1. Read `.omc/sprint-plan/current/requirements.md` (context shedding).
2. Read `.omc/sprint-plan/current/discovery.md` for project context.
3. If `current/ux-design.md` exists, read it for frontend component and UX context. Include relevant UX constraints in the architect agent's context (component hierarchy, screen specs, interaction patterns that affect API shape or state management).
4. If `decisions/active-decisions.md` exists, read it for cross-sprint decision context.

**In thorough mode** -- RALPLAN-DR consensus (max 3 iterations):
1. Dispatch to `planner` (opus): Propose architecture decisions structured as ADR-lite records covering Data Architecture, Authentication/Security, API/Communication, Frontend Architecture (if applicable), Infrastructure/Deployment.
2. Dispatch to `architect` (opus): Review planner's proposals for soundness, feasibility, and alignment with existing patterns.
3. Dispatch to `critic` (opus): Validate decision quality, identify gaps, challenge weak rationale.
4. If critic identifies issues: loop back to step 1 with critic's feedback (max 3 iterations).
5. On consensus or max iterations reached: finalize decisions.

**In fast mode** -- single pass:
1. Dispatch to `planner` (opus): Produce architecture decisions in a single pass. Note any unresolved conflicts.

**After decisions are finalized**:
1. Write to `.omc/sprint-plan/current/architecture-decisions.md` using ADR-lite format:
   ```
   ## D-{NNN}: {Title}
   **Date**: {date}
   **Sprint**: sprint-{NNN}
   **Significance**: CRITICAL | HIGH | MEDIUM
   **Decided by**: user | auto-decided
   **Status**: accepted

   **Context**: [why this came up]
   **Decision**: [what was chosen]
   **Alternatives**: [what was rejected and why]
   **Consequences**: [what this constrains downstream]
   ```
2. Append entries to `decisions/decision-log.md` (create if it doesn't exist).
3. Update `decisions/active-decisions.md` with current non-superseded decisions.
4. Update `phase-state.json`: set `current_phase` to `"architecture"`.

**Decision Steering is MAXIMALLY ACTIVE here.** Every decision must have a significance classification. In GUIDED mode, present all HIGH and CRITICAL decisions to the user using the elicitation format (Section 4).

**Requirements-Architecture Refinement Loop** (thorough mode only):
- After architecture decisions are finalized, check if any decision invalidates, constrains, or creates new requirements not in `requirements.md`.
- If yes: loop back to Phase 1 with the new constraints. Phase 1 re-runs incrementally (only new/changed requirements). Then Phase 2A re-runs.
- Track in `phase-state.json`: increment `refinement_loops.requirements_architecture.count`. Max 3 iterations.
- After max iterations, unresolved conflicts become Decision Steering elicitation points.

---

### Phase 2B: Epic Design

**Read phase instructions from `${CLAUDE_SKILL_DIR}/phases/phase-2b-epic-design.md`**

**Steps**:
1. Read `.omc/sprint-plan/current/requirements.md` (context shedding).
2. Read `.omc/sprint-plan/current/architecture-decisions.md` (context shedding).

**In thorough mode** -- RALPLAN-DR consensus (max 3 iterations):
1. Dispatch to `planner` (opus): Propose epic structure following these principles:
   - User-value-focused grouping (not technical layers)
   - Each epic standalone and enabling future epics
   - FR coverage map showing every FR mapped to an epic
   - No forward dependencies
2. Dispatch to `architect` (opus): Review epic boundaries against architecture decisions.
3. Dispatch to `critic` (opus): Validate FR coverage completeness and dependency ordering.
4. If critic identifies issues: loop (max 3 iterations).

**In fast mode** -- single pass:
1. Dispatch to `planner` (opus): Produce epic structure in a single pass.

**Output**: Write to `.omc/sprint-plan/current/epics.md` with:
- YAML frontmatter (project, sprint, inputDocuments, consensus_iterations)
- Requirements Inventory (FRs, NFRs, Architecture Requirements)
- FR Coverage Map (every FR -> epic)
- Epic List with goals and FR assignments

Update `phase-state.json`: set `current_phase` to `"epic-design"`, update `epics_count`.

**Decision Steering**: Active but less frequent than Phase 2A. Present significant epic-boundary decisions in GUIDED mode.

---

### Phase 3: Story Decomposition

**Read phase instructions from `${CLAUDE_SKILL_DIR}/phases/phase-3-story-decomposition.md`**

**Steps**:
1. Read `.omc/sprint-plan/current/epics.md` (context shedding).
2. Read `.omc/sprint-plan/current/architecture-decisions.md` for technical constraints.

3. Process epics SEQUENTIALLY (later epics reference patterns from earlier ones):

   For each epic:
   a. Dispatch to `planner` (opus): Decompose the epic into stories following these rules:
      - Stories sized for single dev agent completion
      - No forward dependencies within an epic
      - Database/entity creation only when needed by the story
      - Each story references which FRs it implements
      - User story format: "As a [role], I want [action], so that [benefit]"

   b. For each story produced by the planner, dispatch to `writer` (sonnet) in parallel:
      - Write BDD acceptance criteria (Given/When/Then format)
      - Define scope boundaries (DOES / does NOT)
      - Add technical notes referencing architecture decisions

4. Add stories to `.omc/sprint-plan/current/epics.md` under each epic section.
5. Compute and append epic health metrics: story count per epic, estimated complexity, dependency flags.
6. Update `phase-state.json`: set `current_phase` to `"story-decomposition"`, update `stories_total`, `epics_count`.

**Decision Steering**: DORMANT. All significant decisions were made in Phases 1-2.

---

### Phase 4: Story Enrichment

**Read phase instructions from `${CLAUDE_SKILL_DIR}/phases/phase-4-story-enrichment.md`**

**Steps**:
1. Read `.omc/sprint-plan/current/epics.md` (context shedding -- load story stubs).
2. Read `.omc/sprint-plan/current/architecture-decisions.md`.

3. If `current/ux-design.md` exists, read it (context shedding). It will be used to enrich frontend stories.

4. Process stories in parallel within each epic (sequential across epics to enable forward intelligence):

   For each story, dispatch in parallel:
   - `executor` (sonnet): Enrich the story with:
     - Architecture compliance (references to specific decisions)
     - Technical requirements (libraries with versions)
     - File structure (files to create/modify)
     - Testing requirements (framework, coverage, test patterns)
     - Scope boundaries (explicit DOES / does NOT lists)
     - Anti-patterns to avoid
     - Previous story intelligence (files created, patterns established, problems from prior stories in this epic)
     - For sprint N>1: relevant patterns from previous sprint stories
     - Codebase Context section (existing files to modify, patterns to follow, integration points — "N/A" for new repos)
     - For frontend stories (when story implements a frontend FR and `ux-design.md` exists): add a `## UX Specifications` section with the relevant component specs, screen specifications, and interaction patterns from `ux-design.md`
   - `document-specialist` (sonnet): Research latest versions of referenced technologies, API docs, library compatibility.

5. After enrichment, run an **LLM optimization pass** on each story: review for token efficiency, reduce verbosity, eliminate redundancy, make instructions actionable and unambiguous for dev agents.

6. Run **adversarial validation checklist** per story:
   - Does every acceptance criterion have a matching task?
   - Are scope boundaries explicit?
   - Are architecture decisions referenced?
   - Are there any implicit dependencies?
   - Apply corrections automatically.

7. Write each enriched story to `.omc/sprint-plan/current/stories/{epic_num}-{story_num}-{slug}.md` using the story file template:
   ```
   ---
   sprint: sprint-{NNN}
   epic: {N}
   story: {M}
   status: ready-for-dev
   frs: [FR1, FR3]
   decisions: [D-001, D-003]
   ---

   # Story {epic}.{story}: {title}

   ## Story
   As a [role], I want [action], so that [benefit].

   ## Acceptance Criteria
   (BDD Given/When/Then)

   ## Scope Boundaries
   - This story DOES: [list]
   - This story does NOT: [list]

   ## Tasks / Subtasks
   (checkboxes linked to ACs)

   ## Architecture Compliance
   (references to decisions)

   ## Technical Requirements
   (libraries with versions)
   ### Testing
   (framework, coverage, patterns)

   ## Previous Story Intelligence
   (files, patterns, problems from prior stories)

   ## Codebase Context
   (existing files, patterns, integration points)

   ## Anti-Patterns to Avoid

   ## References
   (traceability to requirements.md + architecture-decisions.md)

   ## Dev Agent Record
   ### Agent Model Used
   ### Completion Notes
   ### File List
   ### Debug Log References
   ```

7. Update `phase-state.json`: set `current_phase` to `"story-enrichment"`, increment `stories_enriched` as each story completes.

**Decision Steering**: DORMANT.

---

### Phase 5: Validation Gate

**Read phase instructions from `${CLAUDE_SKILL_DIR}/phases/phase-5-validation.md`**

**Agents**: `critic` (opus) + `verifier` (sonnet)

**Steps**:
1. Read ALL artifacts (context shedding -- load each file fresh):
   - `.omc/sprint-plan/current/requirements.md`
   - `.omc/sprint-plan/current/architecture-decisions.md`
   - `.omc/sprint-plan/current/epics.md`
   - All files in `.omc/sprint-plan/current/stories/`

2. Dispatch to `critic` (opus) for full coverage validation:
   - **FR Coverage**: Every FR has at least one story
   - **Architecture Compliance**: Every architecture decision reflected in relevant stories
   - **Dependency Validation**: No forward dependencies within or across epics
   - **Story Quality**: Each story completable by single agent, has BDD criteria, has scope boundaries
   - **Consistency**: Story file contents align with epics.md stubs
   - **Epic Health**: Flag epics with >8 stories (too large), <2 stories (too small), or cross-epic dependency concerns

3. Dispatch to `verifier` (sonnet) for mechanical correctness:
   - YAML frontmatter validity in all story files
   - FR references exist in requirements.md
   - Decision references exist in architecture-decisions.md
   - No duplicate story numbers
   - All stories listed in epics.md have corresponding story files

4. Collect validation results.

**If validation FAILS** (auto-fix loop, max 2 iterations):
   1. Identify the specific failures.
   2. Dispatch appropriate agents to fix:
      - Coverage gaps: dispatch `planner` to add missing stories
      - Story quality issues: dispatch `executor` to re-enrich
      - Consistency issues: fix directly
   3. Re-run validation.
   4. If still failing after 2 fix iterations: include failures in readiness report for user review.

**If validation PASSES**:
   1. Generate readiness report.

5. Write `.omc/sprint-plan/current/readiness-report.md` containing:
   - Overall status: READY / READY WITH WARNINGS / NOT READY
   - FR coverage matrix
   - Architecture compliance summary
   - Epic health report (story counts, flagged epics)
   - Validation issues (if any remain)
   - Decision summary (all decisions made, how decided)
   - Sprint statistics (epics, stories, estimated complexity)

6. Update `phase-state.json`: set `current_phase` to `"validation"`, set `validation_status` to `"pass"` or `"fail"`.

7. Present the readiness report to the user. This is the **final checkpoint**.

---

## 4. Decision Steering System

### 4a. Significance Classification

Every decision produced by an agent must include a significance level and rationale.

**Levels**:
- **CRITICAL**: Core value proposition, technology constraining entire project, security model, data model affecting all features
- **HIGH**: Architecture patterns, API design approach, authentication method, database selection
- **MEDIUM**: Library selection within category, testing strategy, deployment target
- **LOW**: Naming conventions, file organization, formatting

**Calibration examples** (include in Phase 2A agent prompts):

| Example Decision | Level | Why |
|-----------------|-------|-----|
| "Use PostgreSQL vs MongoDB for primary data store" | CRITICAL | Constrains all data access patterns across every epic |
| "REST vs GraphQL for API layer" | HIGH | Constrains frontend data fetching and all story task structures |
| "Use Zod vs Joi for input validation" | MEDIUM | Library swap within same category; limited blast radius |
| "Name the config file config.yaml vs settings.json" | LOW | Cosmetic; zero downstream impact |

### 4b. Self-Calibration

Store the first 3 user overrides (where the user escalates or de-escalates a significance level) in `phase-state.json` under `significance_calibration`:

```json
{
  "decision": "Cache strategy",
  "agent_level": "MEDIUM",
  "user_level": "HIGH",
  "context": "User considers caching critical for their performance requirements"
}
```

Inject these calibration entries as additional examples for all subsequent decisions in the same sprint.

### 4c. Trigger Threshold

- In **GUIDED** mode: HIGH and CRITICAL decisions trigger user elicitation.
- In **AUTONOMOUS** mode: ALL decisions are auto-decided and logged with `[AUTO-DECIDED]` marker.

### 4d. Elicitation Format

When presenting a decision to the user, use this format:

```
## Decision Point: [Title]

**What is being decided**: [1-sentence framing]
**Why it matters**: [Impact on downstream implementation]
**Significance**: [CRITICAL/HIGH]

### Options

**Option A: [Name]**
- Approach: [1 sentence]
- Pros: [bullets]
- Cons: [bullets]
- Downstream impact: [constraints in later phases]

**Option B: [Name]**
[same structure]

### Recommendation
[Agent's recommended choice with rationale]

### Your Options
1. **Choose Option [A/B/C]** - proceed with your selection
2. **Choose for me** - agent makes the best call and continues
3. **Choose for me and automate the rest** - agent makes all remaining decisions autonomously
```

### 4e. Autonomy State Machine

```
States: GUIDED | AUTONOMOUS
```

- `thorough` mode starts in **GUIDED**
- `fast` mode starts in **AUTONOMOUS**
- Transitions to AUTONOMOUS when user says "choose for me and automate the rest"
- In AUTONOMOUS: decisions auto-made, logged with `[AUTO-DECIDED]` markers
- User can re-enter GUIDED by saying "stop and ask me"

**Phase integration map**:
| Phase | Steering Status |
|-------|----------------|
| Phase 0 (Intake) | Dormant |
| Phase 1 (Requirements) | Active -- scope, assumptions, constraints |
| Phase 1.5 (UX Design) | Active -- UX scope decisions |
| Phase 2A (Architecture) | Maximally active |
| Phase 2B (Epic Design) | Active but less frequent |
| Phase 3 (Story Decomposition) | Dormant |
| Phase 4 (Story Enrichment) | Dormant |
| Phase 5 (Validation) | Dormant -- final report always shown |

---

## 5. State Management

### 5a. phase-state.json

This is the **single source of truth** for all workflow state. Update it after every phase transition and significant event.

**Required updates**:
- `current_phase`: Set when entering a new phase
- `steering_mode`: Updated on user autonomy transitions
- `significance_calibration`: Appended on user overrides (max 3)
- `decisions_log`: Appended for every decision made
- `refinement_loops`: Incremented on Requirements-Architecture loops
- `ral_passes`: Incremented on `ral` invocations
- `stale_phases`: Phases downstream of a re-run phase are added here
- `epics_count`, `stories_total`, `stories_enriched`: Updated as artifacts are produced
- `validation_status`: Set in Phase 5

### 5b. Stale Phase Tracking

When a phase is re-run (via `--restart-from`, `--continue`, or refinement loop):
1. Mark all downstream phases as stale in `phase-state.json`.
2. Stale phases must be re-run before the workflow can complete.
3. The orchestrator automatically re-runs stale phases in order.

**Phase chain for stale propagation** (in order):
`discovery` → `requirements` → `ux-design` → `architecture` → `epic-design` → `story-decomposition` → `story-enrichment` → `validation`

When `requirements` is marked stale, `ux-design` (if it ran) is also marked stale, and all downstream phases are marked stale.
When `ux-design` is marked stale, `architecture` and all downstream phases are marked stale.

### 5c. OMC State (optional)

If available, use `state_write` and `state_read` with `mode="sprint-plan"` for OMC lifecycle management:
- `active`: Whether a sprint planning session is in progress
- `current_sprint`: Which sprint directory is active
- `session_id`: Current session identifier

If OMC state tools are not available, skip — the workflow functions correctly using `phase-state.json` alone.

**Boundary rule**: `phase-state.json` is NEVER read by OMC hooks. OMC state is NEVER read for phase logic. They are independent.

---

## 6. Progress Reporting

After each phase completes, report to the user:

```
--- Phase {N} Complete: {Phase Name} ---
Duration: {time}
Artifacts: {files written}
Decisions: {count made, count pending}
Mode: {thorough/fast}
Next: Phase {N+1}: {name}
```

If Decision Steering triggered during the phase, present all elicitations BEFORE reporting the phase as complete. Wait for user input on each HIGH/CRITICAL decision in GUIDED mode.

At workflow completion, present a summary:

```
--- Sprint Planning Complete: sprint-{NNN} ---
Mode: {thorough/fast}
Epics: {count}
Stories: {count} ({count} enriched)
Decisions: {count} ({user_count} by user, {auto_count} auto-decided)
Validation: {PASS/FAIL}
Readiness report: .omc/sprint-plan/current/readiness-report.md

Next steps:
- Review stories in .omc/sprint-plan/current/stories/
- Use `/sprint-plan --continue` to resume after clearing context
- Use `ral <phase>` to refine any phase
- Begin implementation with /team ralph
```

---

## 7. Codebase Context Handling

Every project gets a codebase inventory and context sections — there is no brownfield/greenfield split. Phase 0 detects `new_repo: true` only for brand-new repositories (< 10 source files, no build manifests or source directories). This flag is informational.

1. **Phase 0**: Always produce "Existing Codebase Inventory" section in `discovery.md` — project structure, tech stack, existing patterns, module boundaries.
2. **Phase 1**: Always dispatch `explore` (haiku) in parallel to scan for existing implementations (skip only for new repos). Requirements must account for existing features.
3. **Phase 2A**: Architecture decisions must consider existing patterns when they exist — "aligned with existing pattern X" or "diverges from existing pattern X because Y." Divergence requires justification in the ADR.
4. **Phase 4**: Every story file includes a `## Codebase Context` section with existing files to modify, patterns to follow, and integration points. Write "N/A" if no existing codebase context applies.

---

## 8. Error Recovery

### 8a. Phase Idempotency
Any phase can be re-run from scratch. Re-running a phase overwrites its output file. Downstream phases are marked stale in `phase-state.json` and must be re-run.

### 8b. Restart
`--continue` resumes from the next incomplete phase (auto-detected or explicit). `--restart-from=<phase>` re-runs a completed phase and marks downstream as stale. Both verify prerequisite artifacts exist before starting.

### 8c. Agent Failure
If an agent dispatch fails:
1. Retry once with the same prompt.
2. If the retry fails, present the error to the user with options:
   - Retry with different parameters
   - Skip the agent (if optional, e.g., `explore` in Phase 1 for new repos)
   - Abort the workflow

### 8d. Stale Phase Recovery
When phases are marked stale:
1. After the current phase completes, check `stale_phases`.
2. Re-run stale phases in order before proceeding to new phases.
3. Remove phases from `stale_phases` as they complete.

### 8e. Session Resume
On resume (workflow was interrupted):
1. Read OMC state to find the active sprint.
2. Read `phase-state.json` to determine last completed phase.
3. Check for stale phases.
4. Resume from the next incomplete phase.

---

## 9. The `ral` Command

The `ral` command triggers RALPLAN-DR refinement on any phase output. This is handled by the separate `/ral` skill, but the orchestrator must support it:

- Track `ral_passes` per phase in `phase-state.json`
- Max 2 `ral` invocations per phase (respond with diminishing returns message after 2; `--force` overrides)
- Each `ral` pass: Planner reviews -> Architect challenges -> Critic validates
- On consensus: apply changes to the phase artifact
- On no consensus after 3 iterations: present disagreements via Decision Steering
- Mark downstream phases as stale after a `ral` pass modifies an artifact
