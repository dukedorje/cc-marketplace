---
name: prd
description: Interactive PRD workshop — conducts a structured interview, researches existing codebase context, generates a validated, scope-tiered PRD consumable by /sprint-plan.
user-invocable: true
argument-hint: "[<topic-text-or-file-path>] [--fast]"
---

# PRD: Product Requirements Document Workshop

You are the `prd` skill for the sprint-plan plugin. You conduct a structured, adaptive interview with the user, research the codebase for relevant context, generate a complete PRD document, and validate it before saving.

---

## 0. Argument Parsing

Parse `$ARGUMENTS` for:

1. **Topic/idea input** (optional): First positional argument. Can be:
   - Inline text: a brief description of the problem or feature idea
   - File path: path to an existing document (requirements doc, brief, etc.) — read it before beginning
2. **`--fast` flag** (optional): Skips blocker promotion to warnings; sets final status to `validated-with-warnings`

Store parsed values for use in Phase A.

---

## Phase A: Interview (interactive)

**Decision Steering: DORMANT** — this phase is information gathering only. No decisions are presented to the user.

### A.1. Seed Input Handling

If `$ARGUMENTS` contained a file path, read the file now. Use its contents as the seed for the interview — skip questions for dimensions already well-covered.

If `$ARGUMENTS` contained inline text, treat it as the initial problem description.

If no input was provided, ask the opening question:

```
What problem are you trying to solve?
```

Wait for the user's response before continuing.

### A.2. Structured Interview

Conduct an adaptive interview across these six dimensions. Ask questions ONE AT A TIME. After each answer, assess whether the dimension is sufficiently covered before moving to the next.

**Dimensions and guiding questions:**

| # | Dimension | Example question |
|---|-----------|-----------------|
| 1 | Problem statement | "What problem does this solve, and for whom?" |
| 2 | User personas | "Who are the primary users? What are their goals and pain points?" |
| 3 | Success metrics | "How will you measure whether this succeeds?" |
| 4 | Scope boundaries | "What is explicitly OUT of scope for the first version?" |
| 5 | Constraints | "Are there technical, timeline, or business constraints I should know about?" |
| 6 | Existing context | "Is there an existing system this builds on or replaces?" |

**Adaptive rules:**
- Skip any dimension the user already addressed in their seed input or a prior answer
- If an answer partially covers a dimension, ask a targeted follow-up before moving on
- If an answer covers multiple dimensions at once, update your coverage tracking accordingly
- **Target: 3–6 questions total** — not a rigid checklist. Stop when you have enough signal to write a high-quality PRD, or when all six dimensions are covered

### A.3. Interview Completion

Once you have sufficient coverage across the dimensions, confirm with the user:

```
I have enough to draft the PRD. A few things I'll include:
- [1-sentence summary of the problem]
- [key personas identified]
- [primary success metric]

Proceeding to research and PRD generation...
```

Do not wait for explicit user approval — proceed automatically after this confirmation message.

---

## Phase B: Research + FR Extraction (automated, parallel agents)

**Decision Steering: DORMANT**

Dispatch the following two agents IN PARALLEL:

### B.1. Codebase Exploration (explore, haiku)

If the user indicated this is a **greenfield project with no existing codebase**, skip this agent entirely and note "Greenfield — no codebase to scan."

Otherwise, dispatch `explore` (haiku):

```
You are scanning a codebase for context relevant to a PRD topic.

PRD Topic Summary:
---
{1-2 sentence summary of the problem and core idea from the interview}
---

Your task:
1. Identify existing implementations, modules, or files relevant to this topic
2. Note the tech stack (languages, frameworks, key libraries)
3. Identify relevant patterns, conventions, or architectural boundaries
4. Flag any existing implementations that overlap with the PRD scope (potential duplication or extension points)

Keep findings brief and concrete — file paths, module names, and 1-line descriptions. No prose padding.
```

### B.2. FR Extraction + Risk Analysis (analyst, opus)

Dispatch `analyst` (opus):

```
You are analyzing an interview transcript to extract functional requirement stubs and identify risks.

Interview Transcript:
---
{full interview transcript, including seed input and all Q&A}
---

Your tasks:

**Part 1: FR Stub Extraction**
Extract numbered functional requirement stubs from the interview. Each stub should be:
- A single, testable behavior (binary pass/fail)
- Written as: "The system shall [action] [condition/context]"
- Numbered: FR1, FR2, FR3, ...
- Provisionally tagged [MVP], [Growth], or [Vision] based on this heuristic:
  - [MVP]: FRs that directly address the core problem statement
  - [Growth]: FRs that enhance or extend MVP behavior
  - [Vision]: Nice-to-haves, stretch goals, or future capabilities

Extract as many FRs as the interview warrants. Err toward completeness — the writer will consolidate.

**Part 2: Risk + Gap Analysis**
- Identify unvalidated assumptions in the interview (things stated as fact that may not be true)
- Identify missing requirement categories (what dimension is conspicuously absent?)
- Identify scope ambiguity (things that could be interpreted multiple ways)
- Identify technical risks based on the codebase context (if provided)

Format:
## FR Stubs
FR1. [MVP] The system shall ...
FR2. [Growth] The system shall ...
...

## Risks & Gaps
- [Assumption] ...
- [Missing] ...
- [Ambiguous] ...
- [Technical] ...
```

### B.3. Merge Research

After both agents complete, compile a research summary containing:
- Codebase scan findings (or "Greenfield" note)
- FR stubs (numbered, provisionally tagged)
- Risks & gaps

This summary is the input to Phase C.

---

## Phase C: PRD Generation (automated)

**Decision Steering: ACTIVE for scope tier assignment only (HIGH significance in GUIDED mode)**

### C.1. Generate PRD (writer, sonnet)

Dispatch `writer` (sonnet):

```
You are writing a Product Requirements Document (PRD) from an interview transcript and research findings.

Interview Transcript:
---
{full interview transcript}
---

Research Summary (codebase scan + FR stubs + risks):
---
{Phase B research summary}
---

Instructions:
1. Write a complete PRD following the exact structure below
2. Use the analyst's FR stubs as your functional requirements — FORMAT them into final prose, do NOT generate new FRs from scratch
3. For User Journeys: synthesize 2-3 narrative walkthroughs from the personas and FRs. These are NOT from the interview — write them yourself. If context is insufficient, write: "Journey detail insufficient — consider refining after review."
4. Every FR must have exactly one tier tag: [MVP], [Growth], or [Vision]
5. Use the analyst's provisional tags as defaults, but apply judgment to adjust them
6. Write in clear, unambiguous language. Avoid subjective adjectives without metrics (e.g., not "fast" — say "under 200ms")
7. For NFRs: infer from the problem context and constraints. Minimum 2 NFRs.

PRD Structure (use this EXACTLY):

---
title: [Product Name — infer from the problem description]
created: {today's date}
status: draft
scope_tier: mvp
---
# PRD: [Product Name]

## Problem Statement
[2-4 sentences describing the problem, its impact, and why it matters now]

## User Personas
[For each persona: Name/role, primary goals, pain points, technical level]

## User Journeys
[2-3 narrative walkthroughs. Each journey: persona + trigger + step-by-step flow + outcome]

## Success Metrics
[Quantifiable targets. Each metric: what, how measured, target value, timeframe]

## Functional Requirements
[FR1 through FRN. Each on its own line, tagged [MVP], [Growth], or [Vision]]

FR1. [MVP] The system shall ...
FR2. [MVP] The system shall ...
...

## Non-Functional Requirements
NFR1. [category] ...
NFR2. [category] ...

## Scope Boundaries
### In Scope
- [Explicit list of what is included]

### Out of Scope
- [Explicit list of what is excluded, including Growth/Vision FRs deferred from MVP]

## MVP / Growth / Vision Tiers
### MVP
[List of FR numbers tagged [MVP] with 1-line summary]

### Growth
[List of FR numbers tagged [Growth] with 1-line summary]

### Vision
[List of FR numbers tagged [Vision] with 1-line summary]

## Constraints
[Technical, timeline, resource, and business constraints from the interview]

## Assumptions & Risks
[Each item: assumption/risk statement + potential impact + mitigation suggestion]

## Open Questions
[Questions that remain unanswered after the interview and research phase]

## Existing System Context
[If applicable: relevant existing code, patterns, or architecture from the codebase scan. If greenfield: "No existing system — greenfield implementation."]
```

### C.2. Scope Tier Decision (GUIDED mode only)

After the writer completes, extract the proposed MVP/Growth/Vision split from the PRD.

**In GUIDED mode:** Present the scope tier assignment to the user:

```
## Decision Point: Scope Tier Assignment

**What is being decided**: Which functional requirements belong in MVP vs. Growth vs. Vision
**Why it matters**: MVP scope determines what /sprint-plan will plan for; Growth/Vision FRs are deferred
**Significance**: HIGH

### Proposed Split

**MVP** ({count} FRs):
{list FR numbers and 1-line summaries}

**Growth** ({count} FRs):
{list FR numbers and 1-line summaries}

**Vision** ({count} FRs):
{list FR numbers and 1-line summaries}

### Options
1. **Accept** — proceed with this tier assignment
2. **Adjust** — specify which FRs to move between tiers
3. **Revise** — describe the change you want and I'll update the PRD
```

Wait for user response. Apply any requested adjustments to the PRD before proceeding to Phase D.

**In AUTONOMOUS mode:** The writer's default tier assignment stands. Proceed to Phase D automatically.

### C.3. Generate Slug and Save Path

Generate a slug from the PRD title:
- Lowercase, hyphens replacing spaces and special characters
- Max 40 characters
- Example: "User Authentication System" → `user-authentication-system`

Save path: `.omc/sprint-plan/prd-{slug}.md`

Ensure `.omc/sprint-plan/` directory exists. Create it if not.

Write the PRD to the save path.

---

## Phase D: Validation & Polish (automated)

**Decision Steering: DORMANT**

### D.1. Critic Validation (critic, opus)

Dispatch `critic` (opus) with the full PRD content:

```
You are validating a Product Requirements Document. Run the following 6 checks and report all findings.

PRD:
---
{full PRD content}
---

**Check 1: FR Testability**
Every FR must be expressible as a binary pass/fail test. Flag FRs containing subjective adjectives without quantifiable targets: "easy", "fast", "intuitive", "simple", "user-friendly", "seamless", "robust" (when used without a metric). For each flagged FR, quote the problematic phrase.

**Check 2: FR Non-Overlap**
No two FRs should describe the same behavior. Identify any FR pairs that overlap in scope. Quote both FRs.

**Check 3: Scope Tier Completeness**
Every FR must have exactly one tier tag ([MVP], [Growth], or [Vision]). Flag any FR missing a tag or having multiple tags.

**Check 4: Parity Check**
Every dimension covered in the interview must appear in the PRD. Check that: problem statement, user personas, success metrics, scope boundaries, constraints, and existing context sections are all present and non-trivial (not placeholder text).

**Check 5: Measurability Audit**
Success metrics must have quantifiable targets (numbers, percentages, timeframes). Flag metrics that are vague or qualitative.

**Check 6: Persona-FR Traceability**
Each FR should connect to at least one persona. Flag "orphan FRs" — FRs that cannot be traced to any persona in the User Personas section.

**Finding Classification:**
For each finding, classify as:
- **auto-fix**: A mechanical fix (add missing tag, remove redundant FR, rewrite vague metric with placeholder value) — the writer can apply without user input
- **warning**: A quality concern that doesn't prevent use but should be addressed
- **blocker**: A structural problem that would make the PRD unreliable as a sprint-plan input (unfixable without additional information)

Format findings as:

## Validation Findings

### Check 1: FR Testability
- [auto-fix|warning|blocker] FR{N}: {description of finding}
...

### Check 2: FR Non-Overlap
...

[continue for all 6 checks]

## Summary
- Auto-fix: {count}
- Warnings: {count}
- Blockers: {count}
```

### D.2. Auto-Fix Loop (max 1 iteration)

If there are **auto-fix findings**:

Dispatch `writer` (sonnet) to apply the auto-fixes:

```
You are applying targeted fixes to a PRD based on validation findings.

PRD:
---
{current PRD content}
---

Auto-fix findings to apply:
---
{list of all auto-fix findings with descriptions}
---

Apply ONLY the auto-fix findings. Do not change anything else. Return the complete updated PRD.
```

Write the updated PRD back to the save path.

Re-dispatch `critic` (opus) once to re-validate. This is the maximum — do not loop further.

### D.3. Status Transition

After validation (and optional auto-fix):

Determine the final status:

| Condition | Status |
|-----------|--------|
| No blockers remain | `validated` |
| Blockers remain (normal mode) | `draft-with-issues` |
| `--fast` flag present (any blockers downgraded to warnings) | `validated-with-warnings` |

Update the PRD frontmatter `status` field to the final status.

Write the final PRD to the save path.

### D.4. Present Results to User

Display the validation summary and next step:

**If status is `validated`:**
```
PRD saved to .omc/sprint-plan/prd-{slug}.md

Validation: PASSED
- Auto-fixes applied: {count}
- Warnings: {count}
- Blockers: 0

Run `/sprint-plan .omc/sprint-plan/prd-{slug}.md` to begin sprint planning.
```

**If status is `validated-with-warnings` (--fast mode):**
```
PRD saved to .omc/sprint-plan/prd-{slug}.md

Validation: PASSED WITH WARNINGS (--fast mode)
- Auto-fixes applied: {count}
- Warnings: {count} (including {blocker-count} downgraded from blocker)

Run `/sprint-plan .omc/sprint-plan/prd-{slug}.md` to begin sprint planning.
Note: Resolve warnings before production use.
```

**If status is `draft-with-issues`:**
```
PRD saved to .omc/sprint-plan/prd-{slug}.md

Validation: ISSUES FOUND
- Auto-fixes applied: {count}
- Warnings: {count}
- Blockers: {count} (must resolve before using with /sprint-plan)

Blocking issues:
{list each blocker with a 1-line description}

Resolve these issues manually by editing .omc/sprint-plan/prd-{slug}.md, then re-run `/prd` on the updated file or proceed to `/sprint-plan` at your own discretion.
```

---

## Decision Steering Summary

| Phase | Decision Steering State |
|-------|------------------------|
| Phase A: Interview | DORMANT |
| Phase B: Research + FR Extraction | DORMANT |
| Phase C: PRD Generation | ACTIVE — scope tier only (HIGH significance in GUIDED) |
| Phase D: Validation & Polish | DORMANT |

---

## /ral prd Support

`prd` is a valid target in the `/ral` skill.

- **Artifact path**: `.omc/sprint-plan/prd-{slug}.md` (most recent by mtime if no explicit path given)
- **RALPLAN-DR pass**: Planner proposes improvements → Architect challenges → Critic arbitrates
- **Downstream stale marking**: Does NOT mark downstream phases stale (PRD is an upstream artifact; sprint-plan phases are downstream and not tracked in PRD's phase-state)

---

## Usage Examples

```
/prd
# No arguments — prompts "What problem are you trying to solve?"

/prd "We need a way for users to track their daily water intake"
# Inline text seed — skips problem statement question, continues with remaining dimensions

/prd /path/to/product-brief.md
# File path seed — reads the file, skips covered dimensions

/prd "notification system" --fast
# Inline text with --fast mode — blockers downgraded to warnings
```

---

## Notes on PRD Consumption by /sprint-plan

When `/sprint-plan` Phase 1 reads a PRD produced by this skill:

- `status: validated` → light-touch processing (focus on expansion and story decomposition, not re-validation)
- `status: draft` or `draft-with-issues` → full validation run as part of requirements expansion
- `scope_tier: mvp` → Phase 1 filters FRs to [MVP] tags only; Growth/Vision FRs are listed in Out of Scope for the sprint

The numbered FR format (FR1, FR2, ...) is what causes Phase 0 to classify the input as `existing-prd` quality, enabling higher-fidelity requirement ingestion.
