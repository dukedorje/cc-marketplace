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
- `current/discovery.md` — existing codebase inventory from Phase 0

---

## Process

### Step 0: Build Shared Context Manifest

Before dispatching any enrichment agents, build a **shared context manifest** from all story stubs across all epics. This replaces sequential backward intelligence with parallel-safe shared context.

Read all story stubs from `SPRINT_DIR/epics.md` and compile:

```json
{
  "stories_by_epic": {
    "1": [
      { "id": "1.1", "title": "...", "frs": [...], "decisions": [...], "files_likely": "..." },
      { "id": "1.2", "title": "...", "frs": [...], "decisions": [...], "files_likely": "..." }
    ],
    "2": [...]
  },
  "shared_entities": ["User", "Organization", "ApiClient"],
  "shared_patterns": {
    "auth": "Stories 1.1, 2.3, 3.2 touch authentication",
    "database": "Stories 1.2, 1.3 define core schema"
  },
  "cross_epic_dependencies": [
    { "from": "2.1", "depends_on": "1.2", "reason": "needs User entity" }
  ]
}
```

This manifest is injected into every enrichment agent's prompt so all stories — regardless of epic — have visibility into what sibling stories are building.

### Step 1: Parallel Story Enrichment (fan-out across ALL epics)

For each story stub in `SPRINT_DIR/epics.md`, execute the following steps. Fire **all stories across all epics simultaneously**. Each agent receives the shared context manifest instead of sequential backward intelligence.

#### Step 1a: Load Context

Collect the following inputs for this story:
1. Story stub (user story + ACs + technical notes) from `SPRINT_DIR/epics.md`
2. Full `SPRINT_DIR/architecture-decisions.md`
3. Full `SPRINT_DIR/requirements.md`
4. **Shared context manifest** (from Step 0) — all sibling stories, shared entities, cross-epic dependencies
5. If sprint N>1: load stories from `sprint-{N-1}/stories/` that share FRs or architecture decisions with this story
6. Load relevant sections of `SPRINT_DIR/discovery.md` — existing files, patterns, integration points

#### Step 1b: Parallel Tech Research

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
Read from: SPRINT_DIR/architecture-decisions.md

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

#### Step 1d: Executor Story Enrichment

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
Read from: SPRINT_DIR/architecture-decisions.md

## Requirements
Read from: SPRINT_DIR/requirements.md

## Sibling Story Context (shared context manifest)
All stories being planned in this sprint — use this to understand what other stories are building, avoid duplication, and reference shared entities correctly:
{shared_context_manifest_json}

Your story's cross-epic dependencies:
{cross_epic_dependencies_for_this_story}

## Library Versions (from document-specialist research)
{library_versions_json}

## Codebase Context
{relevant_sections_from_discovery_md}

## Cross-Sprint Patterns (if sprint N>1)
{relevant_prior_sprint_story_summaries}

Generate the story file using the schema below. Fill every section. If a section does not apply (e.g., codebase context for a new repo), write "N/A" — do not omit the section header.

In the "Previous Story Intelligence" section, populate based on the shared context manifest:
- Files Created by Prior Stories → list files that stories with LOWER IDs in YOUR epic will likely create (from the manifest's files_likely field)
- Patterns Established → reference shared_patterns from the manifest
- Problems Encountered → "N/A" (not yet executed)

{story_file_schema}
"""
)
```

#### Step 1e: LLM Optimization Pass

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

#### Step 1f: Adversarial Validation Checklist

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

#### Step 1g: Write Output

Write the validated, optimized story file to:
```
SPRINT_DIR/stories/{epic_num}-{story_num}-{slug}.md
```

Where `{slug}` is a kebab-case version of the story title (e.g., `user-authentication`, `profile-setup`).

### Step 2: Post-Enrichment Reconciliation

After ALL stories across ALL epics are written, dispatch a verifier agent (sonnet) to scan the enriched story files for cross-epic consistency issues:

```
Agent(
  subagent_type="oh-my-claudecode:verifier",
  model="sonnet",
  prompt="""
Scan all enriched story files for cross-epic consistency issues introduced by parallel enrichment.

## Story Files
Read all files in: SPRINT_DIR/stories/

## Architecture Decisions
Read from: SPRINT_DIR/architecture-decisions.md

Check for:
1. **File ownership conflicts**: Multiple stories claiming to create or modify the same file
2. **Interface contract mismatches**: Story A exports a function with signature X, Story B imports it expecting signature Y
3. **Pattern divergence**: Stories in different epics using different patterns for the same concern (e.g., error handling, API response format)
4. **Dependency ordering issues**: Story assumes a file/module exists that only gets created by a story in a later epic
5. **Testing strategy conflicts**: Different test frameworks or patterns specified for the same module
6. **Naming inconsistencies**: Same entity/concept named differently across stories

## Output Format
{
  "status": "clean|warnings|issues",
  "findings": [
    {
      "type": "file_conflict|interface_mismatch|pattern_divergence|dependency_order|test_conflict|naming",
      "severity": "info|warning|error",
      "description": "...",
      "affected_stories": ["1.2", "3.1"],
      "suggested_fix": "..."
    }
  ]
}
"""
)
```

#### Auto-Fix

- **info**: Log only
- **warning**: Apply suggested fix if unambiguous (e.g., align naming), update affected story files
- **error**: Apply fix if possible. If not auto-fixable, flag for the inter-phase summary. Common auto-fixes:
  - File ownership conflicts → add explicit note to later story: "File X already created by Story N.M — modify, don't recreate"
  - Dependency ordering → add cross-reference in the dependent story's Previous Story Intelligence section

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

## Codebase Context
### Existing Files to Modify
[File paths + what to change — "N/A" if new repo]
### Patterns to Follow
[Existing patterns in the codebase this story must align with — "N/A" if new repo]
### Integration Points
[APIs, services, or modules this story must integrate with — "N/A" if new repo]

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

After all story files are written and reconciliation is complete, update `SPRINT_DIR/phase-state.json`:
- `current_phase`: `"validation"`
- `stories_enriched`: total count of story files written

---

## Agent Dispatch Reference

```
# Step 0: Build shared context manifest (orchestrator, no agent dispatch)

# Step 1: All stories across ALL epics (fully parallel fan-out)
Agent(subagent_type="oh-my-claudecode:document-specialist", model="sonnet", prompt="...")  # per-story, parallel
Agent(subagent_type="oh-my-claudecode:executor", model="sonnet", prompt="...")             # per-story, parallel
Agent(subagent_type="oh-my-claudecode:executor", model="sonnet", prompt="...")             # optimization pass, sequential per story

# Step 2: Post-enrichment reconciliation (after all stories complete)
Agent(subagent_type="oh-my-claudecode:verifier", model="sonnet", prompt="...")
```

Fire all stories across all epics simultaneously (executor + document-specialist pairs). Each agent receives the shared context manifest for sibling awareness. After all complete, run reconciliation to catch cross-epic inconsistencies.
