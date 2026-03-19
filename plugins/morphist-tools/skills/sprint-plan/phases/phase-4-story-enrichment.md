# Phase 4: Story Enrichment

**Purpose**: Transform story stubs from epics.md into comprehensive implementation guides that prevent developer agent mistakes.

**Decision Steering**: Dormant. All significant decisions were made in Phases 1-2.

---

## Agents

- `executor` (sonnet) x N: One per story, all parallelized within the same epic
- `document-specialist` (sonnet): Parallel tech version research, fired alongside executor calls

---

## Input

Load from `current/` at phase start (context shedding — load artifacts only, not prior agent memory):

- `current/epics.md` — all story stubs with BDD criteria from Phase 3
- `current/architecture-decisions.md` — decisions that every story must comply with
- `current/requirements.md` — FR/NFR source of truth
- `current/stories/*.md` — previous story files (for backward intelligence when story_num > 1)
- For sprint N>1: relevant stories from `sprint-{N-1}/stories/` for cross-sprint patterns
- For brownfield: `current/discovery.md` — existing codebase inventory from Phase 0

---

## Process

### Per-Story Enrichment (parallelized across all stories within same epic)

For each story stub in `current/epics.md`, execute the following steps. Fire all stories within the same epic simultaneously. Process epics in order (complete all stories of Epic 1 before starting Epic 2) so backward intelligence is available.

#### Step 1: Load Context

Collect the following inputs for this story:
1. Story stub (user story + ACs + technical notes) from `current/epics.md`
2. Full `current/architecture-decisions.md`
3. Full `current/requirements.md`
4. If `story_num > 1`: contents of all previously enriched story files in `current/stories/` — extract "Files Created", "Patterns Established", "Problems Encountered" from their `## Dev Agent Record` and `## Previous Story Intelligence` sections
5. If sprint N>1: load stories from `sprint-{N-1}/stories/` that share FRs or architecture decisions with this story
6. If brownfield: load relevant sections of `current/discovery.md` — existing files, patterns, integration points

#### Step 2: Parallel Tech Research

Fire `document-specialist` (sonnet) in parallel with the executor call (do not wait for it before starting the executor):

```
Agent(
  subagent_type="oh-my-claudecode:document-specialist",
  model="sonnet",
  prompt="""
Research the latest stable versions and relevant API details for the following technologies referenced in this story.

## Story FRs and Decisions
FRs: {frs}
Architecture Decisions: {decisions}
Technical Notes from story stub: {technical_notes}

## Architecture Decisions (for tech stack context)
{full_content_of_architecture_decisions_md}

For each referenced library or framework:
1. Latest stable version as of today
2. Any breaking changes in recent major versions relevant to this story's use case
3. Relevant API patterns for the specific use case in this story

Return a JSON object:
{
  "libraries": [
    {
      "name": "library-name",
      "version": "x.y.z",
      "notes": "relevant API pattern or warning"
    }
  ]
}
"""
)
```

#### Step 3: Executor Story Enrichment

Fire `executor` (sonnet) to generate the comprehensive story file:

```
Agent(
  subagent_type="oh-my-claudecode:executor",
  model="sonnet",
  prompt="""
Generate a comprehensive implementation guide for this story. Your output will be read directly by a dev agent — it must be actionable, unambiguous, and token-efficient.

## Story Stub
{story_stub_content_from_epics_md}

## Architecture Decisions
{full_content_of_architecture_decisions_md}

## Requirements
{full_content_of_requirements_md}

## Previous Story Intelligence
Files created by prior stories: {file_list_from_prior_stories}
Patterns established: {patterns_from_prior_stories}
Problems encountered: {problems_from_prior_stories}

## Library Versions (from document-specialist research)
{library_versions_json}

## Brownfield Codebase Context (if applicable)
{relevant_sections_from_discovery_md}

## Cross-Sprint Patterns (if sprint N>1)
{relevant_prior_sprint_story_summaries}

Generate the story file using the schema below. Fill every section. If a section does not apply (e.g., brownfield context for a greenfield project), write "N/A" — do not omit the section header.

{story_file_schema}
"""
)
```

#### Step 4: LLM Optimization Pass

After the executor returns, apply a token efficiency review before writing to disk. Either pass this back to the executor with a focused prompt, or apply inline:

```
Agent(
  subagent_type="oh-my-claudecode:executor",
  model="sonnet",
  prompt="""
Review this story file for token efficiency. A dev agent will read this — every word must earn its place.

## Story File (draft)
{draft_story_file}

Apply these optimizations:
1. Remove redundant phrasing — if a fact appears twice, keep the clearest instance
2. Convert prose instructions to bullet points or checklists where possible
3. Ensure all ACs have explicit Given/When/Then (no implicit steps)
4. Ensure scope boundaries are concrete actions, not vague statements
5. Ensure technical requirements reference exact versions (not "latest")
6. Do NOT remove any section — optimize content within sections only

Return the optimized story file in full.
"""
)
```

#### Step 5: Adversarial Validation Checklist

After the optimization pass, run this checklist against the story file. Apply corrections automatically for any failures — do not flag them to the user.

| Check | Pass Condition |
|-------|---------------|
| Every AC has explicit Given/When/Then | No AC omits any clause |
| Scope boundaries are concrete | "DOES: creates X", not "handles X" |
| Architecture decisions referenced correctly | Each decision ID in frontmatter has a corresponding entry in `## Architecture Compliance` |
| Testing requirements are specific | Names specific framework, coverage %, test patterns — not "write tests" |
| Story completable by single agent | No AC depends on output of a different story not yet written |
| No forward dependencies | Story does not reference entities or patterns from later-numbered stories |
| FR traceability | Every FR in frontmatter has at least one AC that directly validates it |

#### Step 6: Write Output

Write the validated, optimized story file to:
```
current/stories/{epic_num}-{story_num}-{slug}.md
```

Where `{slug}` is a kebab-case version of the story title (e.g., `user-authentication`, `profile-setup`).

---

## Story File Schema

```markdown
---
sprint: sprint-[NNN]
epic: [N]
story: [M]
status: ready-for-dev
frs: [FR1, FR3]
decisions: [D-001, D-003]
---

# Story {epic}.{story}: {title}

## Story
As a [role], I want [action], so that [benefit].

## Acceptance Criteria
### AC1: [Title]
**Given** [precondition]
**When** [action]
**Then** [expected result]

### AC2: [Title]
**Given** [precondition]
**When** [action]
**Then** [expected result]

### AC3: [error case title]
**Given** [precondition]
**When** [invalid action]
**Then** [error handling behavior]

## Scope Boundaries
- This story DOES: [explicit list of concrete actions]
- This story does NOT: [explicit list — prevents dev agent scope creep]

## Tasks / Subtasks
- [ ] Task 1 (linked to AC1)
- [ ] Task 2 (linked to AC1)
- [ ] Task 3 (linked to AC2)

## Architecture Compliance
- D-001: [How this story implements/respects this decision]
- D-003: [How this story implements/respects this decision]

## Technical Requirements
### Libraries
- [library@version]: [why needed for this story]

### Testing
- Framework: [test framework]
- Coverage: [expectations]
- Test patterns: [specific patterns for this story]

## Previous Story Intelligence
### Files Created by Prior Stories
[List from earlier story Dev Agent Records — files the dev agent may read, modify, or depend on]
### Patterns Established
[Coding patterns established in prior stories that this story must follow]
### Problems Encountered
[Issues discovered in prior stories — avoid repeating them]

## Codebase Context (brownfield only)
### Existing Files to Modify
[File paths + what to change]
### Patterns to Follow
[Existing patterns in the codebase this story must align with]
### Integration Points
[APIs, services, or modules this story must integrate with]

## Anti-Patterns to Avoid
[Specific pitfalls for this story — not generic advice]

## References
- Requirements: [FR links back to requirements.md sections]
- Architecture: [Decision links back to architecture-decisions.md sections]
- Previous stories: [Links to related story files]

## Dev Agent Record
### Agent Model Used
[filled by dev agent]
### Completion Notes
[filled by dev agent]
### File List
[filled by dev agent]
### Debug Log References
[filled by dev agent]
```

---

## State Updates

After all story files are written, update `current/phase-state.json`:
- `current_phase`: `"validation"`
- `stories_enriched`: total count of story files written

---

## Agent Dispatch Reference

```
# Per-story executor call (parallel within each epic)
Agent(subagent_type="oh-my-claudecode:executor", model="sonnet", prompt="...")

# Per-story document-specialist call (parallel with executor)
Agent(subagent_type="oh-my-claudecode:document-specialist", model="sonnet", prompt="...")

# LLM optimization pass (sequential after executor, same story)
Agent(subagent_type="oh-my-claudecode:executor", model="sonnet", prompt="...")
```

Fire all stories within an epic simultaneously (executor + document-specialist pairs). Wait for an epic's stories to complete before loading their output as backward intelligence for the next epic.
