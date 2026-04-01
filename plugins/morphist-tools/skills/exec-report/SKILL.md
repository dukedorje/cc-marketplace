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
- Story files in `SPRINT_DIR/stories/{N}-*` for the completed epic
- `SPRINT_DIR/epics.md` for epic titles and next-epic information

---

## 2. Epic Completion Report — Inter-Epic Review Brief

After an epic completes, generate a human-oriented review brief. This is the primary checkpoint for the user to assess quality before proceeding. It has four sections: summary, story results, cross-story reconciliation, and suggested next steps (including optional manual testing).

### 2a. Gather Inputs

For each completed story in the epic, read:
- Dev Agent Record: `Completion Notes`, `File List`, `Blocker Type`, `Blocker Detail`
- Spec: acceptance criteria, `decisions` frontmatter
- Story frontmatter: `status`

Also read:
- `SPRINT_DIR/architecture-decisions.md` for referenced ADR titles
- `SPRINT_DIR/epics.md` for epic title and next-epic info

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

### 2e. Manual Test Suggestions (optional)

Dispatch a writer agent (haiku) to generate 3-6 concrete manual test actions derived from the acceptance criteria of all completed stories. Frame as quick, practical things the user can try — not a formal test plan.

```
  Try it yourself (optional):
    □ {concrete action — what to do and what to expect}
    □ {concrete action}
    □ {concrete action}
    ...
```

Guidelines for the writer agent:
- Derive from acceptance criteria, not from implementation details
- Be specific: include example commands, URLs, inputs, or UI actions
- State the expected outcome for each action
- Prioritize user-facing behavior over internal correctness
- Keep it to 3-6 items — enough to build confidence, not enough to be a chore

### 2f. Next Steps

Show contextual next steps based on the epic's outcome. Only show options that are relevant.

```
  ─────────────────────────────────────────────────
  Next steps:
    [continue]     → Epic {N+1}: {next_epic_title} ({story_count} stories)
    {if failed_count > 0}
    [fix {N.M}]    → Re-run a failed story (e.g., /sprint-exec --story={first_failed})
    {/if}
    {if reconcile_warnings > 0}
    [reconcile]    → Full cross-story reconciliation (/reconcile --epic={N})
    {/if}
    [audit {N.M}]  → Deep investigation of a story (/audit --epic={N})
    [refine]       → Revise epic spec before continuing (/refine --epic={N+1})
    [log "..."]    → Annotate a finding (/log "..." --epic={N})
    [backlog]      → Add follow-up items (/backlog add "...")
    [halt]         → Pause execution for manual work
    {if last_epic}
    [retro]        → Generate sprint retrospective (/retro)
    {/if}
  {if background_review}
  Background review dispatched → SPRINT_DIR/reviews/epic-{N}-review.md
  {/if}
═══════════════════════════════════════════════════
```

Wait for user input before proceeding to the next epic.

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
