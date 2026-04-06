---
name: scope
description: Negotiate sprint scope boundary — cluster FRs, propose IN/STRETCH/DEFER split, interactive negotiation with user. Usable standalone or as Phase 1B of sprint-plan.
user-invocable: true
argument-hint: "[--sprint-size=SIZE] [--auto] [--show] [--sprint=ID]"
---

# scope: Sprint Scope Negotiation

Analyzes the full requirement surface and negotiates the sprint boundary with the user. Determines which FRs are in scope (IN), stretch goals (STRETCH), and deferred (DEFER) — before architecture decisions are made.

---

## 1. Prerequisites

### Sprint Resolution

Resolve the target sprint directories:
1. If `--sprint=<id>` was provided in `$ARGUMENTS`, set `STATE_DIR` = `.omc/sprint-plan/<id>/`
2. Else if `state_read` is available, read key `morphist.active_sprint`. If set, `STATE_DIR` = `.omc/sprint-plan/<value>/`
3. Else if `.omc/sprint-plan/current` symlink exists, `STATE_DIR` = `.omc/sprint-plan/current/`
4. Otherwise halt: "No active sprint found. Run `/sprint-plan` first, or pass `--sprint=<id>`."

Verify `STATE_DIR/phase-state.json` exists. Read it and set `SPEC_DIR` from the `spec_dir` field.
Verify `SPEC_DIR` exists. If not, halt: "Sprint spec directory not found at {spec_dir}. Sprint may need re-initialization."

Read `SPEC_DIR/requirements.md`.

If not found: `No requirements found. Run /sprint-plan first to generate requirements, or /prd to start from scratch.`

## 2. Parse Arguments

| Flag | Default | Behavior |
|------|---------|----------|
| `--sprint-size=SIZE` | `standard` | Sizing hint: `focused` (3-8 stories), `standard` (8-18), `ambitious` (15-30) |
| `--auto` | off | Auto-accept the analyst's proposed boundary without negotiation |
| `--show` | off | Show current scope without re-negotiation |

## 3. Show Mode

If `--show`: read and display `SPEC_DIR/sprint-scope.md` if it exists. Show the IN/STRETCH/DEFER split. Stop.

## 4. Execute Scoping

Read the phase instruction file at `${CLAUDE_SKILL_DIR}/../sprint-plan/phases/phase-1b-sprint-scoping.md` for the complete scoping process including:

- FR clustering by cohesion and dependency
- Story count estimation per cluster
- Sprint boundary proposal (IN / STRETCH / DEFER)
- Velocity calibration for sprint N>1
- Interactive negotiation format
- Backlog integration for deferred items

**Agent**: `analyst` (opus)

**Inputs**:
- `SPEC_DIR/requirements.md` — all FRs from Phase 1
- `SPEC_DIR/discovery.md` — velocity data for sprint N>1
- `.omc/backlog.md` and `.omc/backlog-promoted.md` if they exist

## 5. Negotiation

If `--auto`: auto-accept the analyst's proposed boundary. Log with `[AUTO-DECIDED]` marker.

Otherwise: present the scoping proposal as an interactive negotiation. The user can:
- Accept the proposal as-is
- Move FR clusters between IN/STRETCH/DEFER
- Change sprint size
- Adjust individual FRs

This interaction IS the Decision Steering for sprint scope (CRITICAL significance).

## 6. Output

Write `SPEC_DIR/sprint-scope.md` with the negotiated boundary.

For deferred FR clusters: add items to `.omc/backlog.md` with `source: scoping:sprint-{N}` (graceful if backlog doesn't exist).

Update `STATE_DIR/phase-state.json`: set `sprint_size` and `sprint_scope`.

## 7. When Called from sprint-plan

When invoked as Phase 1B within `/sprint-plan`, sprint-plan updates `current_phase` to `"sprint-scoping"` after this skill completes. The scope constrains Phase 2A (Architecture) and Phase 2B (Epic Design) to in-scope FRs only.

## 8. Standalone Usage

```
/scope                         # Re-negotiate sprint scope
/scope --show                  # View current scope split
/scope --sprint-size=focused   # Re-scope as a focused sprint
/scope --auto                  # Auto-accept analyst's proposal
```

Useful for mid-sprint re-scoping when requirements change, or to review the current scope boundary before execution.
