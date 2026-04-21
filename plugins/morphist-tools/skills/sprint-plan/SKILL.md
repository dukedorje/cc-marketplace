---
name: sprint-plan
description: Multi-phase sprint planning workflow that transforms product ideas into implementation-ready user stories with architecture decisions, requirements expansion, and adversarial validation
user-invocable: true
argument-hint: "[product-idea-or-prd-path] [--fast] [--thorough] [--auto] [--step] [--sprint-size=SIZE] [--write-stories] [--skip-stories] [--continue[=phase]] [--restart-from=phase] [--sprint=ID] [--product=<name>]"
---

# Sprint Plan — Thin Orchestrator

Coordinates a multi-phase planning workflow. Each phase is defined in its own file under `${CLAUDE_SKILL_DIR}/phases/`. This orchestrator handles argument parsing, initialization, phase routing, state management, and inter-phase pauses. Phase files are authoritative for agent prompts and dispatch logic.

**OMC persistence**: Sprint-plan automatically activates ralph state on initialization (step 2c), so the OMC Stop hook handles continuation across phases without manual "continue" prompts. The ralph state is deactivated on completion. No explicit `ralph` wrapping is needed.

---

## 1. Argument Parsing

Parse `$ARGUMENTS` for:

1. **Product input**: Remaining text after flags. May be a file path (PRD, brief), inline text, or empty (prompt: "What would you like to build?").

2. **Flags**:

| Flag | Effect |
|------|--------|
| `--fast` | Single-pass, no RALPLAN-DR, AUTONOMOUS steering, implies `--auto`, skip story-writing |
| `--thorough` | Full consensus, GUIDED steering, refinement loops, write full stories for all |
| `--write-stories` | Force Phase 4 story-writing regardless of size gate |
| `--skip-stories` | Force-skip Phase 4 even if size gate would recommend it |
| `--auto` | No inter-phase pauses (only Decision Steering can pause) |
| `--step` | Pause after every phase |
| `--sprint-size=SIZE` | `focused` (3-8 stories), `standard` (8-18, default), `ambitious` (15-30) |
| `--skip-ux` | Skip Phase 1.5 even if frontend detected |
| `--continue[=phase]` | Resume from next incomplete phase (or after specified phase) |
| `--restart-from=phase` | Re-run a phase and mark all downstream stale |
| `--sprint=ID` | Target a specific sprint (e.g., `--sprint=sprint-002`). For `--continue`/`--restart-from`, operates on this sprint instead of the most recent. For new sprints, ignored. |
| `--product=<name>` | Associate this sprint with a product dimension. Writes `product` field to phase-state.json and requirements.md frontmatter. If omitted and the product input is a PRD path under `docs/products/{name}/`, auto-infer the product from the path. |

**Precedence**: `--thorough` > `--fast`. `--restart-from` > `--continue`. `--step` > `--auto`. `--fast` implies `--auto` + `--skip-stories`. `--thorough` implies `--write-stories`. `--write-stories` > `--skip-stories` (explicit opt-in wins).

**Default mode** (neither `--fast` nor `--thorough`): Single-pass architecture and epic design, GUIDED steering at key decision points. Phase 4 "write stories" is **size-gated** by default:

- ≤ 5 stories AND no `large`/`thorough`/3+-decision stories → skip (stubs fine)
- 6–15 stories OR any flagged story → prompt the user at Phase 3.5 (elaborated below)
- 16+ stories OR any `sprint-size=ambitious` → default to write-all (user can opt out with `--skip-stories`)

This is the huddle-driven workflow: plan light when the sprint is small, write proper stories when complexity warrants, execute an epic, huddle, repeat.

---

## 2. Initialization

### 2a. New Sprint (skip if `--continue` or `--restart-from`)

1. Determine sprint number: find highest `sprint-NNN/` in `.omc/sprint-plan/`, increment (or start at `sprint-001`).
2. **Derive sprint slug**: From the product input, generate a short kebab-case slug (e.g., `auth-overhaul`, `api-pagination`). If no input yet, use `planning` as a placeholder (can be renamed later). Combine: `{NNN}-{slug}` (e.g., `001-auth-overhaul`).
3. Create directories:
   - `STATE_DIR` = `.omc/sprint-plan/sprint-{NNN}/` — operational state (gitignored)
   - `SPEC_DIR` = `docs/sprints/{NNN}-{slug}/stories/` (creates stories/ and parent)
4. Update symlink: `rm -f .omc/sprint-plan/current && ln -s sprint-{NNN} .omc/sprint-plan/current`.
5. Write `.omc/sprint-plan/AGENTS.md` — static pointer: "Sprint specs in `docs/sprints/{NNN}-{slug}/`. Sprint state in `.omc/sprint-plan/current/`".

### 2b. Initialize phase-state.json

```json
{
  "sprint": "sprint-{NNN}",
  "spec_dir": "docs/sprints/{NNN}-{slug}/",
  "product": null,
  "mode": "default|thorough|fast",
  "active": true,
  "current_phase": "discovery",
  "steering_mode": "GUIDED|AUTONOMOUS",
  "significance_calibration": [],
  "decisions_log": [],
  "sprint_size": null,
  "sprint_scope": null,
  "refinement_loops": { "requirements_architecture": { "count": 0, "max": 3 } },
  "refine_passes": {},
  "stale_phases": [],
  "epics_count": 0,
  "stories_total": 0,
  "stories_enriched": 0,
  "validation_status": "pending"
}
```

Set `steering_mode` to `GUIDED` if thorough, `AUTONOMOUS` if fast.

### 2c. OMC State & Auto-Continuation (optional)

If `state_write` available:
1. Register `{ mode: "sprint-plan", active: true, current_sprint: "sprint-{NNN}" }`. Skip if unavailable.
2. Write session-scoped sprint binding: key `morphist.active_sprint`, value `sprint-{NNN}`. This allows other skills in the same session to auto-resolve to this sprint without relying on the `current` symlink.
3. **Activate auto-continuation**: Write a ralph state file to `.omc/state/ralph-state.json` (or session-scoped `.omc/state/sessions/{sessionId}/ralph-state.json` if session ID is available):
   ```json
   {
     "active": true,
     "iteration": 1,
     "max_iterations": 20,
     "started_at": "{ISO 8601}",
     "prompt": "/sprint-plan --continue",
     "session_id": "{sessionId if available}",
     "project_path": "{absolute working directory path}",
     "source": "sprint-plan"
   }
   ```
   This piggybacks on OMC's existing Stop hook to auto-continue if Claude stops mid-planning. The `source: "sprint-plan"` field distinguishes it from user-initiated ralph loops.

### 2d. Handle --continue

1. If `--sprint=<id>` provided, use `.omc/sprint-plan/<id>/`. Otherwise find most recent `sprint-NNN/`.
2. Read `phase-state.json`. Set `STATE_DIR` = `.omc/sprint-plan/<resolved>/`. Set `SPEC_DIR` from `phase-state.json`'s `spec_dir` field.
3. Without phase arg: resume from next phase after `current_phase`.
4. With phase arg (`--continue=<phase>`): verify artifact exists, resume from next phase.
5. Update symlink and `current_phase`. Register in OMC session state (step 2c). Report: "Resuming sprint {NNN} from {phase}."

### 2e. Handle --restart-from

1. If `--sprint=<id>` provided, use `.omc/sprint-plan/<id>/`. Otherwise find most recent `sprint-NNN/`.
2. Read `phase-state.json`. Set `STATE_DIR` = `.omc/sprint-plan/<resolved>/`. Set `SPEC_DIR` from `phase-state.json`'s `spec_dir` field.
3. Mark specified phase + all downstream as stale.
4. Update symlink and `current_phase`. Register in OMC session state (step 2c). Report: "Restarting from {phase}. Downstream marked stale."

---

## 3. Phase Orchestration

Execute phases sequentially. For each phase:
1. Read phase instruction file from `${CLAUDE_SKILL_DIR}/phases/`.
2. Load previous phase's output artifact (context shedding — read from disk, not memory).
3. Dispatch agent(s) per the phase file. Include in the agent prompt: **"For this execution, `SPEC_DIR` = `{spec path}` and `STATE_DIR` = `{state path}`."** Phase files use `SPEC_DIR` for committed spec artifacts and `STATE_DIR` for operational state.
4. Write spec outputs (requirements, architecture-decisions, epics, stories, discovery) to `SPEC_DIR/`. Write state (phase-state, logs) to `STATE_DIR/`.
5. Update `current_phase` and metrics in `STATE_DIR/phase-state.json`.
6. Run Decision Steering if active for this phase.
7. Inter-phase summary & pause (if applicable).

### Phase Chain

| Phase | File | Pause? | Steering | Output |
|-------|------|--------|----------|--------|
| 0: Discovery | `phase-0-discovery.md` | step only | Dormant | `discovery.md` |
| 1: Requirements | `phase-1-requirements.md` | **Yes** | Active | `requirements.md` |
| 1B: Scoping | `phase-1b-sprint-scoping.md` | **Yes** | Active (CRITICAL) | `sprint-scope.md` |
| 1.5: UX Design | `phase-1.5-ux-design.md` | Conditional | Active | `ux-design.md` |
| 2A: Architecture | `phase-2a-architecture.md` | **Yes** | **Max Active** | `architecture-decisions.md` |
| 2B: Epic Design | `phase-2b-epic-design.md` | step only | Active | `epics.md` |
| 3: Stories | `phase-3-story-decomposition.md` | step only | Dormant | Updates `epics.md` |
| 3.5: Write-Stories Gate | *(inline — see Phase-Specific Notes)* | **Yes** | Dormant | user choice → feeds Phase 4 or skips |
| 4: Write Stories | `phase-4-write-stories.md` | step only | Dormant | `stories/*.md` | **Size-gated** — skipped if ≤5 simple stories, prompted at 6–15, default-write at 16+ |
| 5: Validation | `phase-5-validation.md` | Informational | Dormant | `readiness-report.md` |

### Phase-Specific Notes

- **Phase 1.5 trigger**: Run if `has_frontend: true` AND `has_ux_artifacts: false` AND not `--skip-ux` AND not `--fast`. Otherwise skip (`ux_design_phase: "skipped"`).
- **Phase 2A refinement loop** (thorough only): If decisions create new requirements, loop back to Phase 1 (incremental), then re-run 2A. Max 3 iterations via `refinement_loops.requirements_architecture.count`.
- **Phase 2B**: Single planner pass with inline validation (default and fast modes). Full RALPLAN-DR consensus only in `--thorough` mode.
- **Phase 3**: All epics decompose in parallel (each planner gets full `epics.md` for context). BDD writers fire in parallel across all stories. Post-decomposition reconciliation scan catches cross-epic issues. Each story gets a `test_tier` (yolo/smoke/thorough) based on risk — defaults to smoke.

- **Phase 3.5 — Write-Stories Gate (size-gated, inline prompt)**: After Phase 3 decomposition, count stories and flag risky ones. "Flagged" = `complexity=large` OR `test_tier=thorough` OR references ≥3 architecture decisions OR references any decision marked `research-backed` without a prior spike artifact (see `/spike` skill).

  Present the user with:

  ```
  Sprint {NNN} decomposition: {N} stories across {M} epics.

  Flagged for story-writing (stub alone may not be enough):
    {list each flagged story with reason}

  Write full stories before execution?

  [write-all]     Expand every story as a full spec (~30s each × {N} stories)
  [write-flagged] Expand only the {K} flagged stories (recommended)  ← default for 6–15 stories
  [keep-stubs]    Proceed with stubs (risky if flagged stories exist)
  [iterate-plan]  Go back to Phase 2B or 3 before committing

  Default (auto-continues in 15s): [write-flagged] if any flagged; else [keep-stubs]
  ```

  Override behavior:
  - `--fast` / `--skip-stories` → skip this gate, keep stubs
  - `--thorough` / `--write-stories` → skip this gate, [write-all]
  - `--auto` / `--full-auto` → apply the default without prompting
  - `--step` → always prompt regardless of size

  Record user choice in `phase-state.json`: `stories_gate_decision: "write-all"|"write-flagged"|"keep-stubs"` with `stories_gate_reason` noting the auto-default vs. explicit override.

- **Phase 4**: Runs only if Phase 3.5 gate chose `write-all` or `write-flagged`. When `write-flagged`, fan out only for flagged stories (not all). Unflagged stories ship as stubs.

- **Phase 5**: Sonnet verifier only (no opus critic). Auto-fix loop (max 2 iterations). Readiness report always shown.

### Inter-Phase Summary & Pause

**Pause modes**: Default pauses at decision points (Phases 1, 1B, 2A, 5). `--step` pauses after every phase. `--auto`/`--fast` never pause (only Decision Steering elicitations in GUIDED mode can pause).

When pausing, show: phase name, artifact path, 2-4 bullet summary of outputs, key decisions made, 1-3 "things to consider" (high-significance decisions, open questions, health flags), and options (continue, review, edit, `/refine`, `--restart-from`).

Wait for user input before proceeding.

---

## 4. Decision Steering

Read `${CLAUDE_SKILL_DIR}/phases/decision-steering.md` for the complete system.

**Orchestrator rules**:
- `thorough` → GUIDED. `fast` → AUTONOMOUS.
- GUIDED: HIGH/CRITICAL decisions trigger user elicitation.
- AUTONOMOUS: all decisions auto-decided with `[AUTO-DECIDED]` markers.
- "Choose for me and automate the rest" transitions to AUTONOMOUS.
- Store first 3 user significance overrides in `significance_calibration`.
- Active in Phases 1, 1B, 1.5, 2A, 2B. Dormant in 0, 3, 4, 5.

---

## 5. State Management

### phase-state.json

Single source of truth. Update after every phase transition:
- `current_phase`, `steering_mode`, `significance_calibration`, `decisions_log`
- `refinement_loops`, `refine_passes`, `stale_phases`
- `epics_count`, `stories_total`, `stories_enriched`, `validation_status`

### Stale Phase Tracking

When a phase is re-run, mark all downstream phases stale. Re-run stale phases in order before proceeding.

**Propagation chain**: `discovery` → `requirements` → `sprint-scoping` → `ux-design` → `architecture` → `epic-design` → `story-decomposition` → `write-stories` → `validation`

(Prior versions used `story-enrichment` at this position; the field is renamed to `write-stories` but readers should accept both names for back-compat when processing existing `phase-state.json` files.)

### OMC State (optional)

Use `state_write`/`state_read` with `mode="sprint-plan"` if available. Skip if not. **Boundary**: `phase-state.json` and OMC state are independent — never cross-read.

---

## 6. Progress Reporting

After each phase: report phase name, artifacts, decisions, next phase. Present Decision Steering elicitations BEFORE reporting phase complete.

At completion:
1. **Deactivate auto-continuation**: Delete the ralph state file written in step 2c (or set `active: false`). This prevents the Stop hook from blocking after planning is done.
2. Report sprint number, mode, epic/story counts, decision counts, validation status, readiness report path, next steps (`/sprint-exec`, `/refine`).

---

## 7. Error Recovery

- **Idempotency**: Any phase can be re-run. Downstream phases marked stale.
- **Resume**: `--continue` auto-detects next phase. `--restart-from` re-runs a specific phase.
- **Agent failure**: Retry once. If still failing, present error with options (retry, skip if optional, abort).
- **Stale recovery**: After current phase, check `stale_phases`, re-run in order.

---

## 8. Refine Interface

The `/refine` skill handles RALPLAN-DR refinement. Orchestrator supports it by:
- Tracking `refine_passes` per phase/scope in `phase-state.json` (max 2 before diminishing-returns warning; `--force` overrides)
- Marking downstream phases stale after refinement modifies an artifact
- When `--propagate` is used and ADRs change: auto-run reconcile
