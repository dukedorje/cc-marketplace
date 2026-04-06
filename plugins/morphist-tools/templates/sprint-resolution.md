# Sprint Resolution Protocol

All morphist-tools skills that operate on sprint artifacts must resolve the target sprint directories before accessing any files. Sprint artifacts are split between two locations:

- **`SPEC_DIR`** — Committed spec artifacts (`docs/sprints/{NNN}-{slug}/`): requirements, architecture decisions, epics, stories, discovery, retrospective. These represent intent and are version-controlled.
- **`STATE_DIR`** — Ephemeral operational state (`.omc/sprint-plan/sprint-{NNN}/`): phase-state.json, execution logs, reviews, work-log. Gitignored.

The bridge between them is `STATE_DIR/phase-state.json`, which contains a `spec_dir` field pointing to the corresponding `SPEC_DIR`.

## Resolution Order

Resolve the sprint using the first match:

1. **Explicit argument** — If `--sprint=<id>` was provided (e.g., `--sprint=sprint-002`), use `.omc/sprint-plan/<id>/` as `STATE_DIR`
2. **Session state** — If `state_read` is available, read key `morphist.active_sprint`. If set, use `.omc/sprint-plan/<value>/` as `STATE_DIR`
3. **Current symlink** — If `.omc/sprint-plan/current` symlink exists, follow it as `STATE_DIR`
4. **No sprint found** — Halt: `No active sprint found. Run /sprint-plan first, or pass --sprint=<id>.`

After resolving `STATE_DIR`, verify `STATE_DIR/phase-state.json` exists. If not, halt with the same message.

Then read `phase-state.json` and set `SPEC_DIR` from its `spec_dir` field. Verify `SPEC_DIR` exists.

## Usage in Skills

Every skill that reads or writes sprint artifacts must:

1. Include `[--sprint=ID]` in its `argument-hint` frontmatter (user-invocable skills only)
2. Resolve `STATE_DIR` and `SPEC_DIR` using the protocol above as the first step of initialization
3. Use `SPEC_DIR/` as the prefix for spec artifact paths (requirements.md, architecture-decisions.md, epics.md, stories/, discovery.md, retrospective.md, etc.)
4. Use `STATE_DIR/` as the prefix for operational state paths (phase-state.json, work-log.md, reviews/, etc.)

### Inline Preamble (copy into skill Section 1)

```markdown
### Sprint Resolution

Resolve the target sprint directories:
1. If `--sprint=<id>` was provided in `$ARGUMENTS`, set `STATE_DIR` = `.omc/sprint-plan/<id>/`
2. Else if `state_read` is available, read key `morphist.active_sprint`. If set, `STATE_DIR` = `.omc/sprint-plan/<value>/`
3. Else if `.omc/sprint-plan/current` symlink exists, `STATE_DIR` = `.omc/sprint-plan/current/`
4. Otherwise halt: "No active sprint found. Run `/sprint-plan` first, or pass `--sprint=<id>`."

Verify `STATE_DIR/phase-state.json` exists. Read it and set `SPEC_DIR` from the `spec_dir` field.
Verify `SPEC_DIR` exists. If not, halt: "Sprint spec directory not found at {spec_dir}. Sprint may need re-initialization."
```

## Registering Active Sprint

When `sprint-plan` creates or resumes a sprint, it registers the sprint ID in OMC session state:

- **Key**: `morphist.active_sprint`
- **Value**: sprint directory name (e.g., `sprint-003`)

This allows other skills invoked in the same session to automatically target the correct sprint without requiring the `--sprint` flag or relying on the `current` symlink.

## Concurrency Model

Multiple sprints can be active simultaneously when:
- Different conversations each bind to different sprints via session state
- Different worktrees each have their own `.omc/sprint-plan/` and `docs/sprints/` directories
- Users explicitly pass `--sprint=<id>` to target a specific sprint

The `current` symlink remains as a convenience default (last-touched sprint) but is no longer the sole mechanism for sprint targeting.
