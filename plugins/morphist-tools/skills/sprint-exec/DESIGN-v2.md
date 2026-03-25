# Sprint-Exec v2: Thin OMC Dispatcher

## Design Goals

1. **Sprint-exec becomes a ~150-line dispatcher** that translates story specs → OMC task definitions
2. **OMC owns execution**: parallelism, persistence, verification, retry
3. **Extracted concerns** become small, focused, independently-invocable skills
4. **Zero "continue" prompts** when run under ralph/team (OMC Stop hook handles persistence)

---

## Architecture

```
User invokes: /sprint-exec [flags]
                    │
                    ▼
    ┌──────────────────────────────┐
    │   sprint-exec (thin dispatcher)   │
    │   ~150 lines                      │
    │                                   │
    │   1. Read phase-state.json        │
    │   2. Resolve scope (epic/story)   │
    │   3. Build task list from stories │
    │   4. Choose execution mode:       │
    │      --story → single executor    │
    │      --epic  → /team N:executor   │
    │      --auto  → /team ralph        │
    │   5. Dispatch to OMC              │
    │   6. Post-exec: update state      │
    └──────────────────────────────┘
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
   Single Story   /team       /team ralph
   (executor)    (per-epic)  (all epics)
        │           │           │
        ▼           ▼           ▼
   OMC executor  OMC team    OMC team+ralph
   reads story   workers     persistence loop
   file itself   read story  + verification
                 files       + retry
```

---

## New Sprint-Exec Flow (~150 lines)

### Section 1: Init (same as current, trimmed)
- Read readiness report, phase-state.json
- Parse arguments (same flags)
- Check execution status (resume logic)
- Update phase-state to "in-progress"
- **Write resume checkpoint** (new)

### Section 2: Resolve Scope
- Determine which stories to execute (same filtering logic)
- For each story: extract `{file_path, title, id, ac_count, decision_ids, status}` from frontmatter
- Build a **task manifest** — a list of task objects

### Section 3: Choose Execution Mode & Dispatch

| User Flag | Execution Mode | OMC Primitive |
|-----------|---------------|---------------|
| `--story=N.M` | Single story | `Agent(oh-my-claudecode:executor)` directly |
| `--next-story` | Single story | Same as above |
| `--epic=N` or default | One epic, parallel stories | `/team N:executor` with story tasks |
| `--auto` | All remaining epics | `/team ralph` with epic-sequential dispatch |
| `--full-auto` | All remaining, no stops | `/team ralph --full-auto` |

**Single story dispatch**: Same as current (direct Agent call with file path, not content).

**Epic dispatch via /team**: The dispatcher creates a team with N workers (one per story, capped by `--concurrency`). Each worker gets the story file path and the standard executor instructions. Team handles parallelism, completion tracking, and worker monitoring.

**All-epics via /team ralph**: Ralph wraps the team dispatch, providing persistence (Stop hook), architect verification between epics, and retry on failure. Epics still run sequentially — ralph dispatches one team per epic, waits for completion, runs post-epic hooks, then dispatches next.

### Section 4: Post-Execution Hooks

After each epic completes (callback from team/ralph):

1. **Update story statuses** — read Dev Agent Records from story files, apply done-validation
2. **Update epic status** in phase-state.json + epics.md
3. **Context checkpoint** — write resume_point
4. **Blocker check** — if architectural blockers found, invoke `/blocker-triage` (extracted skill)
5. **Verification** — invoke `/verify --epic=N` (already a separate skill)
6. **Background review** — invoke `/sprint-review --epic=N` in background (already separate)
7. **Progress report** — print epic completion summary

### Section 5: Completion
- Update phase-state.json: execution_status = "complete"
- Print final summary with next-step suggestions

---

## Extracted Skills

### `/blocker-triage` (NEW — extracted from sprint-exec 4g)

```yaml
---
name: blocker-triage
description: Analyze architectural blockers from failed stories, assess downstream impact, and present resolution options.
user-invocable: true
argument-hint: "[--epic=N] [--story=N.M] [--auto-accept]"
---
```

**What it does**: Reads story files for `blocker_type` fields, scans downstream stories for impact, presents the 4-option elicitation (swap/update-arch/accept/halt). ~80 lines.

**When invoked**: Automatically by sprint-exec post-epic if blockers detected. Also invocable standalone for manual triage.

### `/done-validate` (NEW — extracted from sprint-exec 4d steps 2-8)

```yaml
---
name: done-validate
description: Validate that executor agents actually produced work. Checks file existence, Dev Agent Record, and acceptance criteria coverage.
user-invocable: true
argument-hint: "[--epic=N] [--story=N.M]"
---
```

**What it does**: Reads story files post-execution, checks that listed files exist, Dev Agent Record is filled, AC boxes are checked. Updates story frontmatter with outcome. ~60 lines.

**When invoked**: Automatically by sprint-exec after each executor returns. Also invocable standalone to re-validate.

### `/exec-report` (NEW — extracted from sprint-exec 4i)

```yaml
---
name: exec-report
description: Generate epic or sprint execution progress report from phase-state.json and story files.
user-invocable: false
---
```

**What it does**: Reads execution state and story files, formats the progress report. ~40 lines. Non-user-invocable — called by sprint-exec internally.

---

## What's Removed from Sprint-Exec

| Current Section | Lines | Disposition |
|-----------------|-------|------------|
| Executor prompt (full story content) | 246-287 | **Removed** — executor reads file itself |
| Retry prompt (full story content) | 562-596 | **Removed** — same |
| Done-validation gate (steps 2-8) | 298-341 | **Extracted** → `/done-validate` |
| Blocker triage (full elicitation) | 383-467 | **Extracted** → `/blocker-triage` |
| Epic progress report | 499-529 | **Extracted** → `/exec-report` |
| Verification gate (inline dispatch) | 469-495 | **Replaced** — calls `/verify` directly |
| Background review dispatch | 531-539 | **Replaced** — calls `/sprint-review` directly |

## What's Kept in Sprint-Exec

| Concern | Rationale |
|---------|-----------|
| Init/phase-state reading | Sprint-specific state management |
| Argument parsing | User interface for execution control |
| Scope resolution | Story filtering logic is sprint-domain |
| OMC dispatch logic | The core integration point |
| Post-epic orchestration | Sequencing the hook calls |
| Resume checkpointing | Sprint-specific resume fidelity |
| Completion summary | User-facing output |

---

## OMC Stop Hook Integration

Sprint-exec itself does NOT need a Stop hook. Instead:

- **Single story**: No persistence needed — one executor call, done.
- **Epic via /team**: OMC's team skill has its own Stop hook persistence.
- **All epics via /team ralph**: Ralph's Stop hook provides 100-iteration persistence.

The user invokes sprint-exec, which dispatches to OMC. OMC's persistence keeps things running. Sprint-exec's role ends after dispatch + post-epic hooks.

For planning skills (sprint-plan, prd, refine) that also need continuation:
- Invoke within a ralph context: `ralph /sprint-plan`
- Or use `--fast` mode which auto-decides everything (no elicitation pauses)

---

## Migration Path

### Phase 1 (Done — Quick Wins)
- [x] Default `--stop-at` changed from `all` to `high`
- [x] Story content no longer injected into executor prompts
- [x] Context checkpointing after each epic
- [x] Design principles added to CLAUDE.md

### Phase 2 (Done — Extract Skills)
- [x] Create `/blocker-triage` skill (124 lines)
- [x] Create `/done-validate` skill (112 lines)
- [x] Create `/exec-report` skill (88 lines, non-user-invocable)
- [x] Update sprint-exec to call extracted skills instead of inline logic
- [x] Sprint-exec reduced from 644 → 481 lines (25% reduction)

### Phase 3 (Done — Thin Dispatcher Rewrite)
- [x] Rewrite sprint-exec as thin dispatcher (272 lines, down from 644)
- [x] Single story → direct executor dispatch
- [x] Epic → parallel executor dispatch with post-epic hooks
- [x] All-epics → sequential epic loop, benefits from OMC ralph persistence
- [x] Removed monolithic orchestration loop, inline blocker triage, inline progress reports
- [ ] Test with a real sprint

### Phase 4 (Future — Planning Autonomy)
- [ ] Document `ralph /sprint-plan` pattern for autonomous planning
- [ ] Evaluate `--fast` mode defaults for other skills (prd, refine)
- [ ] Consider extracting Decision Steering elicitations from sprint-plan into a separate concern
