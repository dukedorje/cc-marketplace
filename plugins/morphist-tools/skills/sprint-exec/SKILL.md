---
name: sprint-exec
description: Execute validated sprint stories by dispatching executor agents. Default: next epic. Use --full-auto for all remaining epics, --next-story for one story at a time.
user-invocable: true
argument-hint: "[--epic=N] [--story=N.M] [--next-story] [--full-auto] [--dry-run] [--concurrency=N]"
---

# sprint-exec: Story Execution Orchestrator

You are the `sprint-exec` skill for the sprint-plan plugin. You read validated story files and dispatch executor agents to implement them.

---

## 1. Initialization

### 1a. Read Readiness Report

Read `.omc/sprint-plan/current/readiness-report.md`.

If the file does not exist, halt:
```
No readiness report found. Run /sprint-plan first to complete validation, or run /sprint-plan --continue to resume where you left off.
```

Check the `validation_status` field in the report. It must be `pass` or `pass-with-warnings`.

If `validation_status` is any other value (e.g., `fail`), halt:
```
Sprint has not passed validation. Run `/sprint-plan --continue` to resume, or `/sprint-plan --restart-from=validation` to re-run validation.
```

If `validation_status` is `pass-with-warnings`, display the warnings before proceeding:
```
Note: Sprint passed validation with warnings:
{list warnings from report}

Proceeding with execution...
```

### 1b. Read Phase State

Read `.omc/sprint-plan/current/phase-state.json`.

Extract:
- `sprint_number` (or `sprint` field): the current sprint identifier
- Epic count: determine from `current/epics.md` (count `## Epic` headings)

### 1c. Parse Arguments

Parse `$ARGUMENTS` for the following optional flags:

| Flag | Default | Behavior |
|------|---------|----------|
| `--epic=N` | — | Execute only epic N |
| `--story=N.M` | — | Execute only story N.M |
| `--next-story` | off | Execute only the next unfinished story (first `ready-for-dev` story in the first incomplete epic) |
| `--auto` | off | Execute ALL remaining epics sequentially. Stops on epic failure or architectural blockers (asks user). Does not stop on individual story failures within an epic. |
| `--full-auto` | off | Execute ALL remaining epics without stopping for ANY user confirmation. Auto-resolves failures (proceed to next epic) and blockers (accept partial). |
| `--stop-at=LEVEL` | varies | Decision severity threshold for pausing. Values: `critical`, `high`, `medium`, `all`. Default: `all` (default mode), `high` (`--auto`), `critical` (`--full-auto`). |
| `--dry-run` | off | Show execution plan without dispatching agents |
| `--concurrency=N` | unlimited | Max executor agents running simultaneously within an epic |

**Default scope** (no scope flags): execute only the **next incomplete epic** — the first epic whose `epic_status` is not `"done"`. This keeps execution incremental and controllable.

**Scope precedence**: `--story` > `--next-story` > `--epic` > default (next epic) > `--auto` (all remaining, stop on failure) > `--full-auto` (all remaining, never stop).

If `--story=N.M` is specified, `--epic` is inferred from the story number (N).

If `--next-story` is specified, determine the target by:
1. Find the first epic whose `epic_status` is not `"done"`
2. Within that epic, find the first story with `status: ready-for-dev` (or `in-progress` if resuming)
3. Execute only that single story
4. If no unfinished stories remain, report: `All stories are complete. Nothing to execute.`

### 1c-2. Decision Severity & Stop Threshold

During execution, decision points arise at blocker triage, verification failures, and executor-reported concerns. Each decision point has a severity:

| Severity | Examples |
|----------|----------|
| `critical` | Architectural blocker (`library_incompatible`, `architecture_mismatch`), all stories in epic failed, verification FAIL on a core story |
| `high` | Multiple story failures in an epic, verification FAIL on non-core stories, executor flagged `blocker_type: dependency_missing` |
| `medium` | Single story failure, verification CONCERNS, executor completion notes contain "workaround" or "TODO" |
| `low` | Verification all-PASS with minor notes, individual story retry succeeded |

**Stop behavior by mode**:

| Mode | Default `--stop-at` | Stops for | Auto-resolves |
|------|---------------------|-----------|---------------|
| Default (next epic) | `all` | Everything — every decision point pauses | Nothing |
| `--auto` | `high` | `critical` + `high` | `medium` + `low` (proceeds automatically) |
| `--full-auto` | `critical` | Only `critical` if explicitly set via `--stop-at=critical` | Everything by default |

`--stop-at` overrides the default for any mode. For example:
- `--auto --stop-at=critical` — runs all epics, only stops on critical issues
- `--full-auto --stop-at=high` — full auto but still stops on high-severity decisions
- `--stop-at=medium` — default next-epic mode but auto-resolves low-severity items

When a decision point is **below the stop threshold**, auto-resolve using the least-disruptive option:
- Blocker triage: accept partial (option 3)
- Verification failure: proceed with concerns logged
- All-epic failure: proceed to next epic

Log all auto-resolved decisions to `execution_log` with `"auto_resolved": true` and the severity level.

---

### 1d. Check Existing Execution Status

If `execution_status` already exists in `phase-state.json`:
- If `"complete"`: Inform the user and suggest targeted re-runs:
  ```
  This sprint has already been fully executed (execution_status: complete).
  Use --story=N.M to re-run a specific story, --epic=N for an epic, or --full-auto to re-run all.
  ```
  Stop here unless an explicit scope flag was provided (`--story`, `--epic`, or `--full-auto`).
- If `"in-progress"`: Resume from where execution left off — skip stories already in `execution_log` with status `done`. Also check `epic_status` to skip epics already marked `"done"`.
- If `"halted"`: Inform the user that execution was previously halted for replanning:
  ```
  Sprint execution was halted for replanning.
  Reason: {halt_reason from phase-state.json}
  Halted at: {halted_at from phase-state.json}

  Resume execution? Stories/epics already marked done will be skipped.
  Use --story=N.M to re-run a specific story, or confirm to resume all remaining.
  ```
  Wait for user confirmation. Then resume using the same logic as `"in-progress"` — skip stories already in `execution_log` with status `done`, and skip epics already marked `"done"` in `epic_status`.

### 1e. Update Phase State

**Important**: Do NOT modify `current_phase`. It must remain at `"validation"` (its Phase 5 value). Only add sibling fields.

Update `phase-state.json` by adding or updating these sibling fields alongside `current_phase`:

```json
{
  "current_phase": "validation",
  "stale_phases": [],
  "execution_status": "in-progress",
  "execution_log": [],
  "epic_status": {}
}
```

The `epic_status` object tracks per-epic completion state. Keys are epic numbers (as strings), values are `"pending"` | `"in-progress"` | `"done"` | `"partial"` | `"failed"`. Initialize all epics (in scope) to `"pending"`. Preserve existing `epic_status` if resuming.

Preserve the existing `execution_log` array if resuming a partial execution.

### 1f. Register with OMC State

If OMC state tools are available, register the session (non-blocking — skip if unavailable):
```
state_write({ mode: "sprint-exec", sprint: "{sprint_number}" })
```

---

## 2. Dry-Run Mode

If `--dry-run` was specified, print the execution plan and stop (do not dispatch any agents):

```
--- Sprint Exec Dry Run ---
Sprint: {sprint_number}

Execution Plan:
Epic 1: {epic_title}
  Story 1.1: {story_title} [ready-for-dev] → will execute
  Story 1.2: {story_title} [blocked] → will SKIP
  Story 1.3: {story_title} [done] → will SKIP (already done)

Epic 2: {epic_title}
  Story 2.1: {story_title} [ready-for-dev] → will execute
  Story 2.2: {story_title} [ready-for-dev] → will execute

Concurrency: {concurrency_limit or "unlimited"} stories per epic; epics run sequentially.
To execute: /sprint-exec (remove --dry-run)
```

Stop here if `--dry-run`.

---

## 3. Load Stories

### 3a. Read Epic Ordering

Read `current/epics.md` to determine:
- The ordered list of epics
- The stories belonging to each epic (by heading structure)

### 3b. Resolve Story Files

Stories are located at:
```
.omc/sprint-plan/current/stories/{epic}-{story}-{slug}.md
```
For example, story 2.3 "User Authentication" → `.omc/sprint-plan/current/stories/2-3-user-authentication.md`

Use Glob to find story files matching the pattern `current/stories/{epic}-*` for a given epic.

For each story, read its frontmatter to get current `status`.

### 3c. Filter by Scope

Apply scope filtering based on parsed arguments (in precedence order):
- If `--story=N.M`: only include that single story
- If `--next-story`: only include the next unfinished story (see section 1c for resolution logic)
- If `--epic=N`: only include stories in epic N
- If `--auto`: include all stories across all remaining epics (stops on epic failure/blockers)
- If `--full-auto`: include all stories across all remaining epics (never stops)
- Default (no scope flags): only include stories in the **next incomplete epic** — the first epic whose `epic_status` is not `"done"` in `phase-state.json`. If all epics are `"done"`, report completion and stop.

Within the filtered set, apply status filtering:
- Skip stories with `status: blocked` (log as skipped)
- Skip stories with `status: done` (already complete — log as skipped unless re-run requested)
- Include stories with `status: ready-for-dev` or `status: in-progress` (in-progress means a previous run was interrupted)

---

## 4. Execution Loop

Process epics SEQUENTIALLY. Within each epic, dispatch eligible stories in PARALLEL, respecting `--concurrency=N` if set. When concurrency is limited, dispatch up to N stories at once, and as each completes, dispatch the next eligible story until all stories in the epic are done.

### 4a. Pre-Epic: Update Epic Status

Before processing any stories in an epic:

1. Update `epic_status` in `phase-state.json`: set epic N to `"in-progress"`
2. Update `epics.md`: add or update a `Status: in-progress` line under the `## Epic N` heading (insert after the `### Goal` section if not present)

### 4b. Pre-Dispatch: Update Story Status

Before dispatching an executor for a story:

1. Read the story file
2. Update frontmatter:
   - `status: in-progress`
   - `started_at: {ISO 8601 timestamp}`
3. Write the updated story file back

### 4c. Dispatch Story Executors (Parallel Within Epic)

For each story in the current epic (all in parallel via simultaneous Agent calls):

```python
Agent(
    subagent_type="oh-my-claudecode:executor",
    model="sonnet",  # default; escalate to opus on retry after failure
    prompt="""
You are implementing a story from the sprint plan.

Working directory: {working_directory}
Sprint directory: .omc/sprint-plan/current/
Story file: {story_file_path}

Here is the full story specification:

{story_file_content}

Instructions:
1. Read the story carefully, including all acceptance criteria.
2. Implement every requirement in the story.
3. Follow the architecture decisions and file list specified.
4. When complete, fill in the "Dev Agent Record" section at the bottom of the story file with:
   - Agent Model Used
   - Completion Notes (what you did)
   - Any problems encountered
   - Blocker Type (if you could NOT complete the story, classify why):
     - `none` — completed successfully
     - `coding` — normal bug or implementation difficulty (retry-able)
     - `library_incompatible` — a specified library/framework does not support the required use case
     - `architecture_mismatch` — an architecture decision doesn't hold in practice
     - `dependency_missing` — depends on something that doesn't exist or isn't ready yet
     - `spec_unclear` — the story spec is ambiguous or contradictory
   - Blocker Detail (if Blocker Type is not `none`): 1-2 sentences explaining what specifically doesn't work and why
   - Files created or modified (File List)
5. Do not modify any section of the story file above the Dev Agent Record.
6. IMPORTANT: If you encounter a fundamental blocker (library doesn't work, architecture assumption is wrong),
   do NOT silently work around it or deliver a partial solution. Set the Blocker Type and Detail clearly
   so the orchestrator can surface it for a human decision.
7. If the story frontmatter contains a `tdd_tests` field, run those tests after implementation.
   All listed tests must pass before the story is considered done. Do not delete or weaken tests.
   Report test results in the Completion Notes.
""",
)
```

### 4d. Post-Completion: Update Story Status

**IMPORTANT**: Update story status **immediately** as each individual agent returns — do NOT batch updates until the end of the epic. This ensures that if the session is interrupted mid-epic, completed stories are already persisted as `done` and won't be re-executed on resume.

As each story executor completes (regardless of whether other stories in the epic are still running):

1. Read the story file back
2. Check if the Dev Agent Record section has been filled in
   - If the Dev Agent Record is still empty or missing content: write a note in the Completion Notes field: `"Agent completed without filling Dev Agent Record"`
3. **Done-validation gate** — verify the agent actually produced work:
   a. Extract the File List from the Dev Agent Record
   b. If the File List is empty, missing, or contains no entries:
      - Check git status for any uncommitted changes attributable to this story
      - If still no files found: outcome = `failed`
      - Add to Completion Notes: `"VALIDATION FAILED: Agent reported completion but created/modified zero files"`
   c. If the File List has entries, spot-check that at least one listed file actually exists on disk
      - If none of the listed files exist: outcome = `failed`
      - Add to Completion Notes: `"VALIDATION FAILED: Dev Agent Record lists files that do not exist"`
   d. Check that acceptance criteria have corresponding implementation:
      - Count the tasks/subtasks in the story — if all checkboxes are unchecked AND no files were modified, this is a false completion
4. Extract blocker classification from the Dev Agent Record:
   - Read the `Blocker Type` field (default to `"none"` if not present or if story succeeded)
   - Read the `Blocker Detail` field (if present)
   - Architectural blockers are: `library_incompatible`, `architecture_mismatch`, `dependency_missing`
5. Determine final outcome:
   - If executor reported failure or threw an error: outcome = `failed`
   - If done-validation gate failed (step 3): outcome = `failed`
   - If blocker_type is an architectural blocker (`library_incompatible`, `architecture_mismatch`, `dependency_missing`): outcome = `blocked`
     (The story cannot proceed due to external factors, not agent failure. This is distinct from `failed` which indicates the agent didn't complete its work.)
   - Otherwise: outcome = `done`
6. Update story frontmatter:
   - `status: {done|failed|blocked}`
   - `completed_at: {ISO 8601 timestamp}`
   - If validation failed: `validation_failure: {reason from step 3}`
   - If blocker_type is not `none`: `blocker_type: {value}`, `blocker_detail: {detail}`
7. Write the updated story file back **immediately**
8. Append to `execution_log` in `phase-state.json` **immediately** (read-modify-write):
   ```json
   {
     "story": "N.M",
     "status": "done|failed|blocked",
     "started_at": "...",
     "completed_at": "...",
     "validation_failure": null | "zero files created" | "listed files do not exist",
     "blocker_type": "none|coding|library_incompatible|architecture_mismatch|dependency_missing|spec_unclear",
     "blocker_detail": null | "Eden Treaty does not support streaming responses for SSE endpoints"
   }
   ```
9. Update `stories_completed` count in `phase-state.json`

Each story's status must be persisted to disk before processing the next agent result. This is the source of truth for resume (section 1d) and for `/sprint-review` to know which stories are reviewable.

Stories that fail done-validation are treated identically to executor failures — they appear in the epic progress report as failed and can be retried with `/sprint-exec --story=N.M`.

### 4e. Handle Failures Within an Epic

If an executor fails on a story:
- Mark `status: failed` on the story frontmatter
- Log the failure to `execution_log`
- Continue dispatching/completing remaining stories in the epic (do not abort the epic for one failure)

If ALL stories in an epic fail:
- If `--full-auto`: automatically proceed to the next epic (option 1). Log the auto-decision to `execution_log`:
  ```json
  { "type": "auto_proceed", "epic": N, "reason": "all stories failed, --full-auto active" }
  ```
- If `--auto`: halt and ask the user (same prompt as default below). `--auto` runs all epics but stops on epic-level failures.
- Otherwise (default or `--epic`), halt execution and ask the user:
  ```
  All stories in Epic {N} failed. Possible causes: missing dependencies, environment issues, or ambiguous story spec.

  Options:
  1. Proceed to Epic {N+1} anyway (stories from this epic will be missing)
  2. Abort execution (use /sprint-exec --epic=N to retry this epic)
  3. Retry individual stories with opus: /sprint-exec --story=N.M

  How would you like to proceed?
  ```
  Wait for user input.

### 4f. Post-Epic: Update Epic Status

After all stories in an epic have completed (or been skipped), determine and persist the epic's status:

1. Determine epic outcome:
   - If all stories are `done` (or `done` + skipped-as-already-done): epic status = `"done"`
   - If some stories are `done` and some `failed`: epic status = `"partial"`
   - If all stories `failed`: epic status = `"failed"`
   - If all stories were skipped (blocked or already done): epic status = `"done"`
2. Update `epic_status` in `phase-state.json`: set epic N to the determined status
3. Update `epics.md`: set `Status: {done|partial|failed}` under the `## Epic N` heading

This must happen **before** the progress report so that the persisted state is accurate if the session is interrupted.

### 4g. Blocker Triage (elicitation gate)

After updating epic status, check whether any failed stories in this epic have architectural blockers — i.e., `blocker_type` is `library_incompatible`, `architecture_mismatch`, or `dependency_missing`.

If there are **no architectural blockers**, skip to 4h (Verification Gate).

If there **are** architectural blockers:
- If `--full-auto`: automatically choose option 3 (accept partial) for each blocker. Log the auto-decision to `execution_log`:
  ```json
  { "type": "blocker_auto_accepted", "story": "N.M", "blocker_type": "...", "decision": "accepted_partial_auto" }
  ```
  Skip to 4h (Verification Gate).
- If `--auto`: this is a **blocking elicitation** — `--auto` stops on blockers so the user can decide.
- Otherwise (default), this is a **blocking elicitation** — do NOT proceed to the next epic until the user responds.

#### Step 1: Impact Analysis

For each architectural blocker, scan upcoming epics and stories for potential impact:
- Search remaining story files for references to the same library, API, or architecture decision
- Check epic dependency chains in `epics.md`
- Build a list of potentially affected stories

#### Step 2: Present Blocker Elicitation

```
══════════════════════════════════════════════════════
  ⚠ ARCHITECTURE BLOCKER — Decision Required
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
  3. Accept partial — Continue to Epic {N+1} as-is, address later
  4. Halt sprint — Stop execution for replanning

  Which approach? (If option 1 or 2, describe the alternative)
══════════════════════════════════════════════════════
```

Wait for user input.

#### Step 3: Act on Decision

Based on user response:

- **Option 1 (Swap approach)**: Record the user's alternative in the story's `blocker_detail` field as a resolution note. When the user later retries with `/sprint-exec --story=N.M`, the retry prompt (section 5) will include this context. Also update any affected stories in upcoming epics: append a note to their story file under a `## Blocker Resolution` heading (before the Dev Agent Record) so the executor knows about the change.

- **Option 2 (Update architecture)**: Read the relevant architecture decision from `current/architecture-decisions.md`. Present the current decision text and ask the user to confirm the revision. Update the decision in the file. Then apply the same story annotation as option 1.

- **Option 3 (Accept partial)**: Log the decision and continue. Add to `execution_log`:
  ```json
  {
    "type": "blocker_accepted",
    "story": "N.M",
    "blocker_type": "...",
    "decision": "accepted_partial",
    "rationale": "{user's explanation if provided}"
  }
  ```

- **Option 4 (Halt)**: Update `phase-state.json`:
  - Set `execution_status` to `"halted"`
  - Set `halted_at` to the current ISO 8601 timestamp
  - Set `halt_reason` to a brief summary of the blocker(s) that triggered the halt (e.g., `"Eden Treaty does not support SSE — stories 3.2, 4.1 affected"`)
  - Set `halt_blocker_stories` to an array of story IDs that triggered the halt (e.g., `["3.2", "4.1"]`)

  Print instructions for resuming after replanning. Stop execution.

After processing all architectural blockers for the epic, continue to 4h.

### 4h. Verification Gate

After blocker triage (or skipping it), run a quick independent verification of the epic's completed stories. This is equivalent to dispatching `/verify --epic={N}` inline.

Skip this step if:
- The epic had zero completed stories (all failed or blocked)
- The scope is `--next-story` (single-story execution doesn't trigger epic-level verification)

Dispatch a verifier agent (sonnet) with the story file lists, acceptance criteria, and architecture decisions. The verifier checks: file existence, basic import health, AC spot-check, and architecture compliance. See the `/verify` skill (section 3) for the full agent prompt.

**Display the verification summary** (see `/verify` section 4 for format) before the epic progress report.

**Gate behavior**:
- All stories PASS: proceed normally
- CONCERNS only: show results, proceed (concerns are non-blocking)
- Any FAIL + `--full-auto`: log failures to `verification_log` in `phase-state.json`, proceed
- Any FAIL + `--auto`: pause and ask (same as default — `--auto` stops on failures)
- Any FAIL + default: pause and ask:
  ```
  Verification found failures in Epic {N}. Fix before proceeding?

  1. Proceed to Epic {N+1} anyway
  2. Re-run failed stories: /sprint-exec --story=N.M
  3. Deep audit: /audit-story --epic={N}
  ```
  Wait for user input.

---

### 4i. Epic Progress Report

After all stories in an epic complete (or are skipped), output a prominent report:

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
  Background code review dispatched → .omc/sprint-plan/current/reviews/epic-{N}-review.md
  {/if}
═══════════════════════════════════════════════════
```

### 4j. Background Code Review (optional)

After reporting epic completion, dispatch the `sprint-review` skill in the background to review the epic's work while the next epic starts executing. This is non-blocking — execution continues immediately.

Skip this step if the epic had zero completed stories or if all stories failed.

This is equivalent to running `/sprint-review --epic={N}` in the background. Use the same agent dispatch defined in the sprint-review skill (section 3), with `run_in_background=True`.

Review results accumulate in `current/reviews/` and are available for the user to check at any time. They do NOT block execution. The user can also run `/sprint-review` manually at any point.

### 4k. Notification (optional)

If OMC notification tools are configured (via `/configure-notifications`), send a notification on epic completion. This is non-blocking — skip gracefully if notifications are not set up.

Message format:
```
Sprint {sprint_number} — Epic {N} complete: {epic_title}
{done_count}/{total_count} stories done, {failed_count} failed
Review: .omc/sprint-plan/current/reviews/epic-{N}-review.md
```

Also send a notification on full sprint execution completion (section 6).

---

## 5. Retry Behavior

When `--story=N.M` is used (typically for retrying a failed story):
- Use `model="opus"` for the executor agent instead of `"sonnet"`
- Include additional context in the prompt about the previous failure if available in the execution_log

```python
Agent(
    subagent_type="oh-my-claudecode:executor",
    model="opus",  # escalated model for retry
    prompt="""
You are retrying a story implementation that previously failed.

Working directory: {working_directory}
Sprint directory: .omc/sprint-plan/current/
Story file: {story_file_path}

Previous failure context:
{failure_notes_from_execution_log_if_available}

Here is the full story specification:

{story_file_content}

Instructions:
1. Read the story carefully, including all acceptance criteria.
2. Implement every requirement in the story.
3. Follow the architecture decisions and file list specified.
4. When complete, fill in the "Dev Agent Record" section at the bottom of the story file with:
   - Agent Model Used
   - Completion Notes (what you did and how you resolved any previous issues)
   - Any problems encountered
   - Blocker Type: `none` | `coding` | `library_incompatible` | `architecture_mismatch` | `dependency_missing` | `spec_unclear`
   - Blocker Detail: (if Blocker Type is not `none`) what specifically doesn't work and why
   - Files created or modified (File List)
5. Do not modify any section of the story file above the Dev Agent Record (except the Blocker Resolution section if present).
6. If a "Blocker Resolution" section exists in the story, read it carefully — it contains
   guidance from the user about an alternative approach to use instead of the original spec.
""",
)
```

---

## 6. Completion

After all epics (in scope) have been processed:

1. Update `phase-state.json`: set `execution_status: "complete"`
2. Write the final state back
3. Print the completion summary:

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
Run /sprint-retro to generate a retrospective from the execution results.
```

---

## 7. Known Limitations

- **Parallel story isolation**: Stories executing in parallel within an epic cannot reference each other's Dev Agent Records during execution. Each executor works from the story file as it existed at dispatch time. Cross-epic intelligence works correctly because epics are sequential — by the time Epic 2 starts, all Epic 1 Dev Agent Records are written and available.
- **Re-execution**: Re-running `/sprint-exec` on already-done stories requires explicitly passing `--story=N.M`. The skill will not overwrite `done` stories by default.
- **Blocked stories**: Stories with `status: blocked` are always skipped. To unblock a story, manually update its frontmatter status to `ready-for-dev` and re-run `/sprint-exec --story=N.M`.
