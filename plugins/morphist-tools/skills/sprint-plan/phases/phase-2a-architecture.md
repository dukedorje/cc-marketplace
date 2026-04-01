# Phase 2A: Architecture Decisions

## Purpose

Make critical architecture decisions that constrain all downstream work. This is the most consequential phase — decisions made here ripple through every epic, story, and implementation detail. Decision Steering is maximally active.

---

## Agent Flow

### Thorough Mode (default for sprint 1)

RALPLAN-DR consensus: planner proposes, architect reviews, critic validates. Max 3 iterations until consensus.

| Step | Agent | Model | Role |
|------|-------|-------|------|
| 1 | planner | opus | Propose architecture decisions by category |
| 2 | architect | opus | Review for soundness, completeness, alternatives |
| 3 | critic | opus | Validate significance levels, consequences, consistency |

If no consensus after 3 iterations, present disagreements to the user via Decision Steering.

### Fast Mode (sprint 2+ default)

Planner runs once. No architect or critic review. Conflicts noted in output for user to address.

---

## Inputs

- `current/requirements.md` (Phase 1 output)
- `current/discovery.md` (Phase 0 output)
- `decisions/active-decisions.md` (sprint N>1 — non-superseded decisions from prior sprints)

---

## Decision Categories

The planner must address each applicable category:

1. **Data Architecture** — database selection, data modeling approach, caching strategy
2. **Authentication and Security** — auth provider, session management, authorization model
3. **API and Communication** — REST vs GraphQL, real-time strategy, API versioning
4. **Frontend Architecture** (if applicable) — framework, state management, routing
5. **Infrastructure and Deployment** — hosting, CI/CD, containerization, monitoring

---

## Planner Agent Prompt

The planner proposes architecture decisions structured by category. For each category, the planner must output decisions in this format:

```json
{
  "category": "Data Architecture",
  "decisions": [
    {
      "id": "D-{NNN}",
      "title": "Primary Database Selection",
      "significance": "CRITICAL",
      "rationale": "Constrains all data access patterns across every epic",
      "options": [
        {
          "name": "PostgreSQL",
          "approach": "Relational database with strong ACID guarantees",
          "pros": ["..."],
          "cons": ["..."],
          "downstream_impact": "All data access via SQL; ORM required"
        }
      ],
      "recommendation": "...",
      "recommendation_rationale": "..."
    }
  ]
}
```

### Significance Calibration Examples

Include these examples in the planner prompt to anchor classification:

| Example Decision | Level | Why |
|-----------------|-------|-----|
| "Use PostgreSQL vs MongoDB for primary data store" | CRITICAL | Constrains all data access patterns across every epic |
| "REST vs GraphQL for API layer" | HIGH | Constrains frontend data fetching and all story task structures |
| "Use Zod vs Joi for input validation" | MEDIUM | Library swap within same category; limited blast radius |
| "Name the config file `config.yaml` vs `settings.json`" | LOW | Cosmetic; zero downstream impact |

### Self-Calibration

The first 3 user overrides (user escalates LOW to HIGH or de-escalates HIGH to MEDIUM) are stored in `phase-state.json` under `significance_calibration` and injected as additional calibration examples for subsequent decisions in the same sprint.

---

## Architect Agent Prompt (thorough mode only)

Review each proposed decision for:

- **Technical soundness**: Will this actually work for the project's requirements?
- **Constraint completeness**: What downstream effects are missing from the planner's analysis?
- **Alternative analysis**: Were good alternatives overlooked or dismissed too quickly?
- **Existing pattern alignment**: Does this align with existing patterns in the codebase? If it diverges, is the justification compelling? (Skip if new repo with no existing code.)

Challenge weak rationales with steelman antithesis — argue the strongest case for the best alternative before accepting the recommendation.

May propose modifications to decisions or suggest alternative decisions entirely.

---

## Critic Agent Prompt (thorough mode only)

Validate that each decision:

- Has a clear, justified significance level (not inflated, not understated)
- Lists real consequences specific to this project (not generic boilerplate)
- Considers the project's specific context, constraints, and requirements (not generic advice)
- Does not contradict other decisions in the set

Additionally flag:

- Any decisions that should be escalated to the user but were not (under-classified significance)
- Any decisions classified too high, risking approval fatigue
- Any missing decisions — categories or concerns from `requirements.md` that have no corresponding architecture decision

---

## ADR-lite Format

Each decision is recorded in this format in the output:

```markdown
## D-{NNN}: {Title}
**Date**: {date}
**Sprint**: sprint-{NNN}
**Significance**: CRITICAL | HIGH | MEDIUM
**Decided by**: user | auto-decided
**Status**: accepted | superseded by D-{NNN}

**Context**: [why this decision came up — trace back to specific FRs/NFRs/constraints]
**Decision**: [what was chosen]
**Alternatives**: [what was rejected and why]
**Consequences**: [what this constrains downstream — be specific about which epics/stories are affected]
```

---

## Requirements-Architecture Refinement Loop

After architecture decisions are made, check whether any decision invalidates, constrains, or creates new requirements not captured in `requirements.md`.

**Flow**:
1. Phase 1 produces `requirements.md`
2. Phase 2A runs RALPLAN-DR on architecture decisions
3. If architecture decisions create new requirements or invalidate existing ones:
   - Architect flags the conflict with specifics (which decision, which requirement, what changed)
   - Loop back to Phase 1 with the new constraints
   - Phase 1 re-runs incrementally (only processes new/changed requirements)
   - Phase 2A re-runs with updated requirements
4. Max 3 loop iterations
5. After max: unresolved conflicts become Decision Steering elicitation points presented to the user

**Example**: "FR7 says real-time updates" -> architecture decides WebSockets -> "that means FR3 needs a different auth model" -> requirement updated -> architecture adjusted.

**In fast mode**: The loop is disabled. Phase 1 runs once, Phase 2A runs once. Conflicts are noted in `architecture-decisions.md` for the user to address.

**State tracking**: Increment `refinement_loops.requirements_architecture.count` in `phase-state.json` after each loop iteration.

---

## Decision Steering Integration

Decision Steering is **maximally active** in this phase.

### Behavior by Steering Mode

**GUIDED mode** (thorough default):
- All HIGH and CRITICAL decisions are presented to the user for approval
- MEDIUM decisions are auto-decided and logged
- LOW decisions are auto-decided and logged

**AUTONOMOUS mode** (fast default):
- All decisions are auto-decided
- All decisions logged with `[AUTO-DECIDED]` markers for later review

### Self-Calibration Trigger

Store the first 3 user overrides as calibration examples in `phase-state.json`:

```json
{
  "significance_calibration": [
    {"decision": "Cache strategy", "agent_level": "MEDIUM", "user_level": "HIGH", "context": "User considers cache critical for performance SLA"}
  ]
}
```

Inject these as additional calibration examples for all subsequent decisions in the same sprint.

### Autonomy Transition

If the user responds "Choose for me and automate the rest" to any elicitation, transition `steering_mode` to `AUTONOMOUS` in `phase-state.json`. All remaining decisions in this phase (and subsequent phases) are auto-decided and logged.

### Elicitation Format

Present each HIGH/CRITICAL decision to the user using this format:

```markdown
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

---

## Output

### Primary Output: `current/architecture-decisions.md`

Contains the full ADR-lite records for all decisions made during this sprint.

### Secondary Outputs

- **Append** entries to `decisions/decision-log.md` (chronological index across all sprints)
- **Update** `decisions/active-decisions.md` (compacted view of non-superseded decisions)
- **Update** `current/phase-state.json` with `decisions_log` entries

---

## Agent Dispatch

### Step 1: Planner (always runs)

```python
Agent(
    subagent_type="oh-my-claudecode:planner",
    model="opus",
    prompt="""
You are running Phase 2A: Architecture Decisions for the sprint-plan workflow.

Your role: PLANNER — propose architecture decisions.

## Inputs

Read the following files:
- SPRINT_DIR/requirements.md (Phase 1 output)
- SPRINT_DIR/discovery.md (Phase 0 output)
- .omc/sprint-plan/decisions/active-decisions.md (if it exists — prior sprint decisions)

## Task

Propose architecture decisions for each applicable category:
1. Data Architecture (database selection, data modeling approach, caching strategy)
2. Authentication and Security (auth provider, session management, authorization model)
3. API and Communication (REST vs GraphQL, real-time strategy, API versioning)
4. Frontend Architecture (if applicable: framework, state management, routing)
5. Infrastructure and Deployment (hosting, CI/CD, containerization, monitoring)

For each decision, output the structured JSON format specified in the phase instructions, including:
- Decision ID (D-{NNN}, continuing from the last ID in active-decisions.md or starting at D-001)
- Title, significance level, rationale for the significance level
- All viable options with pros, cons, and downstream impact
- Your recommendation with rationale

## Significance Calibration

Use these anchors to calibrate your significance classifications:

| Example Decision | Level | Why |
|-----------------|-------|-----|
| "Use PostgreSQL vs MongoDB for primary data store" | CRITICAL | Constrains all data access patterns across every epic |
| "REST vs GraphQL for API layer" | HIGH | Constrains frontend data fetching and all story task structures |
| "Use Zod vs Joi for input validation" | MEDIUM | Library swap within same category; limited blast radius |
| "Name the config file config.yaml vs settings.json" | LOW | Cosmetic; zero downstream impact |

{significance_calibration_examples}

## Existing Pattern Alignment

If the project has an existing codebase, for each decision explicitly state:
- "Aligned with existing pattern: [X]" OR
- "Diverges from existing pattern: [X] because [Y]"

Divergence requires a compelling justification. For new repos, skip this section.

## Output

Write your proposed decisions to SPRINT_DIR/architecture-decisions.md using the ADR-lite format.
Also append entries to .omc/sprint-plan/decisions/decision-log.md and update .omc/sprint-plan/decisions/active-decisions.md.

After writing, check: do any decisions invalidate, constrain, or create new requirements not in requirements.md?
If yes, list the conflicts at the end of the file under ## Requirements Conflicts.
""",
)
```

### Step 2: Architect Review (thorough mode only)

```python
Agent(
    subagent_type="oh-my-claudecode:architect",
    model="opus",
    prompt="""
You are running Phase 2A: Architecture Decisions for the sprint-plan workflow.

Your role: ARCHITECT — review proposed architecture decisions.

## Inputs

Read the following files:
- SPRINT_DIR/architecture-decisions.md (planner's proposed decisions)
- SPRINT_DIR/requirements.md (Phase 1 output)
- SPRINT_DIR/discovery.md (Phase 0 output)

## Task

Review each proposed decision for:

1. **Technical soundness**: Will this actually work for the project's requirements? Are there known failure modes?
2. **Constraint completeness**: What downstream effects did the planner miss?
3. **Alternative analysis**: Were good alternatives overlooked or dismissed too quickly?
4. **Existing pattern alignment**: Does this align with existing patterns in the codebase? Is any divergence justified? (Skip if new repo.)

For each decision, either:
- APPROVE with optional notes
- MODIFY with specific changes and rationale
- REJECT with alternative proposal and rationale

Challenge weak rationales with steelman antithesis: argue the strongest possible case for the best alternative before accepting the recommendation.

## Output

Write your review to SPRINT_DIR/architecture-decisions.md, updating each decision with your assessment.
Flag any requirements conflicts not caught by the planner.

Mark each decision: [ARCHITECT: APPROVED], [ARCHITECT: MODIFIED], or [ARCHITECT: REJECTED with alternative].
""",
)
```

### Step 3: Critic Validation (thorough mode only)

```python
Agent(
    subagent_type="oh-my-claudecode:critic",
    model="opus",
    prompt="""
You are running Phase 2A: Architecture Decisions for the sprint-plan workflow.

Your role: CRITIC — validate the quality and consistency of architecture decisions.

## Inputs

Read the following files:
- SPRINT_DIR/architecture-decisions.md (with architect's review marks)
- SPRINT_DIR/requirements.md (Phase 1 output)

## Task

Validate that each decision:

1. Has a clear, justified significance level — not inflated (approval fatigue risk) or understated (missed user input)
2. Lists real consequences specific to THIS project, not generic boilerplate
3. Considers the project's specific context, constraints, and requirements
4. Does not contradict other decisions in the set
5. Has complete alternative analysis (rejected options have real reasons, not strawmen)

Additionally flag:
- Decisions that should be escalated to the user but were not (significance too low)
- Decisions classified too high (approval fatigue risk)
- Missing decisions: requirements or constraints from requirements.md with no corresponding architecture decision
- Contradictions between any two decisions in the set

## Output

Write your validation results to SPRINT_DIR/architecture-decisions.md.

For each decision, add: [CRITIC: VALIDATED] or [CRITIC: ISSUE — {description}].

If all decisions validated, add a summary: ## Consensus: REACHED (iteration {N}).
If issues remain, add: ## Consensus: NOT REACHED — issues listed above.
""",
)
```

---

## Requirements-Architecture Loop Dispatch

After the RALPLAN-DR pass (or single planner pass in fast mode), check `architecture-decisions.md` for a `## Requirements Conflicts` section.

If conflicts exist and `refinement_loops.requirements_architecture.count < 3`:

1. Increment `refinement_loops.requirements_architecture.count` in `phase-state.json`
2. Re-run Phase 1 incrementally with the new constraints
3. Re-run Phase 2A with updated requirements
4. Repeat until no conflicts or max iterations reached

If max iterations reached with unresolved conflicts:
- Present each conflict to the user via Decision Steering elicitation
- Record user decisions in `phase-state.json`

---

## Phase Completion

Phase 2A is complete when:

1. `current/architecture-decisions.md` exists with all decisions in ADR-lite format
2. Consensus reached (thorough mode) or single pass complete (fast mode)
3. No unresolved requirements conflicts (or max loop iterations reached and user resolved them)
4. `decisions/decision-log.md` updated with new entries
5. `decisions/active-decisions.md` updated with non-superseded decisions

Update `current/phase-state.json`:
```json
{
  "current_phase": "epic-design",
  "decisions_log": [
    {"id": "D-001", "title": "...", "chosen": "...", "decided_by": "user|auto"}
  ]
}
```

Proceed to Phase 2B: Epic Design.
