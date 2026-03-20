---
name: sprint-review
description: Review completed epic implementations against story specs, architecture decisions, and acceptance criteria. Callable standalone or dispatched automatically by sprint-exec after each epic.
user-invocable: true
argument-hint: "[--epic=N] [--all]"
---

# Sprint Review

Review completed epic work against sprint specifications. This skill can be:
- Called manually: `/sprint-review --epic=2`
- Called with no args to review the most recently completed epic
- Called with `--all` to review all completed epics
- Dispatched by `sprint-exec` in the background after each epic completes

---

## 1. Argument Parsing

Parse `$ARGUMENTS`:

| Flag | Default | Behavior |
|------|---------|----------|
| `--epic=N` | most recent completed | Review a specific epic |
| `--all` | off | Review all completed epics |

If no flag is provided, determine the most recently completed epic by reading story files in `current/stories/` and finding the highest epic number where at least one story has `status: done`.

---

## 2. Initialization

### 2a. Load Sprint Context

Read from `.omc/sprint-plan/current/`:
- `architecture-decisions.md` — decisions that implementations must comply with
- `requirements.md` — FR/NFR source of truth
- `epics.md` — epic structure and story assignments

### 2b. Determine Epics to Review

Based on parsed arguments, build the list of epics to review.

For each epic in scope, collect:
- All story files in `current/stories/{epic}-*.md`
- Filter to stories with `status: done` (skip `ready-for-dev`, `in-progress`, `blocked`, `failed`)
- If an epic has zero completed stories, skip it and note: "Epic {N}: no completed stories to review."

### 2c. Create Reviews Directory

```bash
mkdir -p .omc/sprint-plan/current/reviews
```

---

## 3. Per-Epic Review

For each epic in scope, dispatch a `code-reviewer` agent. If reviewing multiple epics, dispatch all in parallel.

```python
Agent(
    subagent_type="oh-my-claudecode:code-reviewer",
    model="opus",
    prompt="""
You are reviewing the implementation of Epic {N}: {epic_title} from a sprint plan.

## Sprint Context

Architecture decisions: .omc/sprint-plan/current/architecture-decisions.md
Requirements: .omc/sprint-plan/current/requirements.md

## Stories to Review

{for each completed story in this epic}
Story file: .omc/sprint-plan/current/stories/{story_file}
{end for}

## Review Process

For each completed story:

1. **Read the story spec** — acceptance criteria, scope boundaries, architecture compliance, technical requirements.
2. **Read the Dev Agent Record** at the bottom of the story file — files created/modified, completion notes.
3. **Read the actual implementation files** listed in the Dev Agent Record.
4. **Evaluate against each acceptance criterion**:
   - For each AC: does the implementation satisfy the Given/When/Then?
   - Mark: PASS, PARTIAL (missing edge case), or FAIL (not implemented)
5. **Check architecture compliance**:
   - For each decision referenced in the story frontmatter, verify the implementation follows it.
   - Flag any divergence.
6. **Check scope boundaries**:
   - Did the implementation stay within the "DOES" list?
   - Did it accidentally include anything from the "does NOT" list?
7. **Code quality**:
   - Are tests present and meaningful (not just stubs)?
   - Are there obvious bugs, missing error handling, or security issues?
   - Does the code follow patterns established by earlier stories?

## Output

Write your review to: .omc/sprint-plan/current/reviews/epic-{N}-review.md

Use this format:

```markdown
# Epic {N} Review: {epic_title}
**Reviewed**: {date}
**Stories reviewed**: {count}

## Summary
**Overall**: approved | approved with concerns | needs attention
**AC pass rate**: {pass_count}/{total_ac_count}
**Architecture compliance**: compliant | divergences noted

## Per-Story Reviews

### Story {N}.{M}: {title}
**Status**: pass | concerns | fail

#### Acceptance Criteria
| AC | Result | Notes |
|----|--------|-------|
| AC1: {title} | PASS/PARTIAL/FAIL | {detail} |

#### Architecture Compliance
- D-{NNN}: {compliant / divergence: detail}

#### Scope Check
- Within scope: yes / no — {detail if no}

#### Code Quality Notes
- {findings}

### Story {N}.{M}: {title}
[repeat for each story]

## Cross-Story Concerns
[Issues spanning multiple stories: inconsistencies, missing integration, duplicated code, etc.]

## Recommendations
[Specific actionable items if any stories need rework]
```
""",
)
```

---

## 4. Per-Epic Reconciliation

After each epic's code review completes, dispatch a background reconciliation pass for that epic. This checks for code style inconsistencies across stories within the epic.

Equivalent to `/reconcile --epic={N} --auto` with `run_in_background=True`. Auto mode is used here because the full-sprint reconciliation during `/retro` handles contentious cross-epic issues with user elicitation.

Skip if the epic had fewer than 2 completed stories (nothing to reconcile).

Reconciliation report lands in `.omc/sprint-plan/current/reviews/reconciliation-epic-{N}.md`.

---

## 5. Report Results

After all reviews complete, present a summary to the user:

```
═══════════════════════════════════════════════════
  SPRINT REVIEW COMPLETE
═══════════════════════════════════════════════════

  {for each epic reviewed}
  Epic {N}: {epic_title}
    Status: {approved / approved with concerns / needs attention}
    AC pass rate: {pass}/{total}
    Review: .omc/sprint-plan/current/reviews/epic-{N}-review.md
  {end for}

  {if any epic needs attention}
  Epics needing attention:
    Epic {N}: {count} ACs failed — see review for details
  {/if}
═══════════════════════════════════════════════════
```

---

## 6. Integration with sprint-exec

When `sprint-exec` dispatches this skill after an epic completes (section 4f), it calls the equivalent of `/sprint-review --epic=N` with `run_in_background=True`. The sprint-exec orchestrator should use the same agent dispatch shown in section 3, not a separate implementation.
