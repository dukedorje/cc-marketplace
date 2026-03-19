# Decision Steering System Reference

This document defines the complete Decision Steering system used across all phases of the sprint planning workflow.

---

## Component 1: Significance Classifier

### Levels

| Level | Definition | Examples |
|-------|-----------|----------|
| **CRITICAL** | Core value proposition, technology constraining entire project, security model, data model affecting all features | Primary database selection, core data model design |
| **HIGH** | Architecture patterns, API design approach, authentication method | REST vs GraphQL, auth provider, state management |
| **MEDIUM** | Library selection within category, testing strategy, deployment target | Zod vs Joi, Jest vs Vitest, AWS vs GCP |
| **LOW** | Naming conventions, file organization, formatting | Config file naming, directory structure preferences |

### Classification Mechanism

Each agent producing a decision returns structured output:

```json
{
  "decision": "Description of what is being decided",
  "significance": "HIGH",
  "rationale": "This constrains all API consumers and is hard to change after Epic 1 ships",
  "options": [
    {
      "name": "Option A",
      "approach": "Brief description",
      "pros": ["..."],
      "cons": ["..."],
      "downstream_impact": "What this constrains in later phases"
    }
  ],
  "recommendation": "Option A",
  "recommendation_rationale": "Why this is the best choice"
}
```

The `significance` and `rationale` fields are mandatory. The rationale must explain WHY this level, not restate the level.

### Calibration Examples

| Example Decision | Level | Why |
|-----------------|-------|-----|
| "Use PostgreSQL vs MongoDB for primary data store" | CRITICAL | Constrains all data access patterns across every epic |
| "REST vs GraphQL for API layer" | HIGH | Constrains frontend data fetching and all story task structures |
| "Use Zod vs Joi for input validation" | MEDIUM | Library swap within same category; limited blast radius |
| "Name the config file `config.yaml` vs `settings.json`" | LOW | Cosmetic; zero downstream impact |

### Self-Calibration

The first 3 user overrides (user escalates LOW->HIGH or de-escalates HIGH->MEDIUM) are stored in `phase-state.json` as `significance_calibration`:

```json
{
  "significance_calibration": [
    {
      "decision": "Caching strategy",
      "agent_level": "MEDIUM",
      "user_level": "HIGH",
      "context": "User indicated caching is critical for their scale requirements"
    }
  ]
}
```

These overrides are injected as additional calibration examples for subsequent decisions in the same sprint.

### Trigger Threshold

- **GUIDED mode**: HIGH and CRITICAL decisions trigger user elicitation
- **AUTONOMOUS mode**: All decisions auto-decided but logged with `[AUTO-DECIDED]` markers

---

## Component 2: Elicitation Format

When a decision triggers user elicitation, present it in this format:

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
- Approach: [1 sentence]
- Pros: [bullets]
- Cons: [bullets]
- Downstream impact: [constraints in later phases]

### Recommendation
[Agent's recommended choice with rationale]

### Your Options
1. **Choose Option [A/B/C]** - proceed with your selection
2. **Choose for me** - agent makes the best call and continues
3. **Choose for me and automate the rest** - agent makes all remaining decisions autonomously
```

### Elicitation Rules

- Present at most 3-4 options (more creates decision fatigue)
- Always include a recommendation with rationale
- Always include the "choose for me" escape hatch
- Group related decisions when they occur together (present as a single elicitation)
- Never present LOW or MEDIUM decisions in GUIDED mode

---

## Component 3: Autonomy State Machine

### States

```
GUIDED <---> AUTONOMOUS
```

- `thorough` mode starts in **GUIDED**
- `fast` mode starts in **AUTONOMOUS**

### Transitions

| From | To | Trigger |
|------|----|---------|
| GUIDED | AUTONOMOUS | User says "choose for me and automate the rest" |
| AUTONOMOUS | GUIDED | User says "stop and ask me" |

### Behavior per State

**GUIDED**:
- HIGH and CRITICAL decisions presented to user with options
- MEDIUM and LOW decisions auto-decided and logged
- User can escalate/de-escalate significance (feeds self-calibration)

**AUTONOMOUS**:
- ALL decisions auto-decided
- All decisions logged with `[AUTO-DECIDED]` marker
- User can review all auto-decisions in `architecture-decisions.md`
- User can re-enter GUIDED at any time

### Phase Integration Map

| Phase | Steering Status | Notes |
|-------|----------------|-------|
| Phase 0 (Intake) | **Dormant** | No decisions to make |
| Phase 1 (Requirements) | **Active** | Scope, assumptions, constraints |
| Phase 2A (Architecture) | **Maximally Active** | All architecture decisions flow through steering |
| Phase 2B (Epic Design) | **Active** | Epic boundaries, ordering decisions |
| Phase 3 (Story Decomposition) | **Dormant** | Decisions locked from Phase 2 |
| Phase 4 (Story Enrichment) | **Dormant** | Decisions locked from Phase 2 |
| Phase 5 (Validation) | **Dormant** | Final report always shown regardless |

---

## Recording Decisions

Every decision (user-made or auto-decided) is recorded in ADR-lite format:

```markdown
## D-{NNN}: {Title}
**Date**: {date}
**Sprint**: sprint-{NNN}
**Significance**: CRITICAL | HIGH | MEDIUM
**Decided by**: user | auto-decided
**Status**: accepted | superseded by D-{NNN}

**Context**: [why this came up]
**Decision**: [what was chosen]
**Alternatives**: [what was rejected and why]
**Consequences**: [what this constrains downstream]
```

Decisions are written to:
1. `current/architecture-decisions.md` (sprint-specific)
2. `decisions/decision-log.md` (cross-sprint chronological)
3. `decisions/active-decisions.md` (compacted, non-superseded only)

---

## State Storage

Decision Steering state lives in `phase-state.json`:

```json
{
  "steering_mode": "GUIDED",
  "significance_calibration": [],
  "decisions_log": [
    {
      "id": "D-001",
      "title": "Database Selection",
      "significance": "CRITICAL",
      "chosen": "PostgreSQL",
      "decided_by": "user"
    }
  ]
}
```
