---
name: audit-story
description: Validate story completion against acceptance criteria and current codebase. Identifies gaps, checks against new facts/libraries, produces an actionable work plan, and updates story metadata for re-implementation.
user-invocable: true
argument-hint: "[--story=N.M] [--epic=N] [--all] [--context=\"...\"] [--tdd] [--dry-run]"
---

# audit-story: Story Completion Audit & Work Plan

Validates whether a story is actually complete by checking the implementation against acceptance criteria, architecture decisions, and optionally new facts or library changes. Produces a gap analysis and actionable work plan, then updates story metadata so the next implementor knows exactly what's left.

---

## 1. Argument Parsing

Parse `$ARGUMENTS`:

| Flag | Required | Description |
|------|----------|-------------|
| `--story=N.M` | No | Audit a specific story |
| `--epic=N` | No | Audit all stories in an epic |
| `--all` | No | Audit all stories marked `done` in the sprint |
| `--context="..."` | No | New facts, library changes, or constraints to check against (can be repeated) |
| `--context-file=PATH` | No | Path to a file containing new context (e.g., updated library docs, API changes) |
| `--tdd` | No | Generate failing tests for unmet/partial ACs before creating work plan. Tests become the validation gate for re-implementation. |
| `--dry-run` | No | Report findings without modifying any story files |

At least one of `--story`, `--epic`, or `--all` is required.

If none provided, default to `--all` and confirm with the user:
```
No scope specified. Audit all done stories in the current sprint? (y/n)
```

---

## 2. Gather Context

### 2a. Read Sprint Artifacts

Read:
- `.omc/sprint-plan/current/phase-state.json`
- `.omc/sprint-plan/current/architecture-decisions.md`
- `.omc/sprint-plan/current/epics.md`
- `.omc/sprint-plan/current/requirements.md`
- `.omc/sprint-plan/current/decision-graph.md` (if exists — used to check if dependent ADRs were revised since story was implemented)
- Any replan log: `.omc/sprint-plan/current/replan-log.md` (if exists)

### 2b. Collect Stories to Audit

Based on scope flags, collect the set of story files to audit.

For each story, read:
- Full story file (spec + Dev Agent Record)
- Story frontmatter (status, blocker info, replan notes)

### 2c. Collect New Context

If `--context` or `--context-file` is provided, parse and prepare the additional context. This might be:
- "We switched from Eden Treaty to ky for API calls"
- "The auth endpoint now returns a different response shape"
- A paste of new library documentation
- Updated API specs

---

## 3. Audit Each Story

For each story in scope, dispatch a verifier agent to perform a deep audit.

Process stories in parallel (up to 5 concurrent agents).

```python
Agent(
    subagent_type="oh-my-claudecode:verifier",
    model="sonnet",  # escalate to opus for stories with complex ACs
    prompt="""
You are auditing whether a story implementation is complete and correct.

## Story Specification
{story_file_content}

## Architecture Decisions (relevant to this story)
{relevant_decisions_text}

## Requirements Traced to This Story
{relevant_requirements}

{if replan_notes_exist}
## Replan History
{replan_notes_for_this_story}
{/if}

{if new_context}
## New Context to Check Against
{new_context_text}

IMPORTANT: Check whether the implementation is compatible with this new context.
If the story was implemented against old assumptions that conflict with this context,
flag each conflict as a gap.
{/if}

## Audit Task

Perform a thorough audit of this story's implementation:

### Step 1: Verify Each Acceptance Criterion
For each AC in the story:
1. Read the files listed in the Dev Agent Record
2. Determine if the AC is:
   - **MET**: Implementation satisfies the criterion. Cite the file and line(s).
   - **PARTIAL**: Some aspects implemented but incomplete. Describe what's missing.
   - **NOT MET**: Not implemented at all. Note what's needed.
   - **UNTESTABLE**: Cannot verify from code alone (e.g., requires manual UI testing). Note this.

### Step 2: Check Architecture Compliance
For each architecture decision referenced by the story:
- Is the implementation consistent with the decision?
- If new context was provided, does the implementation conflict with it?
- If the decision graph is available: check whether the ADR has a revision history entry dated AFTER the story's `completed_at` timestamp. If so, flag it — the story was implemented against an older version of the decision.
- If a replan log exists: check whether any replan entries affect decisions this story depends on.

### Step 3: Check Code Quality Basics
- Do the files listed in the Dev Agent Record actually exist?
- Are there obvious errors (imports of nonexistent modules, TODO/FIXME markers, commented-out code)?
- Are there any hardcoded values that should be configurable?

### Step 4: Assess Overall Completeness
Rate the story:
- **COMPLETE**: All ACs met, architecture compliant, no issues
- **MOSTLY_COMPLETE**: Minor gaps (cosmetic, non-functional, or edge cases)
- **INCOMPLETE**: Significant ACs not met
- **NEEDS_REWORK**: Implementation conflicts with new context or architecture changes
- **NOT_STARTED**: No meaningful implementation found despite `done` status

Return structured output:
```json
{
  "story": "N.M",
  "title": "...",
  "overall_verdict": "COMPLETE|MOSTLY_COMPLETE|INCOMPLETE|NEEDS_REWORK|NOT_STARTED",
  "ac_results": [
    {
      "ac": "AC description",
      "verdict": "MET|PARTIAL|NOT_MET|UNTESTABLE",
      "evidence": "file:line — what was found",
      "gap": null | "description of what's missing"
    }
  ],
  "architecture_issues": [
    {
      "decision": "D-NNN",
      "issue": "description",
      "severity": "high|medium|low"
    }
  ],
  "new_context_conflicts": [
    {
      "context": "what changed",
      "conflict": "how implementation conflicts",
      "files": ["affected files"],
      "fix_approach": "suggested fix"
    }
  ],
  "code_issues": [
    {
      "file": "path",
      "issue": "description",
      "severity": "high|medium|low"
    }
  ],
  "work_items": [
    {
      "id": "W-001",
      "description": "what needs to be done",
      "ac_refs": ["AC-1", "AC-3"],
      "files": ["files to modify"],
      "effort": "small|medium|large",
      "priority": "must|should|could"
    }
  ]
}
```
""",
)
```

---

## 4. Compile Audit Results

### 4a. Aggregate Verdicts

Collect results from all verifier agents and build a summary:

```
═══════════════════════════════════════════════════
  STORY AUDIT RESULTS
═══════════════════════════════════════════════════

  Stories audited: {count}

  COMPLETE:         {count} — no action needed
  MOSTLY COMPLETE:  {count} — minor gaps
  INCOMPLETE:       {count} — significant work remaining
  NEEDS REWORK:     {count} — conflicts with new context
  NOT STARTED:      {count} — false completions

  ─────────────────────────────────────────────────

  {for each non-COMPLETE story}
  Story {N.M}: {title} — {verdict}
    ACs: {met}/{total} met, {partial} partial, {not_met} not met
    {if new_context_conflicts}
    ⚠ Conflicts with new context: {count}
    {/if}
    Work items: {count} ({must_count} must, {should_count} should)
  {/for}
═══════════════════════════════════════════════════
```

### 4b. Detect Architectural Issues

If multiple stories have the same architecture issue or new-context conflict, flag it as a systemic problem:

```
⚠ SYSTEMIC ISSUE DETECTED:
  {count} stories conflict with: {description}
  Consider running /replan --decision=D-{NNN} to propagate the fix.
```

If `--dry-run`, stop here.

---

## 5. TDD Test Generation (if `--tdd`)

Skip this section if `--tdd` was not specified. If `--tdd` is active, generate tests for each non-COMPLETE story before updating story files.

### 5a. Detect Test Framework

Read the project's existing test setup to determine:
- Test framework (vitest, jest, bun:test, pytest, go test, etc.)
- Test file naming convention (`.test.ts`, `.spec.ts`, `_test.go`, `test_*.py`, etc.)
- Test directory convention (co-located, `__tests__/`, `tests/`, etc.)
- Any test config files (vitest.config.ts, jest.config.js, pytest.ini, etc.)

Use Glob to find existing test files and infer conventions. If no tests exist yet, ask the user which framework to use.

### 5b. Generate AC Tests

For each non-COMPLETE story, dispatch a test-engineer agent to write tests for the unmet and partial ACs:

```python
Agent(
    subagent_type="oh-my-claudecode:test-engineer",
    model="sonnet",
    prompt="""
You are writing tests that validate acceptance criteria for a story.
These tests should FAIL against the current implementation — they define
what "done" looks like for the next implementor.

## Story
{story_title} (Story {N.M})

## Test Framework
{detected_framework}
Test naming: {convention}
Test location: {directory_convention}

## Acceptance Criteria to Test

{for each AC with verdict PARTIAL or NOT_MET}
### {ac_description}
- Audit verdict: {verdict}
- Gap: {gap_description}
- Relevant files: {files from audit}
- Evidence: {evidence from audit}
{end for}

{if new_context_conflicts}
## New Context Conflicts to Test
{for each conflict}
### {conflict_description}
- Expected behavior: {what the code should do under new context}
- Current behavior: {what it does now}
- Files: {affected files}
{end for}
{/if}

## Instructions

1. Write test files that validate each unmet/partial AC.
2. Each test should:
   - Have a descriptive name that maps to the AC (e.g., `test_cart_drawer_shows_item_count`)
   - Test the EXPECTED behavior (what the AC requires), not the current behavior
   - Be runnable independently
   - Include a comment referencing the AC: `// AC: {ac_description}`
3. Group tests by file/module being tested.
4. Use the project's existing test patterns and utilities where possible.
5. Do NOT fix the implementation — only write tests. The tests should FAIL.
6. If testing requires mocks/fixtures, create minimal ones.
7. Write integration-level tests where possible (test the behavior, not internals).

## Output
- Write test files to the appropriate location following project conventions
- Return a list of test files created and which ACs each covers
""",
)
```

### 5c. Verify Tests Fail

After writing tests, run them to confirm they fail as expected:

```bash
{test_runner_command} {test_files}  # e.g., vitest run tests/story-3.2-audit.test.ts
```

Categorize results:
- **Expected failure**: Test fails because the AC isn't met — this is correct
- **Unexpected pass**: Test passes — either the AC is actually met (audit was wrong) or the test is too weak
- **Error**: Test can't run (import errors, missing dependencies) — test needs fixing

For unexpected passes: re-examine the AC verdict. If the test is valid and passes, upgrade the AC to MET.

For errors: attempt one auto-fix pass. If still broken, note it and continue.

### 5d. Record Test Artifacts

For each story, track the generated tests:
- Test file paths
- Which ACs each test covers
- Pass/fail status of each test

This information is included in the story's Audit Report (section 6a) and Work Plan.

---

## 6. Update Story Files

For each non-COMPLETE story, update the story file:

### 6a. Add Audit Report Section

Insert an `## Audit Report` section before the Dev Agent Record:

```markdown
## Audit Report

**Date**: {date}
**Verdict**: {verdict}
**Audited against**: {standard spec | standard spec + new context}
**TDD**: {yes — N tests generated | no}

### Acceptance Criteria Status
| AC | Status | Gap | Test |
|----|--------|-----|------|
| {ac_description} | MET | — | — |
| {ac_description} | PARTIAL | {what's missing} | {test_file:test_name — FAILING} |
| {ac_description} | NOT MET | {what's needed} | {test_file:test_name — FAILING} |

{if architecture_issues}
### Architecture Issues
{list each issue}
{/if}

{if new_context_conflicts}
### New Context Conflicts
{list each conflict with fix approach}
{/if}

{if tdd_tests_generated}
### TDD Tests
| Test File | ACs Covered | Status |
|-----------|-------------|--------|
| {test_file_path} | AC-1, AC-3 | FAILING (expected) |

Run: `{test_runner_command} {test_files}`
All tests must pass before this story can be marked done.
{/if}

### Work Plan
| # | Task | Priority | Effort | Files | Test |
|---|------|----------|--------|-------|------|
| W-001 | {description} | must | medium | {files} | {test_name or —} |
| W-002 | {description} | should | small | {files} | {test_name or —} |

### Implementation Notes for Next Agent
{synthesized guidance — what to do, what to keep, what to change}
{if tdd}
**TDD mode**: Run the test suite after each change. The work is done when all
audit tests pass. Do not delete or weaken tests to make them pass.
Test command: `{test_runner_command} {test_files}`
{/if}
```

### 6b. Update Story Frontmatter

For stories that need rework:

1. Set `status: ready-for-dev`
2. Add `audit_verdict: {verdict}`
3. Add `audit_date: {ISO 8601}`
4. Add `remaining_work_items: {count}`
5. If new context conflicts exist: add `needs_rework_reason: "{brief reason}"`
6. If TDD tests were generated: add `tdd_tests: {comma-separated test file paths}`
7. Remove stale fields from previous execution if present: `blocker_type`, `blocker_detail`, `validation_failure`
8. Preserve existing Dev Agent Record and all other sections

### 6c. Update Execution Log

Append to `execution_log` in `phase-state.json`:

```json
{
  "type": "audit",
  "story": "N.M",
  "date": "...",
  "verdict": "INCOMPLETE",
  "previous_status": "done",
  "new_status": "ready-for-dev",
  "work_items": 3,
  "must_items": 2,
  "tdd_tests": 4
}
```

### 6d. Update Epic Status

For each epic containing audited stories:
- Recalculate epic status from updated story statuses
- Update `epic_status` in `phase-state.json`
- Update `Status:` in `epics.md`

---

## 7. Write Audit Summary

Write to `.omc/sprint-plan/current/audit-report.md` (append if exists):

```markdown
## Audit: {date}

**Scope**: {story N.M | epic N | all}
**New context**: {yes — summary | no}
**TDD**: {yes — N total tests generated | no}

### Results
| Story | Verdict | ACs Met | Work Items | TDD Tests |
|-------|---------|---------|------------|-----------|
| {N.M} | {verdict} | {met}/{total} | {count} | {test_count or —} |

### Systemic Issues
{if any cross-story patterns detected}

### Re-execution Plan
```
{list of /sprint-exec commands to re-run affected stories}
```
```

---

## 8. Report to User

```
═══════════════════════════════════════════════════
  AUDIT COMPLETE
═══════════════════════════════════════════════════

  Audited:    {count} stories
  Complete:   {count} — no changes needed
  Need work:  {count} — reset to ready-for-dev

  Work items generated: {total}
    Must:   {count}
    Should: {count}
    Could:  {count}

  {if tdd}
  TDD tests generated: {total_tests}
    Failing (expected): {count}
    Passing (AC met):   {count}
    Errors (need fix):  {count}
  {/if}

  Story files updated with:
    ✓ Audit Report (AC status, gaps, issues)
    ✓ Work Plan (prioritized task list)
    ✓ Implementation Notes (guidance for next agent)
    {if tdd}✓ TDD Tests (failing tests as validation gate){/if}
    ✓ Status reset to ready-for-dev

  Audit report: current/audit-report.md

  Re-run affected stories:
    {list /sprint-exec --story=N.M commands}

  {if tdd}
  TDD workflow:
    Tests define "done" — all must pass before story is complete.
    Run: {test_runner_command} {test_glob}
  {/if}

  {if systemic_issues}
  ⚠ Systemic issues detected — consider /replan:
    {list each with suggested /replan command}
  {/if}
═══════════════════════════════════════════════════
```

---

## 9. Integration Points

- **After `/sprint-exec`**: Run `/audit-story --all` to validate the sprint before calling it done
- **With TDD**: Run `/audit-story --all --tdd` to generate failing tests as validation gates for incomplete stories
- **With new context**: When a library or API changes, run `/audit-story --all --context="library X changed to Y"` to find what broke
- **TDD + new context**: `/audit-story --story=3.2 --tdd --context="switched to ky"` — generates tests reflecting the new expectations
- **Feeds into `/replan`**: If the audit finds systemic architecture issues, it suggests `/replan` commands
- **Feeds into `/sprint-exec`**: Stories reset to `ready-for-dev` can be re-executed with `/sprint-exec --story=N.M`. When `tdd_tests` is set in frontmatter, the executor prompt should include the test command so the agent can validate its work
- **After `/replan`**: Run `/audit-story` on replanned stories to verify the updated specs make sense before re-executing
