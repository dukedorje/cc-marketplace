---
name: sprint-plan
description: Multi-phase sprint planning workflow that transforms product ideas into implementation-ready user stories with architecture decisions, requirements expansion, and adversarial validation
user-invocable: true
argument-hint: "[product-idea-or-prd-path] [--fast] [--thorough] [--auto] [--step] [--sprint-size=SIZE] [--continue[=phase]] [--restart-from=phase] [--sprint=ID]"
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
| `--fast` | Single-pass, no RALPLAN-DR, AUTONOMOUS steering, implies `--auto` |
| `--thorough` | Full consensus, GUIDED steering, refinement loops (default) |
| `--auto` | No inter-phase pauses (only Decision Steering can pause) |
| `--step` | Pause after every phase |
| `--sprint-size=SIZE` | `focused` (3-8 stories), `standard` (8-18, default), `ambitious` (15-30) |
| `--skip-ux` | Skip Phase 1.5 even if frontend detected |
| `--continue[=phase]` | Resume from next incomplete phase (or after specified phase) |
| `--restart-from=phase` | Re-run a phase and mark all downstream stale |
| `--sprint=ID` | Target a specific sprint (e.g., `--sprint=sprint-002`). For `--continue`/`--restart-from`, operates on this sprint instead of the most recent. For new sprints, ignored. |

**Precedence**: `--thorough` > `--fast`. `--restart-from` > `--continue`. `--step` > `--auto`. `--fast` implies `--auto`.

---

## 2. Initialization

### 2a. New Sprint (skip if `--continue` or `--restart-from`)

1. Determine sprint number: find highest `sprint-NNN/` in `.omc/sprint-plan/`, increment (or start at `sprint-001`).
2. Create directories: `.omc/sprint-plan/sprint-{NNN}/stories` and `.omc/sprint-plan/decisions`.
3. Update symlink: `rm -f .omc/sprint-plan/current && ln -s sprint-{NNN} .omc/sprint-plan/current`.
4. Write `.omc/sprint-plan/AGENTS.md` — static pointer: "Sprint artifacts in `.omc/sprint-plan/current/`".
5. Set `SPRINT_DIR` = `.omc/sprint-plan/sprint-{NNN}/`.

### 2b. Initialize phase-state.json

```json
{
  "sprint": "sprint-{NNN}",
  "mode": "thorough|fast",
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
2. Read `phase-state.json`. Set `SPRINT_DIR` = `.omc/sprint-plan/<resolved>/`.
3. Without phase arg: resume from next phase after `current_phase`.
4. With phase arg (`--continue=<phase>`): verify artifact exists, resume from next phase.
5. Update symlink and `current_phase`. Register in OMC session state (step 2c). Report: "Resuming sprint {NNN} from {phase}."

### 2e. Handle --restart-from

1. If `--sprint=<id>` provided, use `.omc/sprint-plan/<id>/`. Otherwise find most recent `sprint-NNN/`.
2. Read `phase-state.json`. Set `SPRINT_DIR` = `.omc/sprint-plan/<resolved>/`.
3. Mark specified phase + all downstream as stale.
4. Update symlink and `current_phase`. Register in OMC session state (step 2c). Report: "Restarting from {phase}. Downstream marked stale."

---

## 3. Phase Orchestration

Execute phases sequentially. For each phase:
1. Read phase instruction file from `${CLAUDE_SKILL_DIR}/phases/`.
2. Load previous phase's output artifact (context shedding — read from disk, not memory).
3. Dispatch agent(s) per the phase file. Include in the agent prompt: **"For this execution, `SPRINT_DIR` = `{resolved path}`."** Phase files use `SPRINT_DIR` as the artifact path prefix.
4. Write output to the correct path under `SPRINT_DIR/`.
5. Update `current_phase` and metrics in `SPRINT_DIR/phase-state.json`.
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
| 4: Enrichment | `phase-4-story-enrichment.md` | step only | Dormant | `stories/*.md` |
| 5: Validation | `phase-5-validation.md` | Informational | Dormant | `readiness-report.md` |

### Phase-Specific Notes

- **Phase 1.5 trigger**: Run if `has_frontend: true` AND `has_ux_artifacts: false` AND not `--skip-ux` AND not `--fast`. Otherwise skip (`ux_design_phase: "skipped"`).
- **Phase 2A refinement loop** (thorough only): If decisions create new requirements, loop back to Phase 1 (incremental), then re-run 2A. Max 3 iterations via `refinement_loops.requirements_architecture.count`.
- **Phase 3**: Epics sequential (forward intelligence). Stories parallel within each epic.
- **Phase 4**: Stories parallel within epic, sequential across epics. Agents read artifact files directly (do not inject full content).
- **Phase 5**: Auto-fix loop (max 2 iterations). Readiness report always shown.

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

**Propagation chain**: `discovery` → `requirements` → `sprint-scoping` → `ux-design` → `architecture` → `epic-design` → `story-decomposition` → `story-enrichment` → `validation`

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
