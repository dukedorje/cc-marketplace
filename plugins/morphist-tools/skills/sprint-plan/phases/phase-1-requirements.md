# Phase 1: Requirements Expansion

## Purpose

Transform a product idea into structured, testable requirements: functional requirements (FRs), non-functional requirements (NFRs), technical constraints (TCs), open questions, scope boundaries, and assumptions.

When Phase 0 detected `input_quality: raw-idea`, the analyst agent expands its scope to also produce user personas, success metrics, and risk assessment — effectively producing a PRD-quality output from minimal input.

Decision Steering is active in this phase. After the merge, significant decisions (HIGH/CRITICAL) are presented to the user in GUIDED mode, or auto-decided and logged in AUTONOMOUS mode.

---

## Agents

Three agents run in parallel:

| Agent | Model | Condition |
|---|---|---|
| `analyst` | opus | Always |
| `architect` | opus | Always |
| `explore` | haiku | Always (skip for new repos with no source files) |

---

## Inputs

- `current/discovery.md` (Phase 0 output — only artifact loaded; no prior agent memory)
- User's product idea/brief (original text)
- `decisions/active-decisions.md` (sprint N>1)

---

## Agent Prompts

### Analyst Agent (opus)

**Always active.**

Responsibilities:
1. Extract testable **functional requirements** (FR1, FR2, ...). Each FR must be:
   - Specific: describes a concrete behavior or capability
   - Testable: has an implicit or explicit acceptance criterion
   - Traceable: can be mapped to an epic and story later
   - Non-overlapping: does not duplicate another FR
2. Identify **missing guardrails** — things the product idea implies but doesn't state (e.g., "user accounts" implies "password reset", "data entry" implies "input validation")
3. Identify **scope risks** — areas where the product idea is ambiguous and could expand uncontrollably
4. Identify **unvalidated assumptions** — beliefs embedded in the brief that haven't been tested
5. Define **scope boundaries** — explicitly state what is Out of Scope to prevent scope creep

**Additional responsibilities when `input_quality: raw-idea`:**
- Extract **user personas**: who uses this, what they care about, what they're trying to accomplish
- Define **success metrics**: measurable outcomes that would prove the product succeeded (e.g., "User can complete onboarding in <5 minutes", "Error rate <1%")
- Produce a **risk assessment**: top 3-5 risks to the product succeeding

**Output format** (to be merged by orchestrator):

```markdown
## Analyst Output

### Functional Requirements
#### FR1: [Title]
[Description. What the system must do. One testable behavior per FR.]

#### FR2: [Title]
...

### Open Questions
[Questions that must be answered before or during implementation]

### Scope Boundaries
#### In Scope
- [explicit list]

#### Out of Scope
- [explicit list]

### Assumptions
| Assumption | Validation Method | Impact if Wrong |
|---|---|---|
| [statement] | [how to verify] | [consequence] |

### User Personas (raw-idea input only)
#### [Persona Name]
- Role: [who they are]
- Goal: [what they're trying to do]
- Pain points: [what frustrates them now]

### Success Metrics (raw-idea input only)
- [Measurable outcome 1]
- [Measurable outcome 2]

### Risk Assessment (raw-idea input only)
| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
```

---

### Architect Agent (opus)

**Always active.** Runs in parallel with the analyst agent.

Responsibilities:
1. Identify **technical constraints** (TC1, TC2, ...) — things the architecture must respect:
   - Platform or runtime constraints (browser support, OS, device types)
   - Integration requirements with external systems (third-party APIs, SSO providers, existing databases)
   - Infrastructure constraints (hosting environment, cloud provider lock-in, on-prem requirements)
   - Regulatory or compliance constraints (GDPR, HIPAA, SOC2, etc.)
2. Identify **security requirements**:
   - Authentication requirements (who must log in, how)
   - Authorization requirements (role-based access, resource ownership)
   - Data protection requirements (encryption at rest/in transit, PII handling)
3. Identify **non-functional requirements** (NFR1, NFR2, ...):
   - Performance targets (response time, throughput, concurrency)
   - Availability targets (uptime SLA)
   - Scalability requirements (expected user load, data volume)
   - Maintainability requirements (logging, observability, error handling)
4. Identify **integration constraints** from the existing system, if any (API contracts that cannot change, data models already in production)

**Output format** (to be merged by orchestrator):

```markdown
## Architect Output

### Technical Constraints
#### TC1: [Title]
[Description. Why this constraint exists. Source (regulatory, existing contract, infra, etc.)]

#### TC2: [Title]
...

### Security Requirements
#### SEC1: [Title]
[Authentication, authorization, or data protection requirement]

### Non-Functional Requirements
#### NFR1: [Title]
[Description. Measurable target where possible. e.g., "API responses under 200ms at p95"]

#### NFR2: [Title]
...

### Integration Constraints
[APIs, data models, or contracts that cannot change — leave empty if none exist]
```

---

### Explore Agent (haiku)

**Always dispatched** (skip only for new repos with no source files — `new_repo: true` in discovery.md).

Responsibilities:
1. Scan the codebase for **existing implementations** that overlap with the proposed requirements:
   - Features already built that should not be re-built
   - Partial implementations that should be extended rather than replaced
2. Identify **existing API contracts** that new requirements must respect:
   - Routes and endpoints already defined
   - Request/response shapes in use
   - Database schemas and data models
3. Map **existing auth patterns** — how authentication and authorization currently work in the codebase
4. Identify any **existing test patterns** that new stories should follow

**Output format** (to be merged by orchestrator):

```markdown
## Explore Output

### Existing Feature Overlap
[Features in the codebase that overlap with proposed requirements — with file paths]

### Existing API Contracts
[Endpoints, request/response shapes, versioning conventions — with file paths]

### Existing Data Models
[Schemas, models, migrations that constrain data requirements — with file paths]

### Existing Auth Patterns
[How auth is currently implemented — with file paths]

### Existing Test Patterns
[Test framework, conventions, example test file paths]
```

---

## Agent Dispatch

All agents that apply to this sprint are dispatched in parallel.

```python
# Always dispatch analyst and architect in parallel
analyst_result = Agent(
    subagent_type="oh-my-claudecode:analyst",
    model="opus",
    prompt="""
You are the Analyst agent running Phase 1: Requirements Expansion.

Read the following inputs:
- Discovery output: .omc/sprint-plan/current/discovery.md
- Product idea: {user_product_idea}

Input quality detected in Phase 0: {input_quality}

Your task:
1. Extract all functional requirements (FR1, FR2, ...). Each must be specific, testable, traceable, and non-overlapping.
2. Identify missing guardrails, scope risks, and unvalidated assumptions.
3. Define scope boundaries (in scope / out of scope).
{raw_idea_addendum}

Use the output format specified in the Analyst Output section of the Phase 1 instructions.
Return your complete structured output. Do not write any files — the orchestrator will merge outputs.
""",
)

architect_result = Agent(
    subagent_type="oh-my-claudecode:architect",
    model="opus",
    prompt="""
You are the Architect agent running Phase 1: Requirements Expansion.

Read the following inputs:
- Discovery output: .omc/sprint-plan/current/discovery.md
- Product idea: {user_product_idea}

Your task:
1. Identify technical constraints (TC1, TC2, ...) — platform, integration, infrastructure, regulatory.
2. Identify security requirements — authentication, authorization, data protection.
3. Identify non-functional requirements (NFR1, NFR2, ...) — performance, availability, scalability, observability.
4. Identify integration constraints from the existing system (if any).

Use the output format specified in the Architect Output section of the Phase 1 instructions.
Return your complete structured output. Do not write any files — the orchestrator will merge outputs.
""",
)

# Dispatch explore (skip for new repos with no source files)
if not new_repo:
    explore_result = Agent(
        subagent_type="oh-my-claudecode:explore",
        model="haiku",
        prompt="""
You are the Explore agent running Phase 1: Requirements Expansion (codebase scan).

Read the following inputs:
- Discovery output: .omc/sprint-plan/current/discovery.md
- Existing codebase inventory in the discovery output

Your task:
1. Scan the codebase for existing feature implementations that overlap with proposed requirements.
2. Identify existing API contracts that new requirements must respect.
3. Map existing data models and auth patterns.
4. Note existing test patterns.

Focus on what CONSTRAINS the new requirements. Be specific — include file paths.
If the codebase is minimal, note that and keep the output brief.

Use the output format specified in the Explore Output section of the Phase 1 instructions.
Return your complete structured output. Do not write any files — the orchestrator will merge outputs.
""",
    )
```

**Prompt addendum when `input_quality: raw-idea`** (inject into analyst prompt as `{raw_idea_addendum}`):
```
5. Since input quality is raw-idea, also produce:
   - User personas (who uses this product and what they care about)
   - Success metrics (measurable outcomes proving the product succeeded)
   - Risk assessment (top 3-5 risks to the product succeeding)
```

---

## Merge Process

After all agents return, the orchestrator merges their outputs into a single `requirements.md`:

1. **Collect all outputs.** Analyst output, architect output, and (if explore agent ran) explore output.

2. **Number requirements sequentially.** Assign final FR numbers (FR1, FR2, ...), NFR numbers, TC numbers across all outputs combined. Do not use separate numbering per agent.

3. **Deduplicate.** If analyst and architect both identified an overlapping requirement, merge into one entry with the more precise language. Note the merge in a comment.

4. **Resolve conflicts.** If analyst and architect outputs contradict each other (e.g., analyst assumes REST API, architect recommends GraphQL), flag the conflict as an Open Question rather than silently resolving it.

5. **Ensure non-overlap.** Review all FRs and confirm no two FRs describe the same behavior. Split any FR that describes two distinct behaviors.

6. **Populate codebase context sections.** If the explore agent ran, incorporate its output into the relevant sections of `requirements.md`.

7. **Write `current/requirements.md`** using the output schema below.

---

## Decision Steering

After writing `requirements.md`, evaluate for significant decisions embedded in the requirements:

**Trigger**: Any FR, NFR, TC, or assumption that implies a HIGH or CRITICAL decision (per the significance levels in the architecture spec).

**Examples of Phase 1 decision triggers**:
- An FR implies real-time updates → technology choice (WebSockets vs polling vs SSE) is HIGH
- A TC specifies a specific cloud provider → CRITICAL if it locks the entire architecture
- An assumption about data ownership model → CRITICAL if it affects multi-tenancy design

**In GUIDED mode** (thorough, default):
- Present each HIGH/CRITICAL decision to the user using the Elicitation Format from the architecture spec
- Wait for user choice before proceeding
- Log decision in `decisions_log` in `phase-state.json`

**In AUTONOMOUS mode** (fast, or after user said "choose for me and automate"):
- Auto-decide each significant decision using the recommended option
- Log with `[AUTO-DECIDED]` marker in `phase-state.json`
- No user interaction

---

## Output Schema: `current/requirements.md`

```markdown
---
project: [name]
sprint: sprint-[NNN]
created: [date]
steering_mode: GUIDED | AUTONOMOUS
previous_sprint: sprint-[NNN-1] | null
input_quality: raw-idea | structured-brief | existing-prd
---

## Product Vision
[1-3 sentences capturing the core value proposition]

## User Personas (populated when input is raw idea)
### [Persona Name]
- Role: [who they are]
- Goal: [what they want to accomplish]
- Pain points: [what frustrates them currently]

## Success Metrics (populated when input is raw idea)
- [Metric 1: specific, measurable]
- [Metric 2: specific, measurable]

## Functional Requirements
### FR1: [Title]
[Description. What the system must do. Acceptance criteria hint.]

### FR2: [Title]
[Description.]

...

## Non-Functional Requirements
### NFR1: [Title]
[Description. Measurable target. e.g., "p95 API response time < 200ms under 100 concurrent users"]

### NFR2: [Title]
...

## Technical Constraints
### TC1: [Title]
[Description. Source and reason for this constraint.]

### TC2: [Title]
...

## Open Questions
- [Q1: Question text. Why it matters. Which phase is responsible for resolving it.]
- [Q2: ...]

## Scope Boundaries
### In Scope
- [Explicit list of what this sprint will deliver]

### Out of Scope
- [Explicit list of what is deliberately excluded]

## Assumptions
| Assumption | Validation Method | Impact if Wrong |
|---|---|---|
| [statement] | [how to verify] | [consequence if assumption is false] |

## Previous Sprint Intelligence (for sprint N>1)
[Relevant learnings and constraints from prior sprints that influenced these requirements.
Reference specific decisions from active-decisions.md that constrain or shape these FRs.]

## Existing Codebase Inventory
[Summary from Phase 0 and explore agent output, focused on elements that constrain requirements.
Include: overlapping features, API contracts that must be respected, data models already in use.
For new repos, note "New repository — no existing codebase constraints."]
```

---

## Refinement Loop Note

This phase may loop with Phase 2A. The Requirements <-> Architecture refinement loop works as follows:

1. Phase 1 produces `requirements.md`
2. Phase 2A runs architecture decisions
3. If an architecture decision **invalidates, constrains, or creates** requirements not in Phase 1:
   - Phase 1 re-runs **incrementally** — only processes the new/changed requirements
   - Phase 2A re-runs with the updated requirements
4. Maximum 3 loop iterations. After that, unresolved conflicts become Decision Steering elicitation points.

**In `fast` mode**: This loop is disabled. Phase 1 runs once, Phase 2A runs once. Conflicts are noted in `architecture-decisions.md` for user review.

The loop iteration count is tracked in `phase-state.json` under `refinement_loops.requirements_architecture.count`.

---

## Phase Completion

Phase 1 is complete when `current/requirements.md` exists, contains all required sections, and all Decision Steering elicitations (if any) have been resolved.

Update `current/phase-state.json`:
```json
{
  "current_phase": "architecture",
  "stale_phases": []
}
```

Proceed to Phase 2A: Architecture Decisions.
