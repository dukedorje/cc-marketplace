# Phase 2B: Epic Design

## Purpose

Group requirements into user-value-focused epics with FR coverage maps. Epics are the primary unit of scope and sequencing — they determine what gets built and in what order. Architecture decisions from Phase 2A directly constrain epic boundaries.

---

## Agent Flow

Single planner/opus call. Architect and critic validation concerns are embedded as inline checklist rules in the planner prompt.

| Step | Agent | Model | Role |
|------|-------|-------|------|
| 1 | planner | opus | Design epic structure with built-in architecture and coverage validation |

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

### Validation Checklist (self-verify before writing output)

Before finalizing the epic structure, verify each item. Flag any violations inline in the output.

**Coverage rules** (from FR coverage audit):
- [ ] Every FR from `requirements.md` is assigned to exactly one epic — no orphans, no duplicates
- [ ] Every NFR is noted as applicable to at least one epic

**Size rules** (from epic size validation):
- [ ] Each epic has 2–6 stories estimated — flag with `[SIZE WARNING: too large — consider splitting]` if >6, or `[SIZE WARNING: too small — consider merging]` if <2

**Dependency rules** (from dependency validation):
- [ ] No circular dependencies between epics
- [ ] Epic 1 has zero dependencies on any other epic
- [ ] No hidden cross-epic dependencies left undeclared

**Architecture rules** (from architecture alignment):
- [ ] Epic boundaries respect all decisions in `architecture-decisions.md` — related work is grouped appropriately
- [ ] Any epic that requires an architectural pattern not decided in Phase 2A is flagged with `[ARCH GAP: {description}]`
- [ ] Infrastructure and setup work (database schema, auth scaffolding, CI/CD) is placed in early epics

**Quality rules** (from quality validation):
- [ ] Epic goals are user-value statements, not technical descriptions ("Users can register and log in" is good; "Set up authentication middleware" is bad)
- [ ] Complexity estimates are justified — a "Low" epic with many FRs and architecture constraints should be questioned
- [ ] Earlier epics establish foundations that later epics build on; sequencing is logical

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

### Step 1: Planner (single call)

```python
Agent(
    subagent_type="oh-my-claudecode:planner",
    model="opus",
    prompt="""
You are running Phase 2B: Epic Design for the sprint-plan workflow.

Your role: PLANNER — design the epic structure and self-validate it.

## Inputs

Read the following files:
- SPEC_DIR/requirements.md (Phase 1 output — all FRs, NFRs, constraints)
- SPEC_DIR/architecture-decisions.md (Phase 2A output — all architecture decisions)
- SPEC_DIR/discovery.md (Phase 0 output — project context)

## Task

Design the epic structure for this sprint following these principles:

1. **User-value-focused grouping**: Group by user capability, NOT technical layers. Each epic title should complete "After this epic, users can..."
2. **Standalone epics**: Each epic independently deployable and enabling future epics.
3. **No forward dependencies**: Epic N never depends on Epic N+1. Dependencies flow forward only.
4. **Complete FR coverage**: Every FR maps to exactly one epic. No orphans, no duplicates.
5. **Foundation first**: Auth, data model, and core infrastructure in early epics.
6. **Right-sized**: Target 2-6 stories per epic.
7. **Independently valuable**: Each epic should deliver standalone user or system value where possible.
8. **Foundation before features**: Earlier epics establish foundations that later epics build on.

For each epic provide:
- Title (user-value-focused)
- Goal statement ("After this epic, users can..." or "After this epic, the system can...")
- FR assignments (by ID)
- NFR applicability
- Architecture constraints (by decision ID from architecture-decisions.md)
- Dependencies (which earlier epics, if any)
- Estimated complexity (Low/Medium/High with justification)

## Validation Checklist

After drafting the epic structure, self-verify each item below. Flag any violations inline.

**Coverage**:
- Every FR from requirements.md is assigned to exactly one epic (no orphans, no duplicates)
- Every NFR is noted as applicable to at least one epic

**Size** — flag inline if violated:
- Each epic has 2–6 stories estimated: flag `[SIZE WARNING: too large — consider splitting]` if >6, `[SIZE WARNING: too small — consider merging]` if <2

**Dependencies**:
- No circular dependencies between epics
- Epic 1 has zero dependencies
- No hidden cross-epic dependencies left undeclared

**Architecture alignment**:
- Epic boundaries respect all decisions in architecture-decisions.md
- Any epic requiring an undecided architectural pattern is flagged `[ARCH GAP: {description}]`
- Infrastructure and setup work is in early epics

**Quality**:
- Epic goals are user-value statements, not technical descriptions
- Complexity estimates are justified by FR count and architecture constraints

## Output

Write the complete epic structure to SPEC_DIR/epics.md using the output schema from the phase instructions.

Include:
1. Requirements Inventory (all FRs, NFRs, architecture requirements)
2. FR Coverage Map (every FR mapped to an epic)
3. Epic List summary table
4. Detailed epic sections with all required fields
5. Placeholder sections for Stories and Epic Health Metrics (filled by Phase 3)
""",
)
```

---

## Phase Completion

Phase 2B is complete when:

1. `current/epics.md` exists with all required sections
2. FR Coverage Map shows every FR assigned to exactly one epic
3. Single planner pass complete with no unresolved validation flags
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
