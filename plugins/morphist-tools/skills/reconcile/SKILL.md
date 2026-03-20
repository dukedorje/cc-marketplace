---
name: reconcile
description: Cross-story and cross-epic code style reconciliation. Detects inconsistencies in naming, patterns, and conventions across parallel agent work. Raises elicitations for contentious choices.
user-invocable: true
argument-hint: "[--epic=N] [--all] [--auto]"
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
| `--auto` | off | Auto-resolve all inconsistencies without elicitation (use majority-wins rule) |

If no flag is provided, reconcile the most recently completed epic (same logic as `/sprint-review`).

---

## 2. Load Context

### 2a. Sprint Artifacts

Read from `.omc/sprint-plan/current/`:
- `architecture-decisions.md` — decisions that may prescribe conventions
- `discovery.md` — existing codebase patterns (if any) that set the baseline

### 2b. Determine Scope

Based on flags, collect the relevant story files from `current/stories/`. Only include stories with `status: done`.

For each story, read the Dev Agent Record to get the list of files created/modified.

### 2c. Build File Inventory

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

Architecture decisions: .omc/sprint-plan/current/architecture-decisions.md
Existing codebase patterns: .omc/sprint-plan/current/discovery.md

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

Write the reconciliation report to `.omc/sprint-plan/current/reviews/reconciliation-{scope}.md` where `{scope}` is `epic-{N}` or `full-sprint`.

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

  Report: .omc/sprint-plan/current/reviews/reconciliation-{scope}.md
═══════════════════════════════════════════════════
```

---

## 7. Integration Points

### From `/sprint-review` (per-epic)

After the code review agent completes for an epic (section 3), dispatch reconciliation for that epic:

Equivalent to `/reconcile --epic={N} --auto` in the background (non-blocking). Auto mode is used because the full-sprint reconciliation in retro will handle contentious cross-epic issues with elicitation.

### From `/retro` (full-sprint)

After data collection (section 3) and before generating the retrospective (section 4), dispatch full-sprint reconciliation:

Equivalent to `/reconcile --all`. This runs in GUIDED mode (elicitations for contentious choices). The reconciliation report is included as input to the retrospective writer agent so findings appear in the Patterns Discovered section.
