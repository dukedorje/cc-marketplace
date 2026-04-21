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

Resolve the target sprint directories:
1. If `--sprint=<id>` was provided in `$ARGUMENTS`, set `STATE_DIR` = `.omc/sprint-plan/<id>/`
2. Else if `state_read` is available, read key `morphist.active_sprint`. If set, `STATE_DIR` = `.omc/sprint-plan/<value>/`
3. Else if `.omc/sprint-plan/current` symlink exists, `STATE_DIR` = `.omc/sprint-plan/current/`
4. Otherwise halt: "No active sprint found. Run `/sprint-plan` first, or pass `--sprint=<id>`."

Verify `STATE_DIR/phase-state.json` exists. Read it and set `SPEC_DIR` from the `spec_dir` field.
Verify `SPEC_DIR` exists. If not, halt: "Sprint spec directory not found at {spec_dir}. Sprint may need re-initialization."

### 1b. Read Readiness Report

Read `SPEC_DIR/readiness-report.md`.

If not found: `No readiness report found. Run /sprint-plan first.`

Check `validation_status` — must be `pass` or `pass-with-warnings`. If `pass-with-warnings`, display warnings before proceeding.

### 1c. Read Phase State

Read `STATE_DIR/phase-state.json`. Extract `sprint_number` and epic count from `SPEC_DIR/epics.md`.

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

Read `SPEC_DIR/epics.md` for the ordered list of epics and their stories.

### 3b. Resolve Story Files

Check for enriched story files at: `SPEC_DIR/stories/{epic}-{story}-{slug}.md`

If enriched story files exist, read frontmatter to get `status`, `title`, `decisions`, `test_tier`, and AC count.

If no enriched story files exist (Phase 4 was skipped — the default workflow), read story stubs from `SPEC_DIR/epics.md` instead. Extract `test_tier`, `complexity`, `decisions`, and ACs from the stub format.

### 3c. Filter by Scope

Apply scope filtering (same precedence as 1c). Skip `blocked` and `done` stories. Include `ready-for-dev` and `in-progress`.

### 3d. Build Task Manifest

For each eligible story, build a task object:

```json
{
  "story_id": "N.M",
  "epic": N,
  "title": "Story title",
  "file_path": "SPEC_DIR/stories/N-M-slug.md",
  "source": "enriched|stub",
  "ac_count": 5,
  "test_tier": "yolo|smoke|thorough",
  "decision_ids": ["D-001", "D-003"],
  "complexity": "small|medium|large",
  "status": "ready-for-dev"
}
```

Group tasks by epic for sequential epic dispatch.

### 3e. Pre-Execution Story-Writing Check

Phase 4 "write stories" is the sprint-plan step that turns one-line story stubs into full implementation specs. If the user skipped it at plan time but the sprint now contains stories that would benefit, warn before dispatching.

For each epic about to execute, scan its stories for story-writing candidates. A story is flagged if ALL of:
- No written story file exists (`source: "stub"`)
- Complexity is `large` OR references ≥3 architecture decisions OR `test_tier` is `thorough`

If candidates are found and not in `--full-auto` mode, prompt:

```
{K} of {N} stories in epic {E} are flagged for story-writing (stub alone may not be enough):
  ⚠ Story {N.M}: {title} — {reason}

[write]  → Run Phase 4 for flagged stories now, then dispatch (~30s/story)
[skip]   → Execute from stubs as-is (default in 10s)
```

Auto-skip after 10 seconds if no response. In `--full-auto` mode, always skip (full-auto respects the planning-time decision without re-prompting).

If the user chooses `[write]`, run Phase 4 "write stories" for only the flagged stories, then proceed to dispatch.

**Back-compat**: previous versions of this skill called this step the "Enrichment Check" and used `[enrich]`/`[skip]` labels. Both terminologies reference the same mechanism.

---

### 3f. Pre-Execution Spike Gate

Before dispatching any story that references a decision derived from an ultraresearch synthesis, check whether the underlying hypotheses have been empirically validated.

**Procedure** for each story in scope:

1. For each decision ID referenced in the story's frontmatter (`decisions: [D-NNN, …]`), open `SPEC_DIR/architecture-decisions.md` and read the decision block.
2. Look for citations of the form `synthesis.md §{hypothesis-id}` or `[RESEARCH-BACKED]` tag combined with a `research_artifacts` reference in `phase-state.json`.
3. For each cited hypothesis, open the research artifact (`{path}/synthesis.md`) and look for the hypothesis's structured metadata:
   ```yaml
   - id: H3
     confidence: low | medium | high
     verification: empirically-tested | inferred-from-source | docs-only
     version-pinned: <library>@<version>
     spike-required: true | false
     spike-artifact: docs/spikes/<id>/finding.md  # present only if a spike has been run
   ```
4. A hypothesis is **unvalidated** if `spike-required: true` AND `spike-artifact` is missing/empty, OR if `verification != empirically-tested` AND `confidence != high`.

If any referenced story has an unvalidated hypothesis, halt before dispatch:

```
Spike gate: story {N.M} references decision {D-NNN} which cites hypothesis {H-id}:

  claim:         "{hypothesis claim, trimmed}"
  confidence:    {level}
  verification:  {state}
  version-pinned:{library@version, or "unpinned"}

This hypothesis has NOT been empirically validated. Landing the story before
verifying risks the sprint-004 failure mode (D-002 v1 → v3 journey).

Options:
  [spike]     Dispatch /spike --hypothesis={H-id} — time-boxed 30–120m  ← recommended
  [override]  Proceed anyway (logs a warning to execution_log + replan-log)
  [skip]      Skip this story for now, continue with others

Default behavior:
  --auto        → prompt (like default)
  --full-auto   → [spike] (never silently roll forward on unvalidated research)
  --stop-at=critical+user-approved decisions → [override] (user has already accepted risk)
```

When the user chooses `[spike]`:
1. Invoke the `/spike` skill with the hypothesis prompt (extracted from the synthesis's `spike-prompt` field, or constructed from the claim if absent).
2. `/spike` produces `docs/spikes/{H-id}/finding.md` with a go/no-go verdict and an evidence artifact (code, screenshot, log excerpt, test result).
3. If the spike returns **go**: update the synthesis.md to set `spike-artifact: docs/spikes/{H-id}/finding.md` and bump `confidence: high` + `verification: empirically-tested`. Resume dispatch.
4. If the spike returns **no-go**: halt + invoke `/replan --decision={D-NNN} --reason="spike disproved H-id"`. The replan flow will revise the decision and re-trigger sprint-exec.

When the user chooses `[override]`:
1. Append a spike-override entry to `phase-state.json → execution_log`:
   ```json
   { "ts": "…", "event": "spike-gate-override", "story": "N.M", "decision": "D-NNN", "hypothesis": "H-id", "reason": "user-accepted-risk" }
   ```
2. Record the same in `SPEC_DIR/replan-log.md` under a "Spike Overrides" section (create section if absent).
3. Proceed with dispatch.

**Rationale** (from sprint-004 retro): an ultraresearch synthesis that prescribes a tactical recipe can still be wrong if the underlying hypothesis wasn't empirically tested. The D.R synthesis in sprint-004 was directionally correct but its specific recipes failed when landed — the team burned cycles debugging before a spike surfaced the real pattern. This gate forces a small spike before the big landing, inverting the cost.

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
Spec directory: SPEC_DIR/
State directory: STATE_DIR/
Story file: {story_file_path}
Story: {story_title} ({story_id})
Acceptance criteria count: {ac_count}
Key architecture decisions: {decision_ids_csv}
{if retry}
Previous failure context:
{failure_notes_from_execution_log}
{/if}

Test tier: {test_tier}
{if epic_learnings}
Learnings from previous epics:
{epic_learnings}
{/if}

Instructions:
1. Read the story file at {story_file_path} — it contains the full specification and acceptance criteria.
2. If no enriched story file exists at SPEC_DIR/stories/, read the story stub from SPEC_DIR/epics.md instead. Also read SPEC_DIR/architecture-decisions.md for referenced decisions.
3. Implement every requirement in the story.
4. Follow the architecture decisions specified.
5. Testing approach based on test_tier:
   - "yolo": No tests required. Just make it work — verify it builds/runs.
   - "smoke": Write happy-path tests for key functionality. Skip edge cases and error paths.
   - "thorough": Full TDD — write tests first, cover happy paths, edge cases, and error handling.
6. When complete, fill in the "Dev Agent Record" section at the bottom of the story file with:
   - Agent Model Used
   - Completion Notes (what you did)
   - Any problems encountered
   - Blocker Type: `none` | `coding` | `library_incompatible` | `architecture_mismatch` | `dependency_missing` | `spec_unclear`
   - Blocker Detail (if not `none`): what specifically doesn't work and why
   - Files created or modified (File List)
7. Do not modify any section of the story file above the Dev Agent Record.
8. IMPORTANT: If you encounter a fundamental blocker, set the Blocker Type and Detail clearly — do NOT silently work around it.
9. If the story frontmatter contains a `tdd_tests` field, run those tests regardless of test_tier. Report results in Completion Notes.
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

### 5g. Inter-Epic Huddle

Generate the huddle via the `exec-report` skill (internal). This produces:
1. Epic narrative summary (what was built)
2. Per-story results with cross-story reconciliation
3. Learnings for the next epic (extracted from Dev Agent Records)
4. User feedback prompt (freeform input)
5. Next steps menu: go, enrich, redirect, replan, yolo, hold, fix, reconcile

The huddle persists learnings to `phase-state.json` under `epic_learnings.{N}`. These learnings are injected into executor prompts for the next epic (via the `{epic_learnings}` template variable in section 4a).

**Huddle behavior by mode:**
- **Default**: Always huddle. This is where the human steers.
- **`--auto`**: Huddle only if there are failed stories, reconciliation warnings, or the user previously chose `[redirect]`. Otherwise auto-continue with `[go]`.
- **`--full-auto`**: Never huddle. Learnings are still extracted and persisted but displayed as informational output only.
- **`huddle_mode: "skip"` in phase-state.json**: User chose `[yolo]` at a previous huddle — skip remaining huddles, auto-continue.

**Passing learnings forward:** When dispatching the next epic's stories (section 4), read `epic_learnings` from `phase-state.json` and include all learnings from completed epics in the executor prompt. This is the forward intelligence that replaces sequential enrichment.

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
