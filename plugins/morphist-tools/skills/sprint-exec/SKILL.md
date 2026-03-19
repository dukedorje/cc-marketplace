---
name: sprint-exec
description: Execute validated sprint stories by dispatching executor agents. Processes epics sequentially, stories within each epic in parallel. Supports --epic, --story, and --dry-run flags.
user-invocable: true
argument-hint: "[--epic=N] [--story=N.M] [--dry-run]"
---

# sprint-exec: Story Execution Orchestrator

You are the `sprint-exec` skill for the sprint-plan plugin. You read validated story files and dispatch executor agents to implement them.

---

## 1. Initialization

### 1a. Read Readiness Report

Read `.omc/sprint-plan/current/readiness-report.md`.

If the file does not exist, halt:
```
No readiness report found. Run /sprint-plan first to complete validation, or run /sprint-plan --restart-from=validation.
```

Check the `validation_status` field in the report. It must be `pass` or `pass-with-warnings`.

If `validation_status` is any other value (e.g., `fail`), halt:
```
Sprint has not passed validation. Run `/sprint-plan --restart-from=validation` first.
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
| `--epic=N` | all epics | Execute only epic N |
| `--story=N.M` | all stories in scope | Execute only story N.M |
| `--dry-run` | off | Show execution plan without dispatching agents |

If `--story=N.M` is specified, `--epic` is inferred from the story number (N).

### 1d. Check Existing Execution Status

If `execution_status` already exists in `phase-state.json`:
- If `"complete"`: Inform the user and ask whether to re-run:
  ```
  This sprint has already been fully executed (execution_status: complete).
  Use --story=N.M to re-run a specific story, or confirm to re-run all.
  ```
- If `"in-progress"`: Resume from where execution left off — skip stories already in `execution_log` with status `done`.

### 1e. Update Phase State

**Important**: Do NOT modify `current_phase`. It must remain at `"validation"` (its Phase 5 value). Only add sibling fields.

Update `phase-state.json` by adding or updating these sibling fields alongside `current_phase`:

```json
{
  "current_phase": "validation",
  "stale_phases": [],
  "execution_status": "in-progress",
  "execution_log": []
}
```

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

Concurrency: stories within each epic run in parallel; epics run sequentially.
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

Apply scope filtering based on parsed arguments:
- If `--story=N.M`: only include that single story
- If `--epic=N`: only include stories in epic N
- Otherwise: all stories

Within the filtered set, apply status filtering:
- Skip stories with `status: blocked` (log as skipped)
- Skip stories with `status: done` (already complete — log as skipped unless re-run requested)
- Include stories with `status: ready-for-dev` or `status: in-progress` (in-progress means a previous run was interrupted)

---

## 4. Execution Loop

Process epics SEQUENTIALLY. Within each epic, dispatch ALL eligible stories in PARALLEL.

### 4a. Pre-Dispatch: Update Story Status

Before dispatching an executor for a story:

1. Read the story file
2. Update frontmatter:
   - `status: in-progress`
   - `started_at: {ISO 8601 timestamp}`
3. Write the updated story file back

### 4b. Dispatch Story Executors (Parallel Within Epic)

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
   - Files created or modified (File List)
5. Do not modify any section of the story file above the Dev Agent Record.
""",
)
```

### 4c. Post-Completion: Update Story Status

After each story executor completes:

1. Read the story file back
2. Check if the Dev Agent Record section has been filled in
   - If the Dev Agent Record is still empty or missing content: write a note in the Completion Notes field: `"Agent completed without filling Dev Agent Record"`
3. Determine outcome:
   - If executor reported failure or threw an error: outcome = `failed`
   - Otherwise: outcome = `done`
4. Update story frontmatter:
   - `status: {done|failed}`
   - `completed_at: {ISO 8601 timestamp}`
5. Write the updated story file back
6. Append to `execution_log` in `phase-state.json`:
   ```json
   {
     "story": "N.M",
     "status": "done",
     "started_at": "...",
     "completed_at": "..."
   }
   ```

### 4d. Handle Failures Within an Epic

If an executor fails on a story:
- Mark `status: failed` on the story frontmatter
- Log the failure to `execution_log`
- Continue dispatching/completing remaining stories in the epic (do not abort the epic for one failure)

If ALL stories in an epic fail:
- Halt execution and ask the user:
  ```
  All stories in Epic {N} failed. Possible causes: missing dependencies, environment issues, or ambiguous story spec.

  Options:
  1. Proceed to Epic {N+1} anyway (stories from this epic will be missing)
  2. Abort execution (use /sprint-exec --epic=N to retry this epic)
  3. Retry individual stories with opus: /sprint-exec --story=N.M

  How would you like to proceed?
  ```
  Wait for user input.

### 4e. Epic Progress Report

After all stories in an epic complete (or are skipped), report:

```
--- Epic {N} Complete: {epic_title} ---
Stories: {done_count}/{total_count} done, {failed_count} failed, {blocked_count} skipped (blocked), {already_done_count} skipped (already done)
{if failed_count > 0}
Failed stories: {list of story IDs}
Retry with: /sprint-exec --story=N.M (uses opus model on retry)
{/if}
{if not last_epic}
Next: Epic {N+1}: {next_epic_title}
{/if}
```

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
   - Files created or modified (File List)
5. Do not modify any section of the story file above the Dev Agent Record.
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
