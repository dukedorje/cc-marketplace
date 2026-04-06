---
name: sprint-validate
description: Full adversarial validation of sprint planning artifacts — FR coverage, architecture compliance, story quality, and mechanical correctness. Produces a readiness report.
user-invocable: true
argument-hint: "[--fix] [--force] [--sprint=ID]"
---

# sprint-validate: Full Sprint Validation Gate

Runs adversarial validation of the complete sprint artifact chain. Dispatches a `critic` (opus) for coverage/quality validation and a `verifier` (sonnet) for mechanical correctness in parallel. Optionally auto-fixes issues.

This is the full validation gate — for a quick smoke test of executed stories, use `/verify` instead.

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

This skill can run at any point after story enrichment (Phase 4). It does not require `current_phase` to be `"validation"` — use `--force` to run even if stories are incomplete.

## 2. Parse Arguments

| Flag | Default | Behavior |
|------|---------|----------|
| `--fix` | off | Auto-fix issues found (max 2 iterations) |
| `--force` | off | Run even if story enrichment is incomplete |

## 3. Execute Validation

Read the phase instruction file at `${CLAUDE_SKILL_DIR}/../sprint-plan/phases/phase-5-validation.md` for the complete validation process including:

- Critic agent prompt (opus) — FR coverage, architecture compliance, dependency validation, story quality, epic health
- Verifier agent prompt (sonnet) — YAML validity, reference integrity, consistency checks
- Finding merge and severity classification
- Auto-fix loop (if `--fix`, max 2 iterations)
- Readiness report generation

Both agents are dispatched in parallel. See the phase file for full agent prompts and finding formats.

## 4. Output

Write `STATE_DIR/readiness-report.md` with:
- Overall status: READY / READY WITH WARNINGS / NOT READY
- FR coverage matrix
- Architecture compliance summary
- Epic health report
- Validation issues (if any remain)
- Decision summary
- Sprint statistics

Update `STATE_DIR/phase-state.json`: set `validation_status` to `"pass"` or `"fail"`.

Present the readiness report to the user.

## 5. When Called from sprint-plan

When invoked as Phase 5 within `/sprint-plan`, sprint-plan updates `current_phase` to `"validation"` after this skill completes. The readiness report is the final checkpoint before execution.

## 6. Standalone Usage

```
/sprint-validate              # Validate current sprint artifacts
/sprint-validate --fix        # Validate and auto-fix issues
/sprint-validate --force      # Validate even if enrichment incomplete
```

Useful after manual edits to story files, architecture decisions, or requirements — re-validates the full chain without re-running sprint-plan.
