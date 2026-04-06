---
name: blocker-triage
description: Analyze architectural blockers from failed stories, assess downstream impact, and present resolution options.
user-invocable: true
argument-hint: "[--epic=N] [--story=N.M] [--auto-accept] [--sprint=ID]"
---

# blocker-triage: Architectural Blocker Analysis & Resolution

Analyzes stories with architectural blockers (`library_incompatible`, `architecture_mismatch`, `dependency_missing`), assesses downstream impact on upcoming epics, and presents resolution options.

---

## 1. Parse Arguments

Parse `$ARGUMENTS` for:

| Flag | Default | Behavior |
|------|---------|----------|
| `--epic=N` | — | Scope to blockers in epic N |
| `--story=N.M` | — | Scope to a single blocked story |
| `--auto-accept` | off | Auto-accept partial for all blockers (no elicitation) |

If no arguments: scan all story files in `SPEC_DIR/stories/` for `blocker_type` in frontmatter.

---

## 2. Sprint Resolution

Resolve the target sprint directories:
1. If `--sprint=<id>` was provided in `$ARGUMENTS`, set `STATE_DIR` = `.omc/sprint-plan/<id>/`
2. Else if `state_read` is available, read key `morphist.active_sprint`. If set, `STATE_DIR` = `.omc/sprint-plan/<value>/`
3. Else if `.omc/sprint-plan/current` symlink exists, `STATE_DIR` = `.omc/sprint-plan/current/`
4. Otherwise halt: "No active sprint found. Run `/sprint-plan` first, or pass `--sprint=<id>`."

Verify `STATE_DIR/phase-state.json` exists. Read it and set `SPEC_DIR` from the `spec_dir` field.
Verify `SPEC_DIR` exists. If not, halt: "Sprint spec directory not found at {spec_dir}. Sprint may need re-initialization."

---

## 3. Find Blocked Stories

Read story files (scoped by arguments or all). Collect stories where frontmatter contains `blocker_type` set to one of:
- `library_incompatible`
- `architecture_mismatch`
- `dependency_missing`

If no blocked stories found:
```
No architectural blockers found.
```
Stop here.

---

## 4. Impact Analysis

For each blocked story:

1. Extract the `blocker_detail` from frontmatter
2. Identify the affected library, API, or architecture decision from the detail
3. Search remaining story files (in upcoming epics) for references to the same library, API, or decision ID
4. Check epic dependency chains in `SPEC_DIR/epics.md`
5. Build a list of potentially affected downstream stories

---

## 5. Present Blocker Elicitation

If `--auto-accept`: skip elicitation, log all as `accepted_partial_auto`, and exit.

For each blocker (or group of related blockers in the same epic):

```
══════════════════════════════════════════════════════
  ARCHITECTURE BLOCKER — Decision Required
══════════════════════════════════════════════════════

  Story {N.M} ({story_title}) — {blocker_type}:
  {blocker_detail}

  {if multiple blockers in same epic}
  Also blocked:
    Story {N.X}: {blocker_type} — {blocker_detail}
  {/if}

  Potentially affected stories in upcoming epics:
    {list stories that reference the same library/decision/dependency}
    {or "None identified" if no downstream impact found}

  ──────────────────────────────────────────────────
  Options:

  1. Swap approach — Specify an alternative (e.g., different library,
     different pattern) and re-run affected stories with the new approach
  2. Update architecture decision — Revise the relevant decision in
     architecture-decisions.md, then re-run affected stories
  3. Accept partial — Continue as-is, address later
  4. Halt sprint — Stop execution for replanning

  Which approach? (If option 1 or 2, describe the alternative)
══════════════════════════════════════════════════════
```

Wait for user input.

---

## 6. Act on Decision

- **Option 1 (Swap approach)**: Record the user's alternative in the story's `blocker_detail` field as a resolution note. Update affected stories in upcoming epics: append a note under a `## Blocker Resolution` heading (before the Dev Agent Record) so executors know about the change.

- **Option 2 (Update architecture)**: Read the relevant architecture decision from `SPEC_DIR/architecture-decisions.md`. Present the current decision text and ask the user to confirm the revision. Update the decision in the file. Apply the same story annotation as option 1.

- **Option 3 (Accept partial)**: Log the decision to `execution_log` in `STATE_DIR/phase-state.json`:
  ```json
  {
    "type": "blocker_accepted",
    "story": "N.M",
    "blocker_type": "...",
    "decision": "accepted_partial",
    "rationale": "{user's explanation if provided}"
  }
  ```

- **Option 4 (Halt)**: Update `STATE_DIR/phase-state.json`:
  - Set `execution_status` to `"halted"`
  - Set `halted_at` to current ISO 8601 timestamp
  - Set `halt_reason` to a brief summary of the blocker(s)
  - Set `halt_blocker_stories` to array of affected story IDs
  Print instructions for resuming after replanning. Stop.

---

## 7. RAL Support

The `/refine` command does not target this skill. Blocker triage is a runtime decision, not an artifact to refine.
