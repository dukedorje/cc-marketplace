---
name: reconcile
description: Cross-story and cross-epic code style reconciliation. Detects inconsistencies in naming, patterns, and conventions across parallel agent work. Raises elicitations for contentious choices.
user-invocable: true
argument-hint: "[--epic=N] [--all] [--auto] [--sprint=ID]"
---

# Reconcile: Code Style Reconciliation

Parallel agent execution can produce style drift — different naming conventions, error handling patterns, import styles, or structural choices across stories and epics. This skill detects those inconsistencies and either resolves them or raises elicitations for contentious choices.

Callable standalone or dispatched by:
- `/sprint-review` — per-epic reconciliation after each epic review
- `/retro` — full-sprint cross-epic reconciliation

---

## 1. Argument Parsing

Parse `$ARGUMENTS`:

| Flag | Default | Behavior |
|------|---------|----------|
| `--epic=N` | none | Reconcile within a specific epic's stories |
| `--all` | off | Full-sprint reconciliation across all epics |
| `--decisions` | off | Propagate ADR changes to dependent story specs (reads decision graph). Use after `/refine` revises ADRs. |
| `--auto` | off | Auto-resolve all inconsistencies without elicitation (use majority-wins rule) |

If no flag is provided, reconcile the most recently completed epic (same logic as `/sprint-review`).

---

## 2. Load Context

### 2a. Sprint Resolution

Resolve the target sprint directories:
1. If `--sprint=<id>` was provided in `$ARGUMENTS`, set `STATE_DIR` = `.omc/sprint-plan/<id>/`
2. Else if `state_read` is available, read key `morphist.active_sprint`. If set, `STATE_DIR` = `.omc/sprint-plan/<value>/`
3. Else if `.omc/sprint-plan/current` symlink exists, `STATE_DIR` = `.omc/sprint-plan/current/`
4. Otherwise halt: "No active sprint found. Run `/sprint-plan` first, or pass `--sprint=<id>`."

Verify `STATE_DIR/phase-state.json` exists. Read it and set `SPEC_DIR` from the `spec_dir` field.
Verify `SPEC_DIR` exists. If not, halt: "Sprint spec directory not found at {spec_dir}. Sprint may need re-initialization."

### 2b. Sprint Artifacts

Read from `SPEC_DIR/`:
- `architecture-decisions.md` — decisions that may prescribe conventions
- `discovery.md` — existing codebase patterns (if any) that set the baseline

### 2c. Determine Scope

Based on flags, collect the relevant story files from `SPEC_DIR/stories/`. Only include stories with `status: done`.

For each story, read the Dev Agent Record to get the list of files created/modified.

### 2d. Build File Inventory

Collect all implementation files (not story specs) touched by stories in scope. Group by:
- Language/file type (`.ts`, `.py`, `.rs`, etc.)
- Module/directory
- Story of origin

---

## 3. Reconciliation Analysis

Dispatch a `code-reviewer` agent to analyze the files for cross-story inconsistencies.

```python
Agent(
    subagent_type="oh-my-claudecode:code-reviewer",
    model="opus",
    prompt="""
You are performing code style reconciliation across work produced by parallel dev agents.

## Context

Architecture decisions: SPEC_DIR/architecture-decisions.md
Existing codebase patterns: SPEC_DIR/discovery.md

## Files to Analyze

{grouped file inventory with story attribution}

## Analysis Categories

For each category below, scan ALL files in scope and identify inconsistencies. An inconsistency is when two or more stories made different choices for the same kind of thing.

### 1. Naming Conventions
- Variable/function naming (camelCase vs snake_case vs PascalCase)
- File naming (kebab-case vs camelCase vs snake_case)
- Component/class naming patterns
- Constants naming (SCREAMING_SNAKE vs other)
- Database column/table naming conventions

### 2. Structural Patterns
- File organization (where tests live relative to source)
- Module structure (barrel exports vs direct imports)
- Component structure (hooks extraction, prop patterns)
- Error handling approach (try/catch vs Result types vs error boundaries)
- Logging patterns (what's logged, at what level, format)

### 3. API & Data Patterns
- Request/response shapes (envelope pattern vs flat)
- Error response format
- Validation approach (where, how, which library)
- Database query patterns (raw SQL vs query builder vs ORM methods)
- State management patterns

### 4. Import & Dependency Style
- Import ordering (stdlib, external, internal)
- Absolute vs relative imports
- Aliased paths usage
- Dependency injection vs direct imports

### 5. Testing Patterns
- Test file naming and location
- Test structure (describe/it vs test blocks)
- Mock/stub approach
- Assertion style
- Setup/teardown patterns

## Output Format

For each inconsistency found, output:

```json
{
  "category": "Naming Conventions",
  "description": "Function naming in API handlers",
  "variants": [
    {
      "pattern": "camelCase (e.g., getUserById)",
      "used_in": ["stories 1.1, 1.3, 2.1"],
      "files": ["src/handlers/user.ts", "src/handlers/auth.ts"],
      "count": 12
    },
    {
      "pattern": "snake_case (e.g., get_user_by_id)",
      "used_in": ["story 2.3"],
      "files": ["src/handlers/billing.ts"],
      "count": 4
    }
  ],
  "existing_codebase_pattern": "camelCase (from discovery.md)",
  "architecture_decision": "none",
  "severity": "low | medium | high",
  "recommendation": "Adopt camelCase — matches existing codebase and is used by majority of stories",
  "auto_resolvable": true,
  "resolution_rule": "majority-wins"
}
```

**Severity guide**:
- **high**: Different error handling, validation, or auth patterns — affects correctness or security
- **medium**: Different structural patterns — affects maintainability and developer experience
- **low**: Different naming/formatting — cosmetic but creates inconsistency

If NO inconsistencies are found in a category, omit it from the output.

Return only the JSON array of inconsistencies found. If none at all, return an empty array.
""",
)
```

---

## 4. Resolution

After the analysis agent returns, process each inconsistency:

### 4a. Auto-Resolvable (or --auto mode)

If `auto_resolvable: true` or `--auto` flag is set:
- Apply the recommended resolution
- Log the resolution

### 4b. Contentious Choices (GUIDED elicitation)

If `auto_resolvable: false` or `severity: high`, present an elicitation to the user:

```
## Reconciliation Point: {description}

**Category**: {category}
**Severity**: {severity}
**What's inconsistent**: {description}

### Variants Found

**Variant A: {pattern}**
- Used in: {stories}
- Count: {count} occurrences
- Files: {file list}

**Variant B: {pattern}**
- Used in: {stories}
- Count: {count} occurrences
- Files: {file list}

{if existing_codebase_pattern}
**Existing codebase uses**: {pattern}
{/if}

{if architecture_decision}
**Architecture decision {id}**: {relevant decision}
{/if}

### Recommendation
{recommendation}

### Your Options
1. **Adopt Variant A** — apply across all files
2. **Adopt Variant B** — apply across all files
3. **Accept as-is** — leave the inconsistency (document as a known divergence)
4. **Auto-resolve remaining** — use recommendations for all remaining inconsistencies
```

Wait for user input on each high-severity item. If user chooses option 4, switch to auto-resolve for remaining items.

### 4c. Apply Resolutions

For each resolved inconsistency, dispatch an `executor` (sonnet) to apply the fix:

```python
Agent(
    subagent_type="oh-my-claudecode:executor",
    model="sonnet",
    prompt="""
Apply the following code style reconciliation:

{inconsistency description}
Chosen resolution: {chosen variant}
Files to update: {file list}

Make ONLY the specific style change described. Do not refactor, restructure,
or make any other changes. This is a cosmetic reconciliation pass.
""",
)
```

---

## 5. Write Report

Write the reconciliation report to `STATE_DIR/reviews/reconciliation-{scope}.md` where `{scope}` is `epic-{N}` or `full-sprint`.

```markdown
# Code Reconciliation: {scope}
**Date**: {date}
**Stories in scope**: {count}
**Files analyzed**: {count}

## Summary
- Inconsistencies found: {total}
- Auto-resolved: {count}
- User-resolved: {count}
- Accepted as-is: {count}
- Remaining: {count}

## Resolutions Applied

### {category}: {description}
- **Chosen**: {pattern}
- **Resolved by**: auto / user
- **Files updated**: {list}

## Accepted Divergences
{list of inconsistencies the user chose to leave as-is, with rationale}

## No Issues Found
{list of categories with no inconsistencies — confirms they were checked}
```

---

## 6. Report to User

```
═══════════════════════════════════════════════════
  RECONCILIATION COMPLETE: {scope}
═══════════════════════════════════════════════════

  Files analyzed:      {count}
  Inconsistencies:     {total} found
    Auto-resolved:     {count}
    User-resolved:     {count}
    Accepted as-is:    {count}

  Report: STATE_DIR/reviews/reconciliation-{scope}.md
═══════════════════════════════════════════════════
```

---

## 7. Decision Propagation (`--decisions`)

When `--decisions` is specified, reconcile operates in a different mode: instead of scanning code for style inconsistencies, it propagates ADR changes to dependent story specs.

### 7a. Read Decision Graph

Read `SPEC_DIR/decision-graph.md` to identify which stories depend on which ADRs.

If the graph doesn't exist, halt:
```
No decision graph found. Run /refine to build one, or create it manually at current/decision-graph.md.
```

### 7b. Detect Changed ADRs

Read `architecture-decisions.md` and look for decisions with a `## Revision History` entry newer than the last reconciliation (or since sprint start if no prior reconciliation).

If no recent revisions found:
```
No ADR revisions detected since last reconciliation. Nothing to propagate.
```

### 7c. Find Affected Stories

For each revised ADR, look up its dependents in the decision graph. Filter to stories NOT in the epic where the change was made (those were already updated during `/refine`).

### 7d. Propagate to Story Specs

For each affected story, dispatch an executor agent to update the story spec:

```python
Agent(
    subagent_type="oh-my-claudecode:executor",
    model="sonnet",
    prompt="""
You are updating a story specification due to an architecture decision revision.

## Revised Decision
Decision D-{NNN} has been updated:

### Before
{old_decision_summary}

### After
{new_decision_summary}

### Migration Notes
{migration_notes_from_revision_history}

## Story to Update
{story_file_content}

## Task
Update this story file to reflect the revised decision:

1. Update `## Architecture Compliance` section for the revised decision
2. Update acceptance criteria that reference the old approach
3. Update `## Technical Notes` or implementation guidance
4. Add or update `## Prep Notes` with a note about the decision change:
   - Date: {date}
   - Decision revised: D-{NNN}
   - What changed in this story's spec
5. Do NOT modify the Dev Agent Record (if present)
6. Do NOT change the story number, title, or epic assignment

Return the complete updated story file.
""",
)
```

### 7e. Update Story Status

For affected stories with status `done`:
- Change status to `ready-for-dev` (needs re-implementation with the new decision)
- Add `replan_reason: "D-{NNN} revised during refine for Epic {X}"`
- Preserve existing Dev Agent Record

For affected stories with status `ready-for-dev`:
- Just update the spec, keep status

### 7f. Decision Propagation Report

```
═══════════════════════════════════════════════════
  DECISION PROPAGATION COMPLETE
═══════════════════════════════════════════════════

  ADRs propagated:     {count}
  Stories updated:     {count}
  Stories reset:       {count} (done → ready-for-dev)

  Changes:
    D-{NNN}: {title}
      → Story {N.M}: spec updated
      → Story {N.M}: spec updated, reset to ready-for-dev

  Re-run affected stories:
    /sprint-exec --story={N.M}
═══════════════════════════════════════════════════
```

---

## 8. Integration Points

### From `/refine` (decision propagation)

After `/refine` revises ADRs, it suggests `/reconcile --decisions`. This propagates the decision changes to story specs in other epics that depend on the same ADRs (via the decision graph). The refine skill handles updating stories within its own epic.

### From `/sprint-review` (per-epic)

After the code review agent completes for an epic (section 3), dispatch reconciliation for that epic:

Equivalent to `/reconcile --epic={N} --auto` in the background (non-blocking). Auto mode is used because the full-sprint reconciliation in retro will handle contentious cross-epic issues with elicitation.

### From `/retro` (full-sprint)

After data collection (section 3) and before generating the retrospective (section 4), dispatch full-sprint reconciliation:

Equivalent to `/reconcile --all`. This runs in GUIDED mode (elicitations for contentious choices). The reconciliation report is included as input to the retrospective writer agent so findings appear in the Patterns Discovered section.
