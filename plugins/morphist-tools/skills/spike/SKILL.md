---
name: spike
description: >
  Build the smallest testable chunk to empirically validate or disprove a hypothesis.
  Use before landing a research-backed decision into production code.
  Triggers on: spike, validate hypothesis, prove this works, smallest testable piece.
user-invocable: true
argument-hint: "<hypothesis-id | freeform-claim> [--time-box=<minutes>] [--source=<synthesis-path>] [--out=<dir>]"
model: sonnet
---

# spike: Smallest-Testable-Chunk Validator

A skill that builds a minimal, isolated reproduction of a hypothesis and returns a **go / no-go / inconclusive** verdict backed by evidence (code, screenshot, log excerpt, test result).

Spikes exist to invert the cost curve: a 30-minute isolated test is cheaper than a half-day of production-code debugging after landing an unvalidated research recipe. Use this skill whenever a decision is about to be implemented but the underlying hypothesis has `confidence != high` OR `verification != empirically-tested` in its ultraresearch metadata.

---

## When to Use

- **Automatically** — `sprint-exec`'s spike-gate (section 3f) invokes this skill when a story's referenced decision cites an unvalidated hypothesis.
- **On-demand** — any time a user wants to prove out a claim before committing to it. ("Can we actually animate splats in PC 2.17?" → spike.)
- **During `/replan`** — when a replan proposes a new approach, spike it before landing.

Do **not** use this skill for:
- Production implementation (that's `sprint-exec` + executors)
- Broad research (that's `ultraresearch`)
- Debugging already-shipped code (that's the `debugger` agent)

---

## Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `HYPOTHESIS_ID` | *(required if `--source` provided)* | Identifier (e.g. `H3`) matching an entry in the referenced synthesis.md |
| `CLAIM` | *(required if no `HYPOTHESIS_ID`)* | Freeform one-sentence claim to test (user-invoked mode) |
| `TIME_BOX_MIN` | `60` | Hard time budget in minutes. Default 60; min 15; max 240. |
| `SOURCE` | *(context-dependent)* | Path to the synthesis.md whose hypothesis metadata should be updated on success |
| `OUT_DIR` | `docs/spikes/{id}/` | Where the finding + evidence live |
| `SPIKE_PROMPT` | *(read from synthesis)* | Agent's pre-defined minimal test. If absent, the skill asks the user to confirm one. |

### Resolution Rules

- If invoked with a bare hypothesis id (e.g. `/spike H3`), look up the most recent `.omc/sprint-plan/current/research_artifacts/*/synthesis.md` that mentions `H3` and pull its `spike-prompt`.
- If invoked with a freeform claim, the skill enters **interview mode**: asks the user to refine the claim + agree on the minimal test criteria, then proceeds.

---

## Workflow

### Stage 1 — Plan the Minimum Test (≤ 5 min)

Read the source synthesis (if provided). Extract the hypothesis block including `spike-prompt`, `version-pinned`, and `claim`.

If no pre-defined `spike-prompt` exists, draft one using the following template and confirm with the user:

```
Hypothesis: {claim}
Version pinned to: {version-pinned or "unpinned — this is a smell"}

Minimal test:
  Setup:     {the smallest scaffolding that can exercise the claim}
  Action:    {the one action that distinguishes go from no-go}
  Assertion: {the observable that proves/disproves the claim}
  Evidence:  {what the skill will capture — log, screenshot, test result}

Time box: {TIME_BOX_MIN} minutes.
Proceed? [yes/modify/cancel]
```

### Stage 2 — Build & Execute (≤ TIME_BOX_MIN)

Create an isolated scaffold:

- **Web/frontend claims**: scaffold a new route under `src/routes/spike/{id}/` (or equivalent for non-SvelteKit stacks) that imports the minimum set of modules needed. No production code changes.
- **Backend/library claims**: scaffold a new test file under `spikes/{id}.test.ts` (or equivalent) using the project's existing test runner. Run it; capture stdout.
- **CLI/tool claims**: scaffold a shell script or one-off program under `scripts/spike-{id}.{ext}`. Capture its output.
- **Source-code-only claims** (e.g. "does API X exist in library version Y?"): read the library source and capture a direct quote + line reference. This is still a valid spike outcome — it's faster than running code.

Run the test. Capture evidence. Do NOT modify production code. Do NOT commit to git yet.

If the test cannot run within the time box, return `inconclusive` with a note about what blocked the test (missing credentials, platform dependency, etc.).

### Stage 3 — Write the Finding

Create `OUT_DIR/finding.md` with this schema:

```markdown
---
spike_id: {id}
hypothesis_id: {H-id if applicable, else null}
source_synthesis: {path to synthesis.md or null}
run_at: {ISO 8601}
time_spent_min: {actual time}
verdict: go | no-go | inconclusive
version-pinned: {lib@version or null}
---

# Spike: {claim, trimmed}

## Claim Tested
{exact claim from source hypothesis or user input}

## Minimal Test
{describe the scaffolding and action in 3-5 sentences}

## Evidence
{embed or link the evidence — code, log excerpt, screenshot path, test output}

## Verdict: {go | no-go | inconclusive}

{one paragraph: what the evidence shows and why it supports the verdict}

{if no-go:}
## Why the Hypothesis Is Wrong
{specific failure mode observed}

## Recommended Replan
{suggest a follow-up — different API, different library, different decision}

{if inconclusive:}
## What Blocked Validation
{what prevented a definitive answer}

## Recommended Follow-Up
{what the next spike (or manual human effort) would need}
```

Alongside `finding.md`, preserve the raw evidence files in `OUT_DIR/`:
- `evidence.png` (screenshots)
- `evidence.log` (log excerpts)
- `evidence.code` (the minimal test source, preserved verbatim)
- `repro-command.sh` (the exact command used to run the test, for future re-verification)

### Stage 4 — Update Source Metadata (go only)

If verdict is **go** AND a source synthesis was provided:

1. Open `SOURCE/synthesis.md`.
2. Find the hypothesis block matching `hypothesis_id`.
3. Update:
   ```yaml
   confidence: high                         # was low|medium
   verification: empirically-tested         # was inferred-from-source|docs-only
   spike-required: false                    # auto-derived from the above
   spike-artifact: {relative path to OUT_DIR/finding.md}
   ```
4. If a matching `summary.json` exists in the synthesis dir, apply the same update to the `hypotheses` array entry.
5. Append a note under the hypothesis's prose: `Validated by spike {id} on {date}.`

Do NOT update on `no-go` or `inconclusive` — those outcomes should trigger `/replan`, not rubber-stamp the original decision.

### Stage 5 — Return to Caller

Report to the user (or calling skill):

```
Spike {id} complete.

Verdict:  {go | no-go | inconclusive}
Time:     {actual-min} of {time-box-min} min budget
Evidence: {OUT_DIR/finding.md}
{if go:}
Source synthesis updated: {SOURCE}/synthesis.md hypothesis {H-id} → confidence: high
Caller can now proceed with decision {D-NNN}.

{if no-go:}
Hypothesis disproved. Recommended next step:
  /replan --decision={D-NNN} --reason="{finding-summary}"

{if inconclusive:}
Spike could not reach a verdict. See {OUT_DIR/finding.md} § "What Blocked Validation".
Options:
  [retry with larger time-box] /spike {id} --time-box=120
  [escalate to human expert]   (no automation path — user must investigate manually)
  [override the gate]          /sprint-exec with spike-gate [override] option
```

---

## Agent-Callable Interface

Other skills invoke `/spike` by spawning an agent with these parameters in the prompt:

```
## Spike Request
- HYPOTHESIS_ID: <id matching synthesis>
- SOURCE: <path to synthesis.md>
- TIME_BOX_MIN: <minutes>
- OUT_DIR: <path>
- CALLER: <skill name — sprint-exec, replan, etc.>

Follow the spike skill workflow. Return structured output per the schema above.
```

### Output Contract

After completion, spike guarantees:

1. `OUT_DIR/finding.md` exists with `verdict: go|no-go|inconclusive` in frontmatter
2. Evidence files preserved alongside
3. `repro-command.sh` can re-run the test deterministically
4. On `go`: source synthesis + summary.json both updated
5. Machine-readable verdict surfaces on the last line of agent output: `VERDICT: go|no-go|inconclusive`

Callers parse that final `VERDICT:` line and dispatch accordingly.

---

## What Makes a Good Spike

Inspired by the three sprint-004 LoGAvatar spikes (`/spike/splat-anim`, `/spike/splat-perframe`, `/spike/splat-lbs`), which together unblocked a half-day of dead-ends in ~45 minutes:

- **One claim at a time**. If the hypothesis has two sub-claims, split into two spikes (they can chain: spike-A output becomes spike-B input).
- **Smallest possible scaffold**. No SvelteKit routing layers when a bare `<script>` + `<canvas>` will do. No full test harness when one `bun run` script suffices.
- **Observable in isolation**. If you can't tell the verdict from the spike alone (need side knowledge of the production codebase), the spike is too entangled.
- **Evidence beats prose**. A 100KB screenshot of the splat animating beats a paragraph arguing it should work.
- **Kept, not deleted**. Spike code under `src/routes/spike/` or `spikes/` stays as reference after the production code lands. It's cheap documentation of why the production approach is correct.

---

## Anti-Patterns (don't do these)

- ❌ Spiking inside production modules ("I'll just add a flag to toggle the new behavior")
- ❌ Extending the time box to "finish the feature" in spike mode — if it's not verdict-ready at the time box, declare `inconclusive` and hand off
- ❌ Declaring `go` on a test that ran once without asserting the observable (e.g. "no error was thrown" ≠ "the thing works")
- ❌ Treating spike code as the production implementation. Spikes are throwaway-shaped even when kept — production code needs the full scene contract, tests, reviews, a11y, etc.

---

## Output Structure

```
{{OUT_DIR}}/
├── finding.md           # Verdict + narrative
├── evidence.png         # Screenshot (if applicable)
├── evidence.log         # Captured output (if applicable)
├── evidence.code        # Minimal test source preserved verbatim
└── repro-command.sh     # Exact command to re-run the test
```

---

## Integration Points

- **`ultraresearch`** — outputs `spike-prompt` field in hypothesis metadata; this skill consumes it.
- **`sprint-exec`** — section 3f spike-gate invokes this skill automatically; consumes `VERDICT:` line.
- **`replan`** — on `no-go`, this skill's finding.md feeds `/replan --reason=...`.
- **`sprint-plan`** — Phase 3.5 write-stories gate may suggest spikes for flagged research-backed stories.

---

## Error Handling

- **Cannot scaffold** (tooling missing, platform dependency): return `inconclusive` with clear remediation note
- **Time box exceeded**: cut losses, return `inconclusive` with partial evidence
- **User cancels at Stage 1 confirmation**: write `OUT_DIR/finding.md` with `verdict: cancelled` and return cleanly
- **Source synthesis not found** but `HYPOTHESIS_ID` provided: enter interview mode, treat as freeform claim
