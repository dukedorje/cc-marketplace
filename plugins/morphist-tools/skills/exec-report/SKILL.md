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

## 2. Epic Completion Report

After an epic completes, output:

```
═══════════════════════════════════════════════════
  EPIC {N} COMPLETE: {epic_title}
  Status: {done|partial|failed}
═══════════════════════════════════════════════════

  Stories:  {done_count}/{total_count} done
            {failed_count} failed, {blocked_count} blocked, {already_done_count} already done

  Files changed:
    {list files from all story Dev Agent Records in this epic}

  {if failed_count > 0}
  Failed stories:
    {list each: "Story N.M: {title} — {failure reason from execution_log}"}
  Retry with: /sprint-exec --story=N.M
  {/if}

  {if verification_ran}
  Verification: {pass_count} passed, {concerns_count} concerns, {fail_count} failed
  {/if}

  {if not last_epic}
  Next: Epic {N+1}: {next_epic_title}
  Background code review dispatched -> SPRINT_DIR/reviews/epic-{N}-review.md
  {/if}
═══════════════════════════════════════════════════
```

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
