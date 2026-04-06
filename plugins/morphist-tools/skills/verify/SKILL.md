---
name: verify
description: Quick independent verification that an epic's stories were actually built. Checks file existence, build health, and AC coverage without the full depth of `/audit`.
user-invocable: true
argument-hint: "[--epic=N] [--story=N.M] [--all] [--sprint=ID]"
---

# verify: Lightweight Epic Completion Gate

Quick, independent verification that stories were actually implemented. A fresh agent checks file existence, build health, and AC satisfaction with one-line verdicts — no deep analysis, no work plans, no TDD.

Use this as a smoke test between epics, or before declaring a sprint done. For deep analysis, use `/audit` instead.

---

## 1. Argument Parsing

Parse `$ARGUMENTS`:

| Flag | Required | Description |
|------|----------|-------------|
| `--epic=N` | No | Verify all done stories in epic N |
| `--story=N.M` | No | Verify a single story |
| `--all` | No | Verify all done stories in the sprint |

If no arguments provided, default to the **most recently completed epic** — the highest-numbered epic with `epic_status: "done"` or `"partial"` in `phase-state.json`.

If no completed epics exist:
```
No completed epics found. Run /sprint-exec first.
```

---

## 1a. Sprint Resolution

Resolve the target sprint directories:
1. If `--sprint=<id>` was provided in `$ARGUMENTS`, set `STATE_DIR` = `.omc/sprint-plan/<id>/`
2. Else if `state_read` is available, read key `morphist.active_sprint`. If set, `STATE_DIR` = `.omc/sprint-plan/<value>/`
3. Else if `.omc/sprint-plan/current` symlink exists, `STATE_DIR` = `.omc/sprint-plan/current/`
4. Otherwise halt: "No active sprint found. Run `/sprint-plan` first, or pass `--sprint=<id>`."

Verify `STATE_DIR/phase-state.json` exists. Read it and set `SPEC_DIR` from the `spec_dir` field.
Verify `SPEC_DIR` exists. If not, halt: "Sprint spec directory not found at {spec_dir}. Sprint may need re-initialization."

---

## 2. Gather Context

### 2a. Read Sprint Artifacts

Read:
- `STATE_DIR/phase-state.json`
- `SPEC_DIR/architecture-decisions.md`

### 2b. Collect Stories to Verify

Based on scope, collect story files. Only verify stories with `status: done`.

For each story, read:
- Full story file (spec + Dev Agent Record)
- Extract from Dev Agent Record: `File List`, `Completion Notes`, `Agent Model Used`
- Extract from spec: acceptance criteria, `decisions` frontmatter

If a story has no Dev Agent Record or the Dev Agent Record has no File List, flag it immediately as `NO_RECORD`.

---

## 3. Verify Each Story

Dispatch a **single verifier agent** (sonnet) for the entire epic's stories. This is deliberately lightweight — one agent, one pass, fast.

```
You are performing a quick completion check on implemented stories.
This is a smoke test, NOT a deep audit. Be fast and decisive.

## Stories to Verify

{for each story}
### Story {N.M}: {title}

Acceptance Criteria:
{list each AC}

Files Created (from Dev Agent Record):
{file_list}

Architecture Decisions: {decisions list}

Completion Notes:
{completion_notes}
{end for}

## Verification Tasks

For each story, check the following. Be concise — one line per check.

### Check 1: Files Exist
For each file in the Dev Agent Record's File List:
- Does the file exist? (use Glob to verify)
- If the file doesn't exist, flag immediately

### Check 2: Build Health
If the project has a build step (check for package.json scripts, Makefile, Cargo.toml, etc.):
- Can you detect any obvious import errors in the created files? (missing imports, importing nonexistent modules)
- Do NOT run the full build — just check imports and basic syntax

### Check 3: AC Spot Check
For each acceptance criterion:
- Read the relevant file(s) from the File List
- Can you see evidence that this AC was addressed? One line: YES / PARTIAL / NO / CANT_TELL
- Do NOT do deep analysis — this is a quick scan

### Check 4: Architecture Compliance (quick)
For each decision referenced by the story:
- Does the implementation use the chosen approach? (e.g., if D-003 says "use ky", check if ky is imported)
- One line: COMPLIANT / VIOLATION / CANT_TELL

## Output Format

Return a JSON object:
{
  "stories": [
    {
      "id": "N.M",
      "title": "...",
      "verdict": "PASS|CONCERNS|FAIL",
      "files_exist": { "total": N, "found": N, "missing": ["list"] },
      "build_health": "ok|issues_found",
      "build_issues": ["list of issues if any"],
      "ac_results": [
        { "ac": "description", "status": "YES|PARTIAL|NO|CANT_TELL" }
      ],
      "arch_compliance": [
        { "decision": "D-NNN", "status": "COMPLIANT|VIOLATION|CANT_TELL" }
      ],
      "concerns": ["list of any issues found"]
    }
  ]
}

Verdict rules:
- PASS: All files exist, no build issues, all ACs are YES
- CONCERNS: Minor issues — some ACs are PARTIAL or CANT_TELL, or minor build issues
- FAIL: Missing files, ACs are NO, or architecture violations
```

---

## 4. Display Results

After the verifier returns, display a clear summary:

```
═══════════════════════════════════════════════════
  VERIFY: Epic {N} — {epic_title}
═══════════════════════════════════════════════════

  Story {N.1}: {title}                          PASS ✓
    Files: {found}/{total}  ACs: {yes}/{total}  Arch: compliant

  Story {N.2}: {title}                      CONCERNS ◎
    Files: {found}/{total}  ACs: {yes}/{total} ({partial} partial)
    → {concern description}

  Story {N.3}: {title}                          FAIL ✗
    Files: {found}/{total} — missing: {file1}, {file2}
    ACs: {yes}/{total} ({no_count} not met)
    → {concern description}

  ─────────────────────────────────────────────────
  Summary: {pass_count} passed, {concerns_count} concerns, {fail_count} failed
  ─────────────────────────────────────────────────

  {if all pass}
  ✓ Epic {N} verified. Ready to proceed.
  {/if}

  {if concerns or failures}
  Next steps:
    {if concerns}
    Review concerns — may be fine, or run /audit --epic={N} for deep analysis
    {/if}
    {if failures}
    /sprint-exec --story={N.M}     # Re-run failed stories
    /audit --story={N.M}     # Deep audit before re-running
    {/if}
  {/if}
═══════════════════════════════════════════════════
```

---

## 5. Update State

After verification completes:

1. Read `phase-state.json`
2. Add or update `verification_log` array with an entry per verified story:
   ```json
   {
     "story": "N.M",
     "date": "ISO 8601",
     "verdict": "PASS|CONCERNS|FAIL",
     "files_found": N,
     "files_total": N,
     "acs_met": N,
     "acs_total": N
   }
   ```
3. Write `phase-state.json`

---

## 6. Verify All (`--all`)

When `--all` is specified, verify every done story across all epics. Group output by epic:

```
═══════════════════════════════════════════════════
  VERIFY: Full Sprint
═══════════════════════════════════════════════════

  Epic 1: {title}
    Story 1.1: {title}                          PASS ✓
    Story 1.2: {title}                          PASS ✓
    Epic 1: 2/2 passed ✓

  Epic 2: {title}
    Story 2.1: {title}                      CONCERNS ◎
    Story 2.2: {title}                          PASS ✓
    Story 2.3: {title}                          FAIL ✗
    Epic 2: 1/3 passed, 1 concerns, 1 failed

  ─────────────────────────────────────────────────
  Sprint total: 3/5 passed, 1 concerns, 1 failed
═══════════════════════════════════════════════════
```

---

## 7. Integration Points

### With `/sprint-exec`

`/sprint-exec` auto-dispatches `/verify --epic=N` after each epic completes (section 4f-post). The verification runs inline (not background) so the results are visible before the next epic starts.

- If all stories PASS: proceed to next epic automatically
- If CONCERNS: show results, proceed to next epic (concerns are non-blocking)
- If any FAIL in non-`--full-auto` mode: pause and ask the user whether to proceed or fix first
- If any FAIL in `--full-auto` mode: log the failures and proceed

### With `/audit`

`/verify` is the quick gate; `/audit` is the deep dive. Typical flow:
1. `/verify --epic=2` → spots concerns
2. `/audit --epic=2` → deep analysis of the concerns
3. `/sprint-exec --story=2.3` → re-run what needs fixing

### With `/retro`

The verification log in `phase-state.json` is available to `/retro` for velocity analysis — how many stories passed verification on first try vs needed rework.
