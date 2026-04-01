---
name: replan
description: Scoped mid-sprint replanning when architecture assumptions, libraries, or dependencies break. Updates affected decisions, propagates changes to story specs, and marks stories for re-execution.
user-invocable: true
argument-hint: "[--story=N.M] [--epic=N] [--decision=D-NNN] [--reason=\"...\"] [--dry-run] [--sprint=ID]"
---

# replan: Scoped Mid-Sprint Course Correction

Handles mid-sprint replanning when an architecture decision, library choice, or dependency assumption breaks. Instead of restarting the entire planning cycle, this skill surgically updates only the affected artifacts and marks impacted stories for re-execution.

---

## 1. Argument Parsing

Parse `$ARGUMENTS`:

| Flag | Required | Description |
|------|----------|-------------|
| `--story=N.M` | No | The story that triggered the replan (provides blocker context) |
| `--epic=N` | No | Scope replan to a specific epic |
| `--decision=D-NNN` | No | Target a specific architecture decision for revision |
| `--reason="..."` | No | Free-text explanation of what broke and why |
| `--dry-run` | No | Show what would change without modifying any files |

At least one of `--story`, `--epic`, `--decision`, or `--reason` must be provided. If only `--story` is given, the skill reads blocker context from the story's Dev Agent Record.

---

## 2. Gather Context

### 2a. Sprint Resolution

Resolve the target sprint directory (`SPRINT_DIR`):
1. If `--sprint=<id>` was provided in `$ARGUMENTS`, set `SPRINT_DIR` = `.omc/sprint-plan/<id>/`
2. Else if `state_read` is available, read key `morphist.active_sprint`. If set, `SPRINT_DIR` = `.omc/sprint-plan/<value>/`
3. Else if `.omc/sprint-plan/current` symlink exists, `SPRINT_DIR` = `SPRINT_DIR/`
4. Otherwise halt: "No active sprint found. Run `/sprint-plan` first, or pass `--sprint=<id>`."

Verify `SPRINT_DIR/phase-state.json` exists. If not, halt with the same message.

### 2b. Read Sprint Artifacts

Read the following files:
- `SPRINT_DIR/architecture-decisions.md`
- `SPRINT_DIR/epics.md`
- `SPRINT_DIR/requirements.md`

If `--story=N.M` is specified:
- Read the story file from `SPRINT_DIR/stories/`
- Extract `blocker_type`, `blocker_detail`, and `Completion Notes` from the Dev Agent Record
- Extract the `decisions` list from story frontmatter (architecture decisions this story depends on)

### 2c. Identify the Broken Assumption

Determine what needs to change based on available context:

1. If `--decision=D-NNN`: the target decision is explicit
2. If `--story=N.M` with blocker context: match the blocker to architecture decisions referenced in the story
3. If `--reason` only: search architecture decisions for keywords matching the reason

Present the identified assumption to the user for confirmation:

```
═══════════════════════════════════════════════════
  REPLAN: Broken Assumption Identified
═══════════════════════════════════════════════════

  Trigger: {source — story blocker, user reason, or explicit decision}

  Affected Decision: D-{NNN} — {decision title}
  Current: {current decision text, summarized}

  Reason for change:
  {blocker_detail or user-provided reason}

  Is this the right decision to revise? (yes / specify different one)
═══════════════════════════════════════════════════
```

Wait for user confirmation before proceeding.

---

## 3. Impact Analysis

### 3a. Find Affected Stories

Search all story files in `current/stories/` for references to the affected decision(s):
- Check frontmatter `decisions` field for the decision ID
- Check `## Architecture Compliance` sections for the decision ID
- Check story body for mentions of the specific library, pattern, or technology being changed

### 3b. Categorize Impact

For each affected story, categorize the impact:

- **Direct**: Story frontmatter references the decision AND story implementation depends on it
- **Indirect**: Story doesn't reference the decision but uses the same library/pattern
- **Downstream**: Story depends on a directly affected story (via dependency chain)

### 3c. Check Execution Status

For each affected story, note its current status:
- `done` — already implemented, may need rework
- `in-progress` — currently being worked on (unlikely mid-replan)
- `ready-for-dev` — not yet started, spec needs updating
- `failed` — already failed (possibly because of this issue)

### 3d. Present Impact Report

```
═══════════════════════════════════════════════════
  IMPACT ANALYSIS
═══════════════════════════════════════════════════

  Decision: D-{NNN} — {title}

  Directly affected stories:
    Story {N.M}: {title} [{status}] — {how it's affected}
    Story {N.M}: {title} [{status}] — {how it's affected}

  Indirectly affected stories:
    Story {N.M}: {title} [{status}] — {how it's affected}

  Downstream dependencies:
    Story {N.M}: {title} [{status}] — depends on {affected story}

  Total: {count} stories affected ({done_count} already implemented)

  {if dry_run}
  DRY RUN — no changes will be made.
  {/if}
═══════════════════════════════════════════════════
```

If `--dry-run`, stop here.

---

## 4. Revise Architecture Decision

### 4a. Elicit New Approach

Present the current decision and ask for the revision:

```
## Current Decision: D-{NNN}

{full decision text from architecture-decisions.md}

## What should replace this?

Describe the new approach. For example:
- "Use ky instead of Eden Treaty for API calls"
- "Switch from SSE to WebSocket for real-time updates"
- "Use server-side sessions instead of JWT tokens"

Your revised approach:
```

Wait for user input.

### 4b. Update Architecture Decision

Dispatch an architect agent to formally revise the decision:

```python
Agent(
    subagent_type="oh-my-claudecode:architect",
    model="opus",
    prompt="""
You are revising an architecture decision mid-sprint due to a discovered incompatibility.

## Original Decision
{original_decision_text}

## Reason for Change
{blocker_detail_or_reason}

## User's New Approach
{user_provided_approach}

## Constraints
- This is a mid-sprint revision — minimize blast radius
- Preserve the decision ID (D-{NNN}) — this is an amendment, not a new decision
- Add a "## Revision History" section noting the change, date, and reason
- Keep the decision format consistent with the existing document

## Task
Write the revised decision text. Include:
1. Updated decision and rationale
2. Migration notes: what changes in implementation
3. Revision history entry

Return ONLY the revised decision text (from the decision heading through the end of that decision section).
""",
)
```

### 4c. Write Updated Decision

Update the decision in `current/architecture-decisions.md`:
- Replace the decision section (from `### D-{NNN}` through the next decision heading or end of file)
- Preserve all other decisions unchanged

---

## 5. Propagate to Stories

### 5a. Update Affected Story Specs

For each directly or indirectly affected story, dispatch an executor agent to update the story file:

```python
Agent(
    subagent_type="oh-my-claudecode:executor",
    model="sonnet",
    prompt="""
You are updating a story specification due to a mid-sprint architecture change.

## Architecture Change
Decision D-{NNN} has been revised:

### Before
{old_decision_summary}

### After
{new_decision_summary}

### Migration Notes
{migration_notes_from_architect}

## Story to Update
{story_file_content}

## Task
Update this story file to reflect the new architecture decision. Specifically:

1. Update the `## Architecture Compliance` section to reference the revised decision
2. Update acceptance criteria that reference the old approach
3. Update the `## Technical Notes` or implementation guidance
4. Add a `## Replan Notes` section (before Dev Agent Record) documenting:
   - Date: {date}
   - Reason: {reason}
   - Decision revised: D-{NNN}
   - What changed in this story's spec
5. Do NOT modify the Dev Agent Record (if present)
6. Do NOT change the story number, title, or epic assignment

Return the complete updated story file.
""",
)
```

### 5b. Update Story Statuses

For each affected story:

1. If status was `done`:
   - Change status to `ready-for-dev` (needs re-implementation)
   - Add `replan_reason: "D-{NNN} revised — {brief reason}"`
   - Preserve the existing Dev Agent Record (the re-implementor needs to see what was done before)
2. If status was `failed`:
   - Keep status as `ready-for-dev` (was already going to be retried)
   - Add `replan_reason` as above
3. If status was `ready-for-dev`:
   - Keep status, just update the spec
4. Update `execution_log` in `phase-state.json` with a replan entry:
   ```json
   {
     "type": "replan",
     "story": "N.M",
     "decision": "D-NNN",
     "previous_status": "done",
     "new_status": "ready-for-dev",
     "reason": "..."
   }
   ```

### 5c. Update Epic Status

For each epic containing affected stories:
- Recalculate epic status based on updated story statuses
- Update `epic_status` in `phase-state.json`
- Update `Status:` in `epics.md`

An epic that was `done` but now has stories reset to `ready-for-dev` becomes `partial`.

---

## 6. Update Decision Graph

If `current/decision-graph.md` exists, update it to reflect the replanned decision:

1. Read `current/decision-graph.md`
2. Find the `## D-{NNN}` section for the revised decision
3. Update the decision title if it changed
4. Add any newly affected stories discovered during impact analysis (section 3a) that weren't already listed
5. Remove any stories that no longer depend on this decision (if the revision changed the technology entirely)
6. Write the updated graph back

If the graph file does not exist, skip this step — the graph will be built on the next `/refine` invocation.

---

## 7. Write Replan Log

Write the replan record to `SPRINT_DIR/replan-log.md` (append if file exists):

```markdown
## Replan: {date}

**Trigger**: {story N.M blocker | user-initiated | decision D-NNN}
**Reason**: {reason}
**Decision revised**: D-{NNN} — {title}

### Change Summary
- **Before**: {old approach}
- **After**: {new approach}

### Affected Stories
| Story | Previous Status | New Status | Impact |
|-------|----------------|------------|--------|
| {N.M} | done | ready-for-dev | Direct — uses old library |
| {N.M} | ready-for-dev | ready-for-dev | Spec updated |

### Re-execution Required
```
/sprint-exec --story={N.M}
/sprint-exec --story={N.M}
```
```

---

## 8. Report to User

```
═══════════════════════════════════════════════════
  REPLAN COMPLETE
═══════════════════════════════════════════════════

  Decision revised: D-{NNN} — {title}
  Stories updated:  {count}
  Stories reset to ready-for-dev: {count}

  Architecture: current/architecture-decisions.md ✓
  Stories updated:
    {list each with new status}

  Replan log: current/replan-log.md

  Next steps:
    /sprint-exec --story={N.M}   # Re-run affected stories
    /sprint-exec --epic={N}      # Or re-run the whole epic

  {if any story was previously done}
  Note: {count} previously-completed stories were reset.
  Their Dev Agent Records are preserved — executors will see
  what was done before and what needs to change.
  {/if}
═══════════════════════════════════════════════════
```

---

## 9. Integration with Blocker Triage

When the blocker triage elicitation (sprint-exec section 4g) results in option 1 ("swap approach") or option 2 ("update architecture"), the user can invoke `/replan` with the relevant context:

```
/replan --story=3.2 --decision=D-005 --reason="Eden Treaty doesn't support SSE streaming"
```

The replan skill handles the rest — the user doesn't need to manually edit architecture docs or story files.
