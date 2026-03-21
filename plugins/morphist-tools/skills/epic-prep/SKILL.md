---
name: epic-prep
description: Pre-execution deep dive into an epic — enrich stories, drill into specifics, revise decisions, and propagate changes via the decision graph. Interactive preparation before running /sprint-exec.
user-invocable: true
argument-hint: "[--epic=N] [--story=N.M] [--graph]"
---

# epic-prep: Pre-Execution Epic Deep Dive

Interactive preparation before executing an epic. Mirrors planning-depth rigor but is scoped to a single epic and its stories. Enrich story specs, drill into implementation details, revise architecture decisions, and propagate changes to dependent stories via the decision graph.

---

## 1. Argument Parsing

Parse `$ARGUMENTS`:

| Flag | Required | Description |
|------|----------|-------------|
| `--epic=N` | No | Target epic to prepare. If omitted, selects the next unexecuted epic. |
| `--story=N.M` | No | Drill directly into a specific story for deep review |
| `--graph` | No | Display the decision graph for the epic (or story) and exit |

If neither flag is provided, determine the next epic to execute:
1. Read `phase-state.json` for `epic_status`
2. Find the first epic with status `"pending"` or not yet tracked
3. Confirm with user: `Prepare Epic {N}: {title}?`

---

## 2. Load Context

### 2a. Sprint Artifacts

Read from `.omc/sprint-plan/current/`:
- `phase-state.json` — execution status, epic status
- `architecture-decisions.md` — current ADRs
- `epics.md` — epic definitions and dependencies
- `requirements.md` — traced requirements
- `decision-graph.md` — decision dependency graph (create if missing, see section 8)
- `replan-log.md` — prior replans (if exists)

### 2b. Epic Stories

Read all story files for the target epic from `current/stories/`.

For each story, extract:
- Frontmatter (status, decisions list, blocker info)
- Acceptance criteria
- Technical notes
- Architecture compliance section
- Dev Agent Record (if previously executed)

### 2c. Decision Graph for This Epic

From `decision-graph.md`, extract all ADRs that reference any story in this epic. Also extract any stories in *other* epics that share those same ADRs (these are the ripple targets).

---

## 3. Epic Overview

Present the epic with its decision graph context:

```
═══════════════════════════════════════════════════
  EPIC {N}: {title}
  Status: {status}
═══════════════════════════════════════════════════

  Goal: {epic goal from epics.md}

  Stories:
    {N}.1: {title} [{status}]
    {N}.2: {title} [{status}]
    {N}.3: {title} [{status}]

  ─────────────────────────────────────────────────
  Decision Graph (ADRs affecting this epic):

    D-003: ky for HTTP client
      → {N}.1, {N}.3, {other_epic}.{story}

    D-007: Zustand for client state
      → {N}.2

    D-012: Server-side validation with Zod
      → {N}.1, {N}.2, {N}.3, {other_epic}.{story}

  ─────────────────────────────────────────────────

  Commands:
    story {M}       — drill into story {N}.{M}
    enrich          — auto-enrich all stories with implementation details
    graph           — show full decision graph for this epic
    ready           — mark prep complete, show reconcile suggestion if needed
    quit            — exit prep
═══════════════════════════════════════════════════
```

Wait for user input. This is an interactive session — the user drives the drill-down.

---

## 4. Story Drill-Down (`story {M}` or `--story=N.M`)

When the user selects a story, present it in full with its decision context:

```
═══════════════════════════════════════════════════
  STORY {N}.{M}: {title}
  Status: {status}
═══════════════════════════════════════════════════

  Acceptance Criteria:
    1. {AC-1}
    2. {AC-2}
    3. {AC-3}

  Architecture Decisions:
    D-003: ky for HTTP client — {one-line summary}
    D-012: Server-side validation with Zod — {one-line summary}

  Technical Notes:
    {existing technical notes from story file}

  {if dev_agent_record exists}
  Previous Implementation:
    Files: {file list}
    Notes: {completion notes summary}
    Blocker: {blocker_type — blocker_detail, if any}
  {/if}

  ─────────────────────────────────────────────────
  Commands:
    enrich        — auto-enrich with implementation details
    edit ac       — modify acceptance criteria
    edit tech     — modify technical notes
    edit decision — propose a change to an ADR
    add note      — add a prep note for the executor
    back          — return to epic overview
═══════════════════════════════════════════════════
```

### 4a. `enrich` — Auto-Enrich Story

Dispatch an architect agent to enrich the story with implementation specifics:

```python
Agent(
    subagent_type="oh-my-claudecode:architect",
    model="opus",
    prompt="""
You are enriching a story specification before implementation.

## Story
{story_file_content}

## Architecture Decisions
{relevant_decisions_full_text}

## Existing Codebase Context
{discovery.md content if available}

## Task

Review this story and add concrete implementation guidance:

1. **File Plan**: List the specific files that will need to be created or modified,
   with a brief note on what each file does. Use the codebase conventions from discovery.md.
2. **Implementation Sequence**: Suggest the order to implement (what depends on what within this story).
3. **Edge Cases**: Flag any edge cases or ambiguities in the ACs that the executor should handle.
4. **Integration Points**: Note where this story's work touches other stories' files or APIs.
5. **Risk Areas**: Flag anything that might not work as expected (library limitations, untested patterns).

Do NOT modify acceptance criteria or change the story's scope.
Return ONLY the enrichment content to be added under a `## Prep Notes` section.
""",
)
```

Insert the enrichment under a `## Prep Notes` section in the story file (before the Dev Agent Record if present).

If a `## Prep Notes` section already exists (from a previous `/epic-prep` run), append the new enrichment under a `### Update: {ISO 8601 timestamp}` subsection rather than replacing existing notes. Previous prep context is valuable — the executor should see the full history of preparation.

### 4b. `edit ac` — Modify Acceptance Criteria

Present the current ACs and let the user modify them interactively:
- Add new ACs
- Remove or revise existing ACs
- The skill updates the story file directly

### 4c. `edit tech` — Modify Technical Notes

Present current technical notes for the user to revise. Update the story file.

### 4d. `edit decision` — Propose ADR Change

This is the key integration point with the decision graph.

1. Show the ADRs affecting this story
2. Let the user select which decision to revise
3. Capture the user's proposed change
4. Dispatch an architect agent to formally revise the decision (same pattern as replan section 4b)
5. Update `architecture-decisions.md` with the revision
6. **Update the decision graph** (section 8b)
7. Track the change for the ripple pass (section 6)

```python
Agent(
    subagent_type="oh-my-claudecode:architect",
    model="opus",
    prompt="""
You are revising an architecture decision during pre-execution preparation.

## Original Decision
{original_decision_text}

## User's Proposed Change
{user_input}

## Constraints
- This is a prep-time revision, not a mid-sprint emergency
- Preserve the decision ID (D-{NNN}) — this is an amendment
- Add a "## Revision History" entry noting: date, reason, context (epic-prep for Epic N)
- Keep format consistent with other decisions in the document

## Task
Write the revised decision text. Include:
1. Updated decision and rationale
2. Migration notes: what changes in implementation
3. Revision history entry

Return ONLY the revised decision section.
""",
)
```

### 4e. `add note` — Add Prep Note

Append a free-text note under the `## Prep Notes` section. These notes are visible to the executor agent.

---

## 5. Bulk Enrich (`enrich` from epic overview)

When called from the epic overview (not story drill-down), enrich all stories in the epic:

1. Dispatch architect agents in parallel (one per story, up to 5 concurrent)
2. Each agent enriches its story (same prompt as 4a)
3. Present a summary of enrichments added
4. Return to the epic overview

---

## 6. Decision Change Tracking & Ripple

Throughout the interactive session, track all ADR changes made via `edit decision`:

```json
{
  "changes": [
    {
      "decision": "D-003",
      "title": "HTTP client library",
      "old_summary": "Use Eden Treaty",
      "new_summary": "Use ky",
      "changed_during_story": "3.2",
      "affected_stories_in_epic": ["3.1", "3.3"],
      "affected_stories_other_epics": ["4.1", "4.2"]
    }
  ]
}
```

This is used by the `ready` command (section 7) to suggest reconciliation.

---

## 7. Finalize Prep (`ready`)

When the user signals they're done:

### 7a. Summary

```
═══════════════════════════════════════════════════
  EPIC {N} PREP COMPLETE
═══════════════════════════════════════════════════

  Stories enriched:     {count}/{total}
  ACs modified:         {count} stories
  Tech notes updated:   {count} stories
  ADRs revised:         {count}
  Prep notes added:     {count}

  {if adr_changes}
  ─────────────────────────────────────────────────
  ADR Changes Made:

    D-003: Eden Treaty → ky
      Stories in this epic: 3.1, 3.3
      Stories in other epics: 4.1, 4.2

  ⚠ Decision changes may affect stories in other epics.
    Run: /reconcile --decisions
    This will propagate the changes to affected story specs.
  {/if}

  Ready to execute:
    /sprint-exec --epic={N}
═══════════════════════════════════════════════════
```

### 7b. Auto-Suggest Reconcile

If any ADRs were changed, suggest `/reconcile --decisions` before execution. Do NOT auto-run it — the user should decide.

### 7c. Update Decision Graph

Ensure all decision graph updates from this session have been persisted (section 8b).

---

## 8. Decision Graph

The decision graph is a persistent artifact at `.omc/sprint-plan/current/decision-graph.md` that maps architecture decisions to the stories that depend on them.

### 8a. Format

```markdown
# Decision Graph

Maps architecture decisions to dependent stories. Updated by /sprint-plan, /epic-prep, /replan.

## D-001: {decision title}
- 1.1: {story title}
- 1.3: {story title}
- 2.2: {story title}

## D-003: {decision title}
- 2.1: {story title}
- 2.3: {story title}
- 3.1: {story title}
- 4.2: {story title}

## D-007: {decision title}
- 1.2: {story title}
- 3.3: {story title}
```

Each section is an ADR ID with a flat list of story references (epic.story format) and their titles.

### 8b. Maintenance

The decision graph is updated by:

- **`/sprint-plan`** (Phase 4 or 5): Seeds the initial graph by reading `decisions` fields from all story frontmatter and cross-referencing with `architecture-decisions.md`
- **`/epic-prep`**: Updates when ADRs are revised or when enrichment discovers new dependencies
- **`/replan`**: Updates when decisions are revised mid-sprint
- **`/reconcile --decisions`**: Reads the graph to find stories affected by decision changes

### 8c. Deep Scan

When the graph may be incomplete (e.g., after initial seeding), run a deep scan:

1. Read all story files
2. Read all ADRs from `architecture-decisions.md`
3. For each ADR, search story files for:
   - Explicit `decisions` frontmatter references
   - Mentions of the decision ID (D-NNN) in story body
   - Mentions of the specific library, pattern, or technology named in the decision
4. Update the graph with any newly discovered dependencies
5. Mark each entry as `explicit` (from frontmatter) or `inferred` (from content scan)

Inferred dependencies use a different marker:

```markdown
## D-003: ky for HTTP client
- 2.1: API client setup
- 2.3: Auth token refresh
- 3.1: Data fetching hooks
- ~4.3: Payment webhook handler~ *(inferred — uses HTTP client pattern)*
```

The tilde-wrapped entries are inferred and should be confirmed during `/epic-prep` or removed if false.

### 8d. Bootstrap (if graph doesn't exist)

If `decision-graph.md` does not exist when `/epic-prep` is invoked:

1. Run the deep scan (8c) to build the initial graph
2. Present the generated graph to the user for review
3. Write it to `current/decision-graph.md`

---

## 9. Integration Points

### With `/sprint-exec`
- Run `/epic-prep --epic=N` before `/sprint-exec --epic=N` for a prepared execution
- Epic-prep enrichments appear as `## Prep Notes` in story files, visible to executor agents

### With `/replan`
- If an ADR was revised during epic-prep, `/replan` is NOT needed — epic-prep handles the revision directly
- `/replan` is for mid-execution breakage; `/epic-prep` is for pre-execution deliberation
- Both update the decision graph

### With `/reconcile`
- After epic-prep, if ADRs changed: `/reconcile --decisions` propagates changes to affected story specs in other epics
- `/reconcile` (without `--decisions`) handles code-level style inconsistencies post-execution

### With `/update-status --show`
- The status dashboard renders the decision graph for each epic (see update-status integration)

### With `/audit-story`
- After executing a prepped epic, `/audit-story` can verify that prep notes and enrichments were followed
