---
name: review-fix
description: Validate and fix issues from sprint reviews and reconciliation reports. Verifies each finding against actual code, discards false positives, fixes real issues, and escalates contested decisions.
user-invocable: true
argument-hint: "[review-file-path] [--auto] [--tdd] [--dry-run] [--sprint=ID]"
---

# Review Fix: Validate & Fix Review Findings

Takes a review artifact (from `/sprint-review`, `/reconcile`, or any review file) and processes each finding: validates it against the actual code, discards false positives, fixes real issues, and escalates contested or high-impact decisions via elicitation.

---

## 1. Sprint Resolution & Argument Parsing

### Sprint Resolution

Resolve the target sprint directory (`SPRINT_DIR`):
1. If `--sprint=<id>` was provided in `$ARGUMENTS`, set `SPRINT_DIR` = `.omc/sprint-plan/<id>/`
2. Else if `state_read` is available, read key `morphist.active_sprint`. If set, `SPRINT_DIR` = `.omc/sprint-plan/<value>/`
3. Else if `.omc/sprint-plan/current` symlink exists, `SPRINT_DIR` = `SPRINT_DIR/`
4. Otherwise halt: "No active sprint found. Run `/sprint-plan` first, or pass `--sprint=<id>`."

Verify `SPRINT_DIR/phase-state.json` exists. If not, halt with the same message.

### Argument Parsing

Parse `$ARGUMENTS`:

| Argument | Required | Description |
|----------|----------|-------------|
| `review-file-path` | No | Path to a review file. If omitted, looks for the most recent file in `SPRINT_DIR/reviews/` |
| `--auto` | No | Auto-fix all validated issues without elicitation (skip only false positives) |
| `--tdd` | No | Write a failing test reproducing each confirmed finding before applying the fix. Tests serve as regression guards. |
| `--dry-run` | No | Validate findings and report what would be fixed, but make no changes |

If no path is provided:
1. List files in `SPRINT_DIR/reviews/`
2. If multiple reviews exist, present them to the user:
   ```
   Multiple review files found:
   1. epic-1-review.md (2026-03-19)
   2. epic-2-review.md (2026-03-19)
   3. reconciliation-full-sprint.md (2026-03-19)

   Which review would you like to process? (number, or "all" for all of them)
   ```
3. If only one exists, use it automatically.

---

## 2. Parse Review Findings

Read the review file and extract each discrete finding. A finding is any item that:
- Has a status of `concerns`, `fail`, `PARTIAL`, or `FAIL` (from sprint-review)
- Is listed as an inconsistency or resolution (from reconcile)
- Is flagged as a code quality issue, architecture divergence, or scope violation
- Appears in a "Recommendations" or "Cross-Story Concerns" section

For each finding, extract:
- **ID**: Sequential (F-001, F-002, ...) for tracking
- **Source**: Which review file and section
- **Type**: `ac-failure` | `architecture-divergence` | `scope-violation` | `code-quality` | `style-inconsistency` | `missing-test` | `other`
- **Description**: What the reviewer flagged
- **Files involved**: File paths mentioned in the finding
- **Story**: Which story (if attributable)
- **Severity**: From the review, or infer: `high` (correctness/security), `medium` (architecture/structure), `low` (style/cosmetic)

Present a summary to the user before proceeding:

```
═══════════════════════════════════════════════════
  REVIEW FIX: {review_file_name}
═══════════════════════════════════════════════════

  Findings extracted: {total}
    High severity:    {count}
    Medium severity:  {count}
    Low severity:     {count}

  Types:
    AC failures:              {count}
    Architecture divergences: {count}
    Code quality:             {count}
    Style inconsistencies:    {count}
    Missing tests:            {count}
    Other:                    {count}

  Proceeding to validate each finding...
═══════════════════════════════════════════════════
```

---

## 3. Validate Findings

For each finding, dispatch a `verifier` agent to check whether the issue is real. Process findings in parallel (batch by file to avoid conflicts — all findings touching the same file go to the same agent).

```python
Agent(
    subagent_type="oh-my-claudecode:verifier",
    model="sonnet",
    prompt="""
You are validating review findings against actual code.

## Finding

ID: {finding_id}
Type: {type}
Description: {description}
Files: {file_list}
Story: {story_id}

## Story Spec (for context)
{story_file_content_if_available}

## Architecture Decisions (for context)
{relevant_decisions_if_architecture_divergence}

## Task

1. Read each file listed in the finding.
2. Determine if the finding is:
   - **CONFIRMED**: The issue exists as described. Include evidence (line numbers, code snippets).
   - **PARTIAL**: The issue exists but is less severe than described, or only affects some of the cited locations. Explain what's accurate and what's not.
   - **FALSE POSITIVE**: The issue does not exist, or the reviewer misread the code. Explain why.
   - **OUTDATED**: The issue may have existed but has since been fixed (e.g., by a later story). Show the fix.

3. If CONFIRMED or PARTIAL, assess fixability:
   - **auto-fixable**: Clear fix, no judgment calls needed (e.g., rename, add missing error handling, add test)
   - **needs-decision**: Multiple valid approaches, or fix has significant downstream impact
   - **complex**: Requires substantial refactoring or touches multiple subsystems

4. If auto-fixable, describe the specific fix needed (files, changes).

Return structured output:
```json
{
  "finding_id": "F-001",
  "verdict": "CONFIRMED | PARTIAL | FALSE_POSITIVE | OUTDATED",
  "evidence": "line 42 of src/auth.ts: uses localStorage instead of httpOnly cookie as required by D-003",
  "fixability": "auto-fixable | needs-decision | complex",
  "fix_description": "Replace localStorage token storage with httpOnly cookie in src/auth.ts:38-45",
  "decision_context": null | "Two valid approaches: httpOnly cookie vs server-side session. Cookie is simpler but session aligns with D-003 more strictly."
}
```
""",
)
```

---

## 4. Triage Results

After all validation agents return, sort findings into buckets:

### 4a. Discard False Positives

Remove findings with verdict `FALSE_POSITIVE` or `OUTDATED`. Log them:
```
Discarded {count} findings:
  F-003: FALSE POSITIVE — {reason}
  F-007: OUTDATED — {reason}
```

### 4b. Auto-Fix Queue

Findings with verdict `CONFIRMED` or `PARTIAL` and fixability `auto-fixable`:
- In default mode: apply fixes (section 5)
- In `--dry-run` mode: list what would be fixed
- In `--auto` mode: apply fixes without confirmation

### 4c. Decision Queue

Findings with fixability `needs-decision`. Present each as an elicitation:

```
## Fix Decision: {finding_id} — {description}

**Type**: {type}
**Severity**: {severity}
**Evidence**: {evidence from verifier}

### Options

**Option A: {fix approach 1}**
- Change: {specific change}
- Impact: {what it affects}
- Files: {file list}

**Option B: {fix approach 2}**
- Change: {specific change}
- Impact: {what it affects}
- Files: {file list}

**Option C: Accept as-is**
- Leave the code unchanged, document as a known issue

### Recommendation
{verifier's recommendation if any}

### Your Options
1. **Option A**
2. **Option B**
3. **Accept as-is** — skip this finding
4. **Auto-fix remaining** — use recommendations for all remaining decisions
```

In `--auto` mode: use the verifier's recommendation for all decisions.

### 4d. Complex Findings

Findings with fixability `complex`. Present to the user as informational:
```
## Complex Finding: {finding_id}

{description}
{evidence}

This finding requires substantial changes and is better addressed as a dedicated task.
Consider creating a story for the next sprint.
```

---

## 5. TDD Test Generation (if `--tdd`)

Skip this section if `--tdd` was not specified.

### 5a. Detect Test Framework

Read the project's existing test setup to determine:
- Test framework (vitest, jest, bun:test, pytest, go test, etc.)
- Test file naming convention and directory convention
- Any test config files

Use Glob to find existing test files and infer conventions. If no tests exist yet, ask the user which framework to use.

### 5b. Write Failing Tests for Confirmed Findings

For each finding in the auto-fix queue (verdict CONFIRMED or PARTIAL, fixability auto-fixable or needs-decision), dispatch a test-engineer agent to write a test that reproduces the issue:

```python
Agent(
    subagent_type="oh-my-claudecode:test-engineer",
    model="sonnet",
    prompt="""
You are writing regression tests for confirmed review findings.
Each test should FAIL against the current code, proving the issue exists.
After the fix is applied, the test should PASS.

## Test Framework
{detected_framework}
Test naming: {convention}
Test location: {directory_convention}

## Findings to Test

{for each finding in batch}
### {finding_id}: {description}
- Type: {type}
- Severity: {severity}
- Evidence: {evidence}
- Files: {file_list}
- Expected fix: {fix_description}
{end for}

## Instructions

1. Write one or more tests per finding that reproduce the issue.
2. Each test should:
   - Have a descriptive name mapping to the finding (e.g., `test_auth_uses_httponly_cookie_not_localstorage`)
   - Include a comment: `// Finding: {finding_id} — {description}`
   - Test the CORRECT behavior (what the code should do after the fix)
   - FAIL against the current implementation
3. Use the project's existing test patterns and utilities.
4. Do NOT fix the implementation — only write tests.
5. Write tests that will serve as regression guards after the fix.

## Output
- Write test files following project conventions
- Return a list of test files created and which findings each covers
""",
)
```

### 5c. Verify Tests Fail

Run the generated tests to confirm they fail:

```bash
{test_runner_command} {test_files}
```

- **Expected failure**: Correct — the test proves the issue exists
- **Unexpected pass**: Re-examine the finding — it may be a false positive after all
- **Error**: Attempt one auto-fix pass on the test

### 5d. Record Test Mapping

Track which tests correspond to which findings. This mapping is used in section 6 (Apply Fixes) to verify that fixes actually resolve the issues, and in section 7 (Write Report) for documentation.

---

## 6. Apply Fixes

For each finding in the auto-fix queue (and decided findings from 4c), dispatch an `executor` agent to apply the fix. Group fixes by file to avoid conflicts — all fixes touching the same file go to a single agent.

```python
Agent(
    subagent_type="oh-my-claudecode:executor",
    model="sonnet",
    prompt="""
Apply the following review fixes. Make ONLY the changes described — no other modifications.

## Fixes to Apply

{for each fix targeting files in this batch}
### {finding_id}: {description}
Files: {file_list}
Fix: {fix_description}
{end for}

## Constraints
- Make only the specified changes
- Do not refactor surrounding code
- Do not add comments explaining the fix (the review-fix report documents it)
- Preserve existing formatting and style
- If a fix conflicts with another fix in this batch, apply the higher-severity one and note the conflict

{if tdd}
## TDD Validation
After applying all fixes, run the test suite to verify fixes work:
  {test_runner_command} {test_files}
Report which tests now pass and which still fail.
{/if}

After applying, list each fix and whether it was applied successfully.
""",
)
```

### 6a. TDD Verification (if `--tdd`)

After fixes are applied, run the TDD tests again:

```bash
{test_runner_command} {test_files}
```

- **Now passing**: Fix is verified — the test serves as a regression guard
- **Still failing**: Fix didn't fully resolve the issue. Flag it in the report and mark the finding as `partial_fix`
- **New failures**: Fix broke something else. Flag as a side-effect for investigation

---

## 7. Write Report

Write the review-fix report to `SPRINT_DIR/reviews/{source}-fixes.md` (e.g., `epic-2-review-fixes.md`).

```markdown
# Review Fix Report: {source_review_file}
**Date**: {date}
**Findings processed**: {total}

## Summary
| Category | Count |
|----------|-------|
| Confirmed & fixed | {count} |
| Confirmed & decided by user | {count} |
| Confirmed & accepted as-is | {count} |
| Complex (deferred) | {count} |
| False positive (discarded) | {count} |
| Outdated (discarded) | {count} |
| TDD tests generated | {count or "—"} |
| TDD tests passing after fix | {count or "—"} |

## Fixes Applied

### {finding_id}: {description}
- **Verdict**: {verdict}
- **Fix**: {what was changed}
- **Files modified**: {list}

## Decisions Made

### {finding_id}: {description}
- **Chosen**: {option chosen}
- **Decided by**: user | auto
- **Fix**: {what was changed}

## Accepted As-Is
{list of findings user chose to skip}

## Deferred (Complex)
{list of complex findings with suggested next-sprint stories}

## False Positives Discarded
{list with reasons — useful for calibrating future reviews}
```

---

## 8. Report to User

```
═══════════════════════════════════════════════════
  REVIEW FIX COMPLETE
═══════════════════════════════════════════════════

  Findings:     {total} processed
  Fixed:        {count}
  Decided:      {count} (by user)
  Accepted:     {count} (as-is)
  Deferred:     {count} (complex — next sprint)
  Discarded:    {count} (false positives)

  {if tdd}
  TDD:
    Tests generated:        {count}
    Passing after fix:      {count}
    Still failing:          {count}
    Run: {test_runner_command} {test_files}
  {/if}

  Report: SPRINT_DIR/reviews/{source}-fixes.md

  {if deferred_count > 0}
  Note: {deferred_count} complex findings deferred.
  Consider adding them as stories in the next sprint.
  {/if}

  {if tdd_still_failing > 0}
  ⚠ {tdd_still_failing} fixes did not fully resolve their findings.
  Check the report for details.
  {/if}
═══════════════════════════════════════════════════
```
