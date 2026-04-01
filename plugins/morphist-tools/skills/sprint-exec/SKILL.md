---
name: sprint-exec
description: Execute validated sprint stories by dispatching to OMC execution primitives. Default: next epic. Use --full-auto for all remaining epics, --next-story for one story at a time.
user-invocable: true
argument-hint: "[--epic=N] [--story=N.M] [--next-story] [--full-auto] [--dry-run] [--concurrency=N] [--sprint=ID]"
---

# sprint-exec: Thin OMC Dispatcher

Reads validated story files, builds a task manifest, and dispatches to OMC execution primitives (executor, team, team+ralph). Sprint-exec owns *what* to execute; OMC owns *how*.

---

## 1. Initialization

### 1a. Sprint Resolution

Resolve the target sprint directory (`SPRINT_DIR`):
1. If `--sprint=<id>` was provided in `$ARGUMENTS`, set `SPRINT_DIR` = `.omc/sprint-plan/<id>/`
2. Else if `state_read` is available, read key `morphist.active_sprint`. If set, `SPRINT_DIR` = `.omc/sprint-plan/<value>/`
3. Else if `.omc/sprint-plan/current` symlink exists, `SPRINT_DIR` = `.omc/sprint-plan/current/`
4. Otherwise halt: "No active sprint found. Run `/sprint-plan` first, or pass `--sprint=<id>`."

Verify `SPRINT_DIR/phase-state.json` exists. If not, halt with the same message.

### 1b. Read Readiness Report

Read `SPRINT_DIR/readiness-report.md`.

If not found: `No readiness report found. Run /sprint-plan first.`

Check `validation_status` — must be `pass` or `pass-with-warnings`. If `pass-with-warnings`, display warnings before proceeding.

### 1c. Read Phase State

Read `SPRINT_DIR/phase-state.json`. Extract `sprint_number` and epic count from `SPRINT_DIR/epics.md`.

### 1d. Parse Arguments

| Flag | Default | Behavior |
|------|---------|----------|
| `--epic=N` | — | Execute only epic N |
| `--story=N.M` | — | Execute only story N.M |
| `--next-story` | off | Execute only the next unfinished story |
| `--auto` | off | All remaining epics, stops on epic failure/blockers |
| `--full-auto` | off | All remaining epics, never stops |
| `--stop-at=LEVEL` | `high` | Decision severity threshold. Values: `critical`, `high`, `medium`, `all` |
| `--dry-run` | off | Show execution plan without dispatching |
| `--concurrency=N` | unlimited | Max parallel stories within an epic |

**Default scope**: next incomplete epic (first epic whose `epic_status` is not `"done"`).

**Scope precedence**: `--story` > `--next-story` > `--epic` > default > `--auto` > `--full-auto`.

**Stop-at defaults by mode**: default/`--auto` = `high`, `--full-auto` = `critical`.

When a decision point is below the stop threshold, auto-resolve (accept partial, proceed, log with `"auto_resolved": true`).

### 1e. Check Existing Execution Status

- `"complete"`: Inform user, stop unless explicit scope flag provided
- `"in-progress"`: Resume — skip stories/epics already done (use `resume_point` if available)
- `"halted"`: Show halt reason, wait for confirmation, then resume

### 1f. Update Phase State

Add sibling fields (do NOT modify `current_phase`):

```json
{
  "execution_status": "in-progress",
  "execution_log": [],
  "epic_status": {},
  "resume_point": null
}
```

Initialize `epic_status` for in-scope epics to `"pending"`. Preserve existing state if resuming.

### 1g. Register with OMC State

If OMC state tools are available (non-blocking):
```
state_write({ mode: "sprint-exec", sprint: "{sprint_number}" })
```

---

## 2. Dry-Run Mode

If `--dry-run`: print execution plan showing each epic/story with status and disposition, then stop.

---

## 3. Load Stories & Build Task Manifest

### 3a. Read Epic Ordering

Read `SPRINT_DIR/epics.md` for the ordered list of epics and their stories.

### 3b. Resolve Story Files

Stories at: `SPRINT_DIR/stories/{epic}-{story}-{slug}.md`

For each story, read frontmatter to get `status`, `title`, `decisions`, and AC count.

### 3c. Filter by Scope

Apply scope filtering (same precedence as 1c). Skip `blocked` and `done` stories. Include `ready-for-dev` and `in-progress`.

### 3d. Build Task Manifest

For each eligible story, build a task object:

```json
{
  "story_id": "N.M",
  "epic": N,
  "title": "Story title",
  "file_path": "SPRINT_DIR/stories/N-M-slug.md",
  "ac_count": 5,
  "decision_ids": ["D-001", "D-003"],
  "status": "ready-for-dev"
}
```

Group tasks by epic for sequential epic dispatch.

---

## 4. Dispatch to OMC

Choose execution mode based on scope and dispatch accordingly.

### 4a. Single Story Mode (`--story` or `--next-story`)

Direct executor dispatch — no team needed.

If retrying a previously failed story, use `model="opus"` and include failure context from `execution_log`.

```python
Agent(
    subagent_type="oh-my-claudecode:executor",
    model="sonnet",  # or "opus" for retry
    prompt="""
You are implementing a story from the sprint plan.

Working directory: {working_directory}
Sprint directory: SPRINT_DIR/
Story file: {story_file_path}
Story: {story_title} ({story_id})
Acceptance criteria count: {ac_count}
Key architecture decisions: {decision_ids_csv}
{if retry}
Previous failure context:
{failure_notes_from_execution_log}
{/if}

Instructions:
1. Read the story file at {story_file_path} — it contains the full specification and acceptance criteria.
2. Implement every requirement in the story.
3. Follow the architecture decisions and file list specified.
4. When complete, fill in the "Dev Agent Record" section at the bottom of the story file with:
   - Agent Model Used
   - Completion Notes (what you did)
   - Any problems encountered
   - Blocker Type: `none` | `coding` | `library_incompatible` | `architecture_mismatch` | `dependency_missing` | `spec_unclear`
   - Blocker Detail (if not `none`): what specifically doesn't work and why
   - Files created or modified (File List)
5. Do not modify any section of the story file above the Dev Agent Record.
6. IMPORTANT: If you encounter a fundamental blocker, set the Blocker Type and Detail clearly — do NOT silently work around it.
7. If the story frontmatter contains a `tdd_tests` field, run those tests. All must pass. Report results in Completion Notes.
""",
)
```

After the executor returns, run post-story hooks (section 5). Done.

### 4b. Epic Mode (`--epic` or default — single epic)

Dispatch parallel story executors for one epic using OMC team.

**Pre-epic**: Update `epic_status` to `"in-progress"` in `phase-state.json` and `epics.md`.

**Dispatch**: For each story in the epic, dispatch in parallel (all simultaneously via Agent calls), respecting `--concurrency=N`:

```python
# For each story in the epic (parallel):
Agent(
    subagent_type="oh-my-claudecode:executor",
    model="sonnet",
    run_in_background=True,  # parallel dispatch
    prompt="... same executor prompt as 4a, with story-specific values ..."
)
```

As each executor returns, immediately run `/done-validate --story=N.M --update` to persist status.

After all stories complete, run post-epic hooks (section 5). Done.

### 4c. All-Epics Mode (`--auto` or `--full-auto`)

Process all remaining epics sequentially, with each epic's stories running in parallel.

For each incomplete epic (in order):
1. Run **Epic Mode** (4b) for the current epic
2. Run **Post-Epic Hooks** (section 5)
3. If blocker-triage halted execution, stop
4. If all stories failed and stop threshold met, present options or auto-proceed
5. Write context checkpoint, advance to next epic

**OMC persistence**: When running `--auto` or `--full-auto`, the orchestrator benefits from being invoked within an OMC persistence context (e.g., `ralph /sprint-exec --auto`). OMC's Stop hook prevents context exhaustion from requiring "continue" prompts. Sprint-exec itself does not implement a persistence loop — it delegates that to OMC.

---

## 5. Post-Epic Hooks

After each epic completes (called from 4b or 4c):

### 5a. Validate & Update Stories

Run `/done-validate --epic={N} --update` for any stories not yet validated (handles batch validation for stories that completed while others were still running).

### 5b. Update Epic Status

Determine epic outcome from story statuses:
- All `done`: epic = `"done"`
- Mix of `done` and `failed`: epic = `"partial"`
- All `failed`: epic = `"failed"`

Update `epic_status` in `phase-state.json` and `epics.md`.

### 5c. Context Checkpoint

Write `resume_point` to `phase-state.json`:
```json
{ "resume_point": { "last_completed_epic": N, "next_epic": N+1, "timestamp": "..." } }
```

### 5d. Blocker Triage

If any stories have architectural blockers:
- `--full-auto`: `/blocker-triage --epic={N} --auto-accept`
- Otherwise: `/blocker-triage --epic={N}`

If triage halts execution, stop here.

### 5e. Verification

If the epic had completed stories and scope is not `--next-story`:
- Invoke `/verify --epic={N}`
- Gate: PASS → proceed, CONCERNS → proceed, FAIL + above threshold → pause and ask

### 5f. Background Review

If epic had completed stories: dispatch `/sprint-review --epic={N}` with `run_in_background=True`.

### 5g. Inter-Epic Review Brief

Generate the review brief via the `exec-report` skill (internal). This produces the epic narrative summary, per-story results, cross-story reconciliation scan, manual test suggestions, and contextual next steps.

The review brief ends with a **pause for user input** (unless `--full-auto`). The user can continue, fix a story, run reconcile, audit, refine the next epic, annotate findings, or halt.

- `--auto`: Pause only if there are failed stories or reconciliation warnings. Otherwise auto-continue.
- `--full-auto`: Never pause (all output is informational only, auto-continues immediately).
- Default (single epic): Always pause.

### 5h. Notification

If OMC notifications configured, send epic completion notification (non-blocking).

---

## 6. Completion

After all in-scope epics are processed:

1. Update `phase-state.json`: `execution_status: "complete"`
2. Generate sprint completion report via `exec-report`

---

## 7. Known Limitations

- **Parallel story isolation**: Stories in the same epic cannot see each other's Dev Agent Records during execution. Cross-epic intelligence works because epics are sequential.
- **Re-execution**: Requires explicit `--story=N.M`. Will not overwrite `done` stories by default.
- **Blocked stories**: Always skipped. To unblock, manually set `status: ready-for-dev` and re-run.
- **OMC dependency**: Epic and all-epics modes use OMC executor agents. Single-story mode works without OMC team infrastructure.
