# Sprint Resolution Protocol

All morphist-tools skills that operate on sprint artifacts must resolve the target sprint directory before accessing any files. This enables concurrent sprints by decoupling skills from the `current` symlink singleton.

## Resolution Order

Resolve `SPRINT_DIR` using the first match:

1. **Explicit argument** — If `--sprint=<id>` was provided (e.g., `--sprint=sprint-002`), use `.omc/sprint-plan/<id>/`
2. **Session state** — If `state_read` is available, read key `morphist.active_sprint`. If set, use `.omc/sprint-plan/<value>/`
3. **Current symlink** — If `.omc/sprint-plan/current` symlink exists, follow it
4. **No sprint found** — Halt: `No active sprint found. Run /sprint-plan first, or pass --sprint=<id>.`

After resolving, verify `SPRINT_DIR/phase-state.json` exists. If not, halt with the same message.

## Usage in Skills

Every skill that reads or writes sprint artifacts must:

1. Include `[--sprint=ID]` in its `argument-hint` frontmatter (user-invocable skills only)
2. Resolve `SPRINT_DIR` using the protocol above as the first step of initialization
3. Use `SPRINT_DIR/` as the prefix for all artifact paths (e.g., `SPRINT_DIR/phase-state.json`, `SPRINT_DIR/stories/`, `SPRINT_DIR/architecture-decisions.md`)

### Inline Preamble (copy into skill Section 1)

```markdown
### Sprint Resolution

Resolve the target sprint directory (`SPRINT_DIR`):
1. If `--sprint=<id>` was provided in `$ARGUMENTS`, set `SPRINT_DIR` = `.omc/sprint-plan/<id>/`
2. Else if `state_read` is available, read key `morphist.active_sprint`. If set, `SPRINT_DIR` = `.omc/sprint-plan/<value>/`
3. Else if `.omc/sprint-plan/current` symlink exists, `SPRINT_DIR` = `.omc/sprint-plan/current/`
4. Otherwise halt: "No active sprint found. Run `/sprint-plan` first, or pass `--sprint=<id>`."

Verify `SPRINT_DIR/phase-state.json` exists. If not, halt with the same message.
```

## Registering Active Sprint

When `sprint-plan` creates or resumes a sprint, it registers the sprint ID in OMC session state:

- **Key**: `morphist.active_sprint`
- **Value**: sprint directory name (e.g., `sprint-003`)

This allows other skills invoked in the same session to automatically target the correct sprint without requiring the `--sprint` flag or relying on the `current` symlink.

## Concurrency Model

Multiple sprints can be active simultaneously when:
- Different conversations each bind to different sprints via session state
- Different worktrees each have their own `.omc/sprint-plan/` directory
- Users explicitly pass `--sprint=<id>` to target a specific sprint

The `current` symlink remains as a convenience default (last-touched sprint) but is no longer the sole mechanism for sprint targeting.
