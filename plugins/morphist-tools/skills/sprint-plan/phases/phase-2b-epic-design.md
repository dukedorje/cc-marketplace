# Phase 2B: Epic Design

## Purpose

Group requirements into user-value-focused epics with FR coverage maps. Epics are the primary unit of scope and sequencing — they determine what gets built and in what order. Architecture decisions from Phase 2A directly constrain epic boundaries.

---

## Agent Flow

### Thorough Mode (default for sprint 1)

RALPLAN-DR consensus: planner proposes, architect reviews, critic validates. Max 3 iterations until consensus.

| Step | Agent | Model | Role |
|------|-------|-------|------|
| 1 | planner | opus | Propose epic structure with FR coverage map |
| 2 | architect | opus | Verify epic boundaries respect architecture decisions |
| 3 | critic | opus | Validate FR coverage, dependencies, and epic quality |

If no consensus after 3 iterations, present disagreements to the user via Decision Steering.

### Fast Mode (sprint 2+ default)

Planner runs once. No architect or critic review.

---

## Inputs

- `current/requirements.md` (Phase 1 output)
- `current/architecture-decisions.md` (Phase 2A output)
- `current/discovery.md` (Phase 0 output)

---

## Planner Agent Prompt

Design epic structure following these principles:

### Epic Design Principles

1. **User-value-focused grouping**: Group by user capability, NOT technical layers. "User Authentication" not "Database Layer". Each epic title should complete the sentence "After this epic, users can..."
2. **Standalone epics**: Each epic must be independently deployable and enable future epics without requiring them.
3. **No forward dependencies**: Epic N must never depend on Epic N+1. Dependencies flow forward only (later epics may depend on earlier ones).
4. **FR coverage**: Every FR from `requirements.md` must map to exactly one epic. No orphaned FRs, no duplicate assignments.
5. **Epic ordering**: Foundation epics first (auth, data model, core infrastructure), feature epics after.
6. **Right-sized epics**: Target 3-6 FRs per epic. Fewer than 2 suggests merging; more than 6 suggests splitting.

### Required Output Per Epic

For each epic, the planner must provide:
- **Title**: User-value-focused name
- **Goal statement**: Completes "After this epic, users can..." or "After this epic, the system can..."
- **FR assignments**: Which FRs this epic implements
- **NFR applicability**: Which NFRs apply to this epic
- **Architecture constraints**: Which decisions from `architecture-decisions.md` constrain this epic
- **Dependencies**: Which earlier epics this one depends on, and which later epics it enables
- **Estimated complexity**: Low / Medium / High (based on FR count, architecture constraints, and integration surface)

---

## Architect Agent Prompt (thorough mode only)

Review the proposed epic structure for:

1. **Architecture alignment**: Do epic boundaries respect the architecture decisions? For example, if the architecture specifies a specific auth provider, is all auth-related work in a single epic (or explicitly split with justification)?
2. **Undecided patterns**: Does any epic require an architectural pattern that was not decided in Phase 2A? If so, flag it — Phase 2A may need a supplemental decision.
3. **Epic ordering**: Does the proposed ordering respect technical dependencies? Can Epic 1 actually be built without anything from Epic 2+?
4. **Foundation coverage**: Is infrastructure and setup work (database schema, auth scaffolding, CI/CD) placed in early epics where it belongs?
5. **Integration surfaces**: Are integration points between epics clearly identified? Are they minimal and well-defined?

For each epic, either:
- APPROVE with optional notes
- MODIFY with specific boundary changes and rationale
- FLAG with concerns requiring planner revision

---

## Critic Agent Prompt (thorough mode only)

Validate the epic structure for completeness and quality:

### Coverage Validation
- **Every FR must appear in exactly one epic.** List any orphaned FRs (not assigned to any epic) or duplicated FRs (assigned to multiple epics).
- **Every NFR must be noted** as applicable to at least one epic (most NFRs apply broadly).

### Size Validation
- Flag epics with more than 5-6 FRs — suggests the epic should be split into smaller, more focused units.
- Flag epics with only 1 FR — suggests the epic should be merged with a related epic.

### Dependency Validation
- Check for hidden dependencies between epics that the planner did not declare.
- Verify no circular dependencies exist.
- Confirm that the first epic has zero dependencies.

### Quality Validation
- Verify epic goals are user-value statements, not technical descriptions. "Users can register and log in" is good. "Set up authentication middleware" is bad.
- Check that estimated complexity is justified (a "Low" epic with 6 FRs and 3 architecture constraints should be questioned).

For each issue found, provide: [CRITIC: ISSUE — {description with specific fix suggestion}].

---

## Decision Steering Integration

Decision Steering is **active but less frequent** than Phase 2A. Most architecture decisions are already made.

Decisions that may trigger user elicitation:
- Epic boundary disputes (where to draw the line between epics)
- Epic ordering conflicts (when two orderings are equally valid with different tradeoffs)
- Scope decisions (whether a requirement belongs in this sprint or should be deferred)

Use the same elicitation format as Phase 2A.

---

## Output Schema: `current/epics.md`

```markdown
---
project: [name]
sprint: sprint-[NNN]
inputDocuments: [requirements.md, architecture-decisions.md]
consensus_iterations: [number]
---

## Requirements Inventory

### Functional Requirements
[List all FRs from requirements.md with their IDs and titles]

### Non-Functional Requirements
[List all NFRs from requirements.md]

### Architecture Requirements
[Requirements derived from architecture decisions — e.g., "Must use PostgreSQL (D-002)", "Must implement OAuth2 flow (D-003)"]

## FR Coverage Map

| FR | Epic | Story (filled in Phase 3) |
|----|------|---------------------------|
| FR1 | Epic 1 | - |
| FR2 | Epic 1 | - |
| FR3 | Epic 2 | - |

## Epic List

| # | Title | Goal | FRs | Estimated Stories | Complexity |
|---|-------|------|-----|-------------------|------------|
| 1 | [Title] | [User-value goal] | FR1, FR2 | 3-4 | Medium |

---

## Epic 1: [Title]

### Goal
[User-value statement: "Users can..." or "The system can..."]

### Assigned Requirements
- FR1: [title]
- FR2: [title]
- NFR1: [if applicable]

### Architecture Constraints
[Relevant decisions from architecture-decisions.md, referenced by ID]
- D-001: [title] — [how it constrains this epic]
- D-003: [title] — [how it constrains this epic]

### Dependencies
- Depends on: [none for first epic]
- Enables: [Epic 2, Epic 3]

### Estimated Complexity
[Low | Medium | High] — [brief justification]

### Stories (populated in Phase 3)
[Placeholder — filled by Phase 3: Story Decomposition]

---

## Epic 2: [Title]
[Same structure as Epic 1]

---

## Epic Health Metrics (populated in Phase 3)
[Placeholder — filled by Phase 3 with story counts, complexity estimates, dependency flags per epic]
```

---

## Agent Dispatch

### Step 1: Planner (always runs)

```python
Agent(
    subagent_type="oh-my-claudecode:planner",
    model="opus",
    prompt="""
You are running Phase 2B: Epic Design for the sprint-plan workflow.

Your role: PLANNER — design the epic structure.

## Inputs

Read the following files:
- SPRINT_DIR/requirements.md (Phase 1 output — all FRs, NFRs, constraints)
- SPRINT_DIR/architecture-decisions.md (Phase 2A output — all architecture decisions)
- SPRINT_DIR/discovery.md (Phase 0 output — project context)

## Task

Design the epic structure for this sprint following these principles:

1. **User-value-focused grouping**: Group by user capability, NOT technical layers. Each epic title should complete "After this epic, users can..."
2. **Standalone epics**: Each epic independently deployable and enabling future epics.
3. **No forward dependencies**: Epic N never depends on Epic N+1. Dependencies flow forward only.
4. **Complete FR coverage**: Every FR maps to exactly one epic. No orphans, no duplicates.
5. **Foundation first**: Auth, data model, and core infrastructure in early epics.
6. **Right-sized**: Target 3-6 FRs per epic.

For each epic provide:
- Title (user-value-focused)
- Goal statement ("After this epic, users can..." or "After this epic, the system can...")
- FR assignments (by ID)
- NFR applicability
- Architecture constraints (by decision ID from architecture-decisions.md)
- Dependencies (which earlier epics, if any)
- Estimated complexity (Low/Medium/High with justification)

## Output

Write the complete epic structure to SPRINT_DIR/epics.md using the output schema from the phase instructions.

Include:
1. Requirements Inventory (all FRs, NFRs, architecture requirements)
2. FR Coverage Map (every FR mapped to an epic)
3. Epic List summary table
4. Detailed epic sections with all required fields
5. Placeholder sections for Stories and Epic Health Metrics (filled by Phase 3)
""",
)
```

### Step 2: Architect Review (thorough mode only)

```python
Agent(
    subagent_type="oh-my-claudecode:architect",
    model="opus",
    prompt="""
You are running Phase 2B: Epic Design for the sprint-plan workflow.

Your role: ARCHITECT — review the proposed epic structure.

## Inputs

Read the following files:
- SPRINT_DIR/epics.md (planner's proposed epic structure)
- SPRINT_DIR/architecture-decisions.md (Phase 2A output)
- SPRINT_DIR/requirements.md (Phase 1 output)

## Task

Review the proposed epic structure for:

1. **Architecture alignment**: Do epic boundaries respect architecture decisions? Is related work grouped appropriately?
2. **Undecided patterns**: Does any epic require an architectural pattern not decided in Phase 2A? Flag for supplemental decision.
3. **Epic ordering**: Does ordering respect technical dependencies? Can Epic 1 be built standalone?
4. **Foundation coverage**: Is infrastructure/setup work in early epics?
5. **Integration surfaces**: Are integration points between epics minimal and well-defined?

For each epic, mark:
- [ARCHITECT: APPROVED] with optional notes
- [ARCHITECT: MODIFY — {specific boundary changes and rationale}]
- [ARCHITECT: FLAG — {concern requiring planner revision}]

## Output

Update SPRINT_DIR/epics.md with your review marks and any proposed modifications.
""",
)
```

### Step 3: Critic Validation (thorough mode only)

```python
Agent(
    subagent_type="oh-my-claudecode:critic",
    model="opus",
    prompt="""
You are running Phase 2B: Epic Design for the sprint-plan workflow.

Your role: CRITIC — validate FR coverage, dependencies, and epic quality.

## Inputs

Read the following files:
- SPRINT_DIR/epics.md (with architect's review marks)
- SPRINT_DIR/requirements.md (Phase 1 output)
- SPRINT_DIR/architecture-decisions.md (Phase 2A output)

## Task

### Coverage Validation
- Verify every FR appears in exactly one epic. List any orphaned or duplicated FRs.
- Verify every NFR is noted as applicable to at least one epic.

### Size Validation
- Flag epics with >5-6 FRs (suggest splitting).
- Flag epics with only 1 FR (suggest merging).

### Dependency Validation
- Check for hidden dependencies not declared by the planner.
- Verify no circular dependencies.
- Confirm Epic 1 has zero dependencies.

### Quality Validation
- Verify epic goals are user-value statements, not technical descriptions.
- Check that complexity estimates are justified.

## Output

Update SPRINT_DIR/epics.md with validation results.

For each issue: [CRITIC: ISSUE — {description with specific fix suggestion}].
If all valid: [CRITIC: VALIDATED].

Add summary: ## Consensus: REACHED (iteration {N}) or ## Consensus: NOT REACHED — issues listed above.
""",
)
```

---

## Phase Completion

Phase 2B is complete when:

1. `current/epics.md` exists with all required sections
2. FR Coverage Map shows every FR assigned to exactly one epic
3. Consensus reached (thorough mode) or single pass complete (fast mode)
4. No orphaned or duplicated FRs
5. No circular dependencies between epics

Update `current/phase-state.json`:
```json
{
  "current_phase": "story-decomposition",
  "epics_count": 4
}
```

Proceed to Phase 3: Story Decomposition.
