---
name: exec-report
description: Generate epic or sprint execution progress report from phase-state.json and story files.
user-invocable: false
---

# exec-report: Execution Progress Report Generator

Generates formatted progress reports for epic completion or full sprint completion. Called internally by sprint-exec after each epic; not user-invocable.

---

## 1. Determine Report Type

This skill is called with context about which epic just completed. It reads:
- `phase-state.json` for `epic_status`, `execution_log`, `sprint_number`
- Story files in `SPEC_DIR/stories/{N}-*` for the completed epic
- `SPEC_DIR/epics.md` for epic titles and next-epic information

---

## 2. Epic Completion Report — Inter-Epic Huddle

After an epic completes, run the inter-epic huddle. This is the primary checkpoint for the user to assess quality, absorb learnings, and decide how to proceed before the next epic begins.

### 2a. Gather Inputs

For each completed story in the epic, read:
- Dev Agent Record: `Completion Notes`, `File List`, `Blocker Type`, `Blocker Detail`
- Spec: acceptance criteria, `decisions` frontmatter
- Story frontmatter: `status`

Also read:
- `SPEC_DIR/architecture-decisions.md` for referenced ADR titles
- `SPEC_DIR/epics.md` for epic title and next-epic info

### 2b. Epic Narrative Summary

Dispatch a writer agent (haiku) to synthesize a 2-3 sentence changelog-style summary from all story Dev Agent Records. The summary should describe **what was built** in concrete terms, not just "stories were completed."

```
═══════════════════════════════════════════════════
  EPIC {N} COMPLETE: {epic_title}
  Status: {done|partial|failed}
═══════════════════════════════════════════════════

  {narrative_summary}

```

### 2c. Story Results

```
  Stories:  {done_count}/{total_count} done
            {failed_count} failed, {blocked_count} blocked, {already_done_count} already done

  {for each story}
    {✓|✗|⚠} Story {N.M}: {title} — {status}
      {1-line completion note from Dev Agent Record, or failure reason}
      Files: {comma-separated file list, abbreviated if >5}
  {/for}

  {if verification_ran}
  Verification: {pass_count} passed, {concerns_count} concerns, {fail_count} failed
  {/if}
```

### 2d. Cross-Story Reconciliation (quick scan)

Since stories in an epic run in parallel, they can make conflicting choices. Dispatch a verifier agent (sonnet) to do a consistency scan across the file lists from all completed stories. Check for:

- **Duplicate utilities**: Multiple stories creating similar helper functions
- **Naming inconsistencies**: Same concept with different names (e.g., `ApiError` vs `HttpError`)
- **Import pattern divergence**: Different import styles or module resolution
- **Shared file conflicts**: Multiple stories modifying the same file (check git status for conflicts)
- **Style drift**: Obvious pattern mismatches (e.g., one story uses classes, another uses functions for the same concern)

Output:

```
  Cross-story check:
    {✓|⚠} {finding description}
    ...
```

If all clean, show a single `✓ No cross-story inconsistencies detected`. If issues found, mark with `⚠` — these are informational, not blocking.

### 2e. Learnings for Next Epic

Dispatch a sonnet agent to read all Dev Agent Records from the just-completed epic and extract insights relevant to upcoming epics. The agent should look for:

- Patterns discovered during implementation that upcoming stories should follow
- Unexpected API behaviors or library quirks that affect future work
- Files/modules created that future epics will need to integrate with
- Anything that worked well or went wrong that should inform the next epic

Format output as concrete, actionable bullet points:

```
  Learnings for Epic {N+1}:
    • {actionable insight}
    • {actionable insight}
    ...
```

Persist the learnings to `phase-state.json` under `epic_learnings.{N}` before proceeding.

### 2f. User Feedback Prompt

After displaying everything above, prompt the user:

```
  Anything you're noticing? Feedback, feelings, course corrections?
  (Type your thoughts, or just hit enter to see options)
```

Wait for user input. If they type feedback:
- Fold it into the learnings by appending to `epic_learnings.{N}` in `phase-state.json`
- Acknowledge briefly (one line)
- Then show the menu

If the user hits enter with no input, proceed directly to the menu.

### 2g. Next Steps Menu

Show contextual options based on epic outcome. Only show options that are relevant.

```
  ─────────────────────────────────────────────────
  Next up: Epic {N+1} — {title} ({story_count} stories)

  [go]           → Execute as-is
  [enrich N.M]   → Flesh out an underspecified story before executing
  [redirect]     → Adjust next epic's stories based on learnings
  [replan]       → Significant course correction (/replan)
  [yolo]         → Skip huddles for remaining epics, just build
  [hold]         → Pause execution for manual work
  {if failed stories}
  [fix N.M]      → Re-run a failed story (/sprint-exec --story=N.M)
  {/if}
  {if reconcile_warnings}
  [reconcile]    → Full reconciliation (/reconcile --epic={N})
  {/if}
  {if last_epic}
  [retro]        → Generate sprint retrospective (/retro)
  {/if}

  {if background_review}
  Background review dispatched → STATE_DIR/reviews/epic-{N}-review.md
  {/if}
═══════════════════════════════════════════════════
```

Wait for user input before proceeding.

**Menu action handling:**

- **go**: Proceed to next epic
- **enrich N.M**: Run enrichment for that specific story (invoke Phase 4 enrichment for just that story), then return to the menu
- **redirect**: Present the learnings and user feedback to a planner (sonnet) agent that reads the next epic's stories from `SPEC_DIR/epics.md` and proposes adjustments. Show proposed changes, let user approve/modify, then update the stories.
- **replan**: Invoke `/replan` skill
- **yolo**: Set `huddle_mode: "skip"` in `phase-state.json` so remaining epics auto-continue without huddles
- **hold**: Stop execution, preserve `resume_point` in `phase-state.json`
- **fix N.M**: Invoke `/sprint-exec --story=N.M`
- **reconcile**: Invoke `/reconcile --epic={N}`
- **retro**: Invoke `/retro`

---

## 3. Sprint Completion Report

When all epics are done, output the full sprint summary:

```
--- Sprint {sprint_number} Execution Complete ---

Results:
  Done:    {total_done} stories
  Failed:  {total_failed} stories
  Skipped: {total_skipped} stories (blocked or already done)

{if total_failed > 0}
Failed stories require attention:
{list each failed story: "  - Story N.M: {title}"}

Retry individual stories with:
  /sprint-exec --story=N.M

Retries use opus model for improved reasoning.
{/if}

{if total_failed == 0}
All stories completed successfully.
{/if}

Dev Agent Records have been written to each story file.
Next steps:
  /retro                    # Generate sprint retrospective
  /audit --all              # Deep audit of all stories (optional)
  /backlog --scan           # Catch follow-ups and TODOs
```
