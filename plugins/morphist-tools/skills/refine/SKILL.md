---
name: refine
description: Refine any sprint artifact with Planner-Architect-Critic consensus. Scope to a phase, epic, or story. When targeting an epic, enables interactive deep-dive with decision graph propagation.
user-invocable: true
argument-hint: "[<phase>] [--epic=N] [--story=N.M] [--graph] [--propagate] [--force] [--sprint=ID]"
---

# refine: Artifact Refinement & Epic Deep-Dive

Triggers a RALPLAN-DR consensus pass (Planner → Architect → Critic) on a specified artifact. When scoped to an epic, provides an interactive deep-dive session with decision graph propagation — the pre-execution preparation workflow.

---

## 1. Argument Parsing

Parse `$ARGUMENTS` for:

1. **Phase name** (optional): First positional argument. Valid values: `requirements`, `sprint-scoping`, `architecture`, `epics`, `stories`, `enrichment`, `prd`, `retro`
2. **`--epic=N`** (optional): Scope to a specific epic. When used without a phase name, enters **interactive deep-dive mode** (section 10). When used with a phase, scopes the consensus pass to that epic.
3. **`--story=N.M`** (optional): Scope to a specific story. Valid for `enrichment` phase or standalone deep-dive.
4. **`--graph`** (optional): Display the decision graph for the epic/story and exit
5. **`--propagate`** (optional): After refinement, auto-run `/reconcile --decisions` to ripple ADR changes to dependent stories
6. **`--force`** (optional): Bypass the 2-pass limit on refinement invocations

**Mode detection**:
- Phase name provided → **consensus mode** (sections 2-9, the RALPLAN-DR pass)
- `--epic=N` without phase → **interactive deep-dive mode** (section 10)
- `--story=N.M` without phase → **story deep-dive** (section 10, jumps to story drill-down)
- Nothing → show usage

**Scope validation** (consensus mode):
- `--epic` is valid for: `epics`, `stories`, `enrichment` phases
- `--story` is valid for: `enrichment` phase only
- If used with an invalid phase: `--epic is not supported for the "{phase}" phase.`
- `--epic` and `--story` are mutually exclusive. If both provided, `--story` wins and `--epic` is inferred.

If no arguments provided:
```
Usage: refine [<phase>] [--epic=N] [--story=N.M] [--propagate] [--force]

Consensus mode (refine a phase artifact):
  refine architecture                    # Refine architecture decisions
  refine requirements                    # Refine requirements
  refine sprint-scoping                  # Re-evaluate sprint boundary
  refine epics --epic=2                  # Refine Epic 2's design
  refine enrichment --story=3.1          # Refine a single story's spec

Interactive deep-dive (pre-execution preparation):
  refine --epic=3                        # Deep dive into Epic 3
  refine --story=3.2                     # Deep dive into a story
  refine --epic=3 --graph                # Show decision graph for Epic 3
  refine --epic=3 --propagate            # Deep dive + auto-reconcile after

Valid phases: requirements, sprint-scoping, architecture, epics, stories, enrichment, prd, retro
```

---

## 2. Sprint Resolution & Phase Validation

### Sprint Resolution

Resolve the target sprint directories:
1. If `--sprint=<id>` was provided in `$ARGUMENTS`, set `STATE_DIR` = `.omc/sprint-plan/<id>/`
2. Else if `state_read` is available, read key `morphist.active_sprint`. If set, `STATE_DIR` = `.omc/sprint-plan/<value>/`
3. Else if `.omc/sprint-plan/current` symlink exists, `STATE_DIR` = `.omc/sprint-plan/current/`
4. Otherwise halt: "No active sprint found. Run `/sprint-plan` first, or pass `--sprint=<id>`."

Verify `STATE_DIR/phase-state.json` exists. Read it and set `SPEC_DIR` from the `spec_dir` field.
Verify `SPEC_DIR` exists. If not, halt: "Sprint spec directory not found at {spec_dir}. Sprint may need re-initialization."

### Validate Phase Completion

Check that the specified phase has been completed:

| Phase Argument | Completed When `current_phase` Is... |
|----------------|--------------------------------------|
| prd | Always valid (standalone artifact) |
| requirements | requirements, sprint-scoping, architecture, epic-design, story-decomposition, story-enrichment, validation |
| sprint-scoping | sprint-scoping, architecture, epic-design, story-decomposition, story-enrichment, validation |
| architecture | architecture, epic-design, story-decomposition, story-enrichment, validation |
| epics | epic-design, story-decomposition, story-enrichment, validation |
| stories | story-decomposition, story-enrichment, validation |
| enrichment | story-enrichment, validation |
| retro | Always valid (check `retrospective_status: "complete"` in phase-state.json) |

If the phase has NOT been completed:
```
Phase "{phase}" has not been completed yet. Complete this phase first before refining it.
```

---

## 3. Check Refinement Pass Limit

Read `refine_passes` (or legacy `ral_passes`) from `phase-state.json`.

**Pass tracking key**: When a scope flag is used, track passes with a scoped key:
- `--epic=N` → key is `{phase}:epic-{N}` (e.g., `epics:epic-2`)
- `--story=N.M` → key is `enrichment:story-{N.M}` (e.g., `enrichment:story-3.1`)
- No scope → key is `{phase}`

If `refine_passes[key] >= 2` AND `--force` is NOT present:
```
This {scope description} has been refined 2 times. Further refinement has diminishing returns.
Override with: refine --force {phase}
```

If `--force` is present, proceed regardless.

---

## 4. Load Phase Artifact

Load ONLY the artifact file for the specified phase (context shedding).

**Phase-to-artifact mapping**:

| Phase | Artifact Path |
|-------|--------------|
| prd | `.omc/sprint-plan/prd-{slug}.md` (most recent by mtime) |
| requirements | `SPEC_DIR/requirements.md` |
| sprint-scoping | `SPEC_DIR/sprint-scope.md` |
| architecture | `SPEC_DIR/architecture-decisions.md` |
| epics | `SPEC_DIR/epics.md` |
| stories | `SPEC_DIR/epics.md` (stories sections) |
| enrichment | All files in `SPEC_DIR/stories/` |
| retro | `SPEC_DIR/retrospective.md` |

### Scoped artifact loading

**`epics` or `stories` with `--epic=N`**:
- Load `epics.md`, extract only the `## Epic N: ...` section

**`enrichment` with `--epic=N`**:
- Load only story files matching `{N}-*` from `current/stories/`

**`enrichment` with `--story=N.M`**:
- Load only the matching story file from `current/stories/`

If the scoped artifact cannot be found:
```
Could not find {scope description} in the artifact. Check that the epic/story number is correct.
```

---

## 5. Run RALPLAN-DR Consensus

Three sequential agent passes. Each receives the artifact and outputs from prior agents.

When scoped, include: `Scope: Epic {N}` or `Scope: Story {N.M}` in each prompt.

### 5a. Planner Pass

Dispatch to `planner` (opus):

```
You are reviewing a sprint planning artifact for quality improvements.

Phase: {phase}
{if scoped}Scope: {scope_description}{/if}
Artifact:
---
{artifact_content}
---

Review this {phase} artifact{if scoped} (scoped to {scope_description}){/if}. Identify weaknesses, gaps, inconsistencies, or quality issues. Propose specific improvements with rationale for each. Focus on substance over style.

For each proposed improvement, provide:
- What to change (be specific — quote or reference the exact section)
- Why it should change (the quality problem it addresses)
- What the improved version should say or contain

Do not rewrite the entire artifact. Propose targeted, discrete improvements only.
```

### 5b. Architect Pass

Dispatch to `architect` (opus), with artifact + planner output:

```
You are adversarially reviewing proposed improvements to a sprint planning artifact.

Phase: {phase}
{if scoped}Scope: {scope_description}{/if}
Original Artifact:
---
{artifact_content}
---

Proposed Improvements from Planner:
---
{planner_output}
---

For each proposed improvement, argue the strongest case AGAINST making it. Identify:
- Which proposed changes would actually degrade the artifact
- Which are unnecessary (the artifact already handles this adequately)
- Which introduce new problems or inconsistencies
- Which are genuinely valuable and you cannot refute

Be adversarial but fair. Your goal is to prevent unnecessary changes, not to reject everything.

For each improvement, verdict: OPPOSE (state your strongest objection) or CONCEDE (you cannot argue against it).
```

### 5c. Critic Pass

Dispatch to `critic` (opus), with artifact + planner + architect:

```
You are the final arbiter determining which changes should be applied to a sprint planning artifact.

Phase: {phase}
{if scoped}Scope: {scope_description}{/if}
Original Artifact:
---
{artifact_content}
---

Planner's Proposed Improvements:
---
{planner_output}
---

Architect's Challenges:
---
{architect_output}
---

For each proposed improvement, weigh the planner's rationale against the architect's challenge and issue a verdict:
- APPLY: The improvement is valuable and the architect's objection does not hold
- REJECT: The architect's challenge reveals the improvement would be net negative or unnecessary
- MODIFY: The improvement is valuable but needs adjustment per the architect's feedback (specify the adjusted version)

After issuing all verdicts, produce a consolidated list of:
1. All APPLY changes (with exact new text or instruction)
2. All MODIFY changes (with adjusted text)
3. All REJECT changes (with brief reason)

If there are no APPLY or MODIFY changes, state: "No changes recommended. The artifact is already at high quality for this iteration."
```

---

## 6. Apply Consensus

### If consensus produced APPLY or MODIFY verdicts:

Apply all APPLY and MODIFY changes to the artifact. Do not change untargeted sections.

**Scoped write-back**:
- `--epic=N` on `epics`/`stories`: Replace only the `## Epic N` section in `epics.md`
- `--epic=N` on `enrichment`: Write each modified story file back to its original path
- `--story=N.M`: Write modified content back to the story file
- No scope: Write back to the same path

**Decision graph maintenance**: If the phase is `architecture` and changes were applied (decisions added, removed, or renamed), update `current/decision-graph.md` if it exists:
- Decision renamed: update the heading in the graph
- Decision removed: remove its section
- Decision added: add section with empty story list
- Graph doesn't exist: skip

Similarly, if `enrichment` and story `decisions` frontmatter changed, update graph entries.

### If no consensus (no APPLY or MODIFY):

Do not modify the artifact. Report:
```
Refinement complete. No changes to apply — the artifact is at high quality for this iteration.
```

### If irreconcilable disagreement:

Present to the user using Decision Steering elicitation format, wait for input.

---

## 7. Propagation

After changes are applied, handle propagation:

### 7a. Auto-propagate (`--propagate`)

If `--propagate` was specified AND architecture decisions were modified:
1. Run the equivalent of `/reconcile --decisions` — propagate ADR changes to dependent story specs
2. Report which stories were updated

### 7b. Suggest propagation

If architecture decisions were modified but `--propagate` was NOT specified:
```
D-{NNN} revised. {count} stories in other epics depend on this decision.
Run: /refine --propagate    (auto-reconcile next time)
  or: /reconcile --decisions (manual reconcile now)
```

### 7c. Stale phase marking

If the artifact was modified, mark downstream phases as stale:

| Phase Refined | Phases Marked Stale |
|--------------|-------------------|
| prd | None (upstream of sprint-plan) |
| requirements | sprint-scoping, architecture, epic-design, story-decomposition, story-enrichment, validation |
| sprint-scoping | architecture, epic-design, story-decomposition, story-enrichment, validation |
| architecture | epic-design, story-decomposition, story-enrichment, validation |
| epics | story-decomposition, story-enrichment, validation |
| stories | story-enrichment, validation |
| enrichment | validation |
| retro | None (terminal artifact) |

---

## 8. Update State

1. Read `phase-state.json`
2. Increment `refine_passes[key]` by 1 (also check legacy `ral_passes` key for backwards compat)
3. If artifact modified, update `stale_phases`
4. Write back

---

## 9. Report Results (Consensus Mode)

```
--- Refinement Complete: {phase}{if scoped} ({scope_description}){/if} ---
Pass: {refine_passes[key]} of 2 (use --force to exceed)

Changes applied: {count}
Changes rejected: {count}

Applied:
{list each with 1-line description}

Rejected:
{list each with 1-line reason}

{if artifact modified}
Downstream phases marked stale: {list}
{/if}

{if adr_changes and not propagate}
ADR changes detected — run /refine --propagate or /reconcile --decisions
{/if}

{if refine_passes[key] == 2}
Note: This scope has been refined 2 times. Further calls require --force.
{/if}
```

---

## 10. Interactive Deep-Dive Mode

Activated when `--epic=N` (or `--story=N.M`) is used without a phase name. This is the pre-execution preparation workflow — an interactive session for drilling into an epic before execution.

### 10a. Load Context

Read from `STATE_DIR/`:
- `phase-state.json` — execution status, epic status

Read from `SPEC_DIR/`:
- `architecture-decisions.md` — current ADRs
- `epics.md` — epic definitions and dependencies
- `requirements.md` — traced requirements
- `decision-graph.md` — decision dependency graph (create if missing, see section 12)
- `replan-log.md` — prior replans (if exists)

Read all story files for the target epic from `current/stories/`.

If no `--epic` provided and no phase provided, determine the next epic:
1. Read `phase-state.json` for `epic_status`
2. Find the first epic with status `"pending"` or not yet tracked
3. Confirm: `Prepare Epic {N}: {title}?`

### 10b. Epic Overview

Present the epic with decision graph context:

```
═══════════════════════════════════════════════════
  EPIC {N}: {title}
  Status: {status}
═══════════════════════════════════════════════════

  Goal: {epic goal from epics.md}

  Stories:
    {N}.1: {title} [{status}]
    {N}.2: {title} [{status}]
    {N}.3: {title} [{status}]

  ─────────────────────────────────────────────────
  Decision Graph (ADRs affecting this epic):

    D-003: ky for HTTP client
      → {N}.1, {N}.3, {other_epic}.{story}

    D-007: Zustand for client state
      → {N}.2

  ─────────────────────────────────────────────────

  Commands:
    story {M}       — drill into story {N}.{M}
    enrich          — auto-enrich all stories
    consensus       — run RALPLAN-DR consensus on this epic
    graph           — show full decision graph
    ready           — finalize, show reconcile suggestion if needed
    quit            — exit
═══════════════════════════════════════════════════
```

Wait for user input. This is interactive.

### 10c. Story Drill-Down (`story {M}` or `--story=N.M`)

Present story in full with decision context:

```
═══════════════════════════════════════════════════
  STORY {N}.{M}: {title}
  Status: {status}
═══════════════════════════════════════════════════

  Acceptance Criteria:
    1. {AC-1}
    2. {AC-2}

  Architecture Decisions:
    D-003: ky for HTTP client — {summary}

  {if dev_agent_record}
  Previous Implementation:
    Files: {file list}
    Notes: {completion notes}
    Blocker: {blocker info, if any}
  {/if}

  ─────────────────────────────────────────────────
  Commands:
    enrich          — auto-enrich with implementation details
    consensus       — run RALPLAN-DR on this story
    edit ac         — modify acceptance criteria
    edit tech       — modify technical notes
    edit decision   — propose an ADR change
    add note        — add a prep note for the executor
    back            — return to epic overview
═══════════════════════════════════════════════════
```

### 10d. Story Commands

**`enrich`** — Dispatch architect agent (opus) to add implementation guidance:

```python
Agent(
    subagent_type="oh-my-claudecode:architect",
    model="opus",
    prompt="""
You are enriching a story specification before implementation.

## Story
{story_file_content}

## Architecture Decisions
{relevant_decisions_full_text}

## Existing Codebase Context
{discovery.md content if available}

## Task

Review this story and add concrete implementation guidance:

1. **File Plan**: Specific files to create/modify with brief descriptions.
2. **Implementation Sequence**: Order of implementation within this story.
3. **Edge Cases**: Ambiguities in ACs the executor should handle.
4. **Integration Points**: Where this story touches other stories' files or APIs.
5. **Risk Areas**: Things that might not work as expected.

Do NOT modify acceptance criteria or change the story's scope.
Return ONLY the enrichment content for a ## Prep Notes section.
""",
)
```

Insert under `## Prep Notes` (before Dev Agent Record). If section exists, append under `### Update: {timestamp}`.

**`consensus`** — Run the full RALPLAN-DR consensus pass (sections 5-6) scoped to this story's enrichment artifact.

**`edit ac`** — Present ACs for interactive modification. Update story file.

**`edit tech`** — Present technical notes for revision. Update story file.

**`edit decision`** — ADR revision flow:
1. Show ADRs affecting this story
2. User selects which to revise
3. Capture proposed change
4. Dispatch architect agent to formally revise
5. Update `architecture-decisions.md`
6. Update decision graph (section 12)
7. Track change for propagation (section 10f)

**`add note`** — Append free-text note under `## Prep Notes`.

### 10e. Bulk Enrich (`enrich` from epic overview)

Dispatch architect agents in parallel (one per story, up to 5 concurrent). Same prompt as story-level enrich. Present summary, return to overview.

### 10f. Decision Change Tracking

Track all ADR changes during the session:

```json
{
  "changes": [
    {
      "decision": "D-003",
      "title": "HTTP client library",
      "old_summary": "Use Eden Treaty",
      "new_summary": "Use ky",
      "changed_during_story": "3.2",
      "affected_stories_in_epic": ["3.1", "3.3"],
      "affected_stories_other_epics": ["4.1", "4.2"]
    }
  ]
}
```

### 10g. Finalize (`ready`)

```
═══════════════════════════════════════════════════
  EPIC {N} REFINEMENT COMPLETE
═══════════════════════════════════════════════════

  Stories enriched:     {count}/{total}
  ACs modified:         {count} stories
  Tech notes updated:   {count} stories
  ADRs revised:         {count}
  Prep notes added:     {count}

  {if adr_changes}
  ─────────────────────────────────────────────────
  ADR Changes Made:

    D-003: Eden Treaty → ky
      Stories in this epic: 3.1, 3.3
      Stories in other epics: 4.1, 4.2

  {if propagate}
  Auto-propagating changes to dependent stories...
  {else}
  ⚠ Decision changes may affect stories in other epics.
    Run: /reconcile --decisions
  {/if}
  {/if}

  Ready to execute:
    /sprint-exec --epic={N}
═══════════════════════════════════════════════════
```

If `--propagate`, auto-run reconcile. Otherwise suggest it.

---

## 11. `--graph` Mode

When `--graph` is provided with `--epic=N` or `--story=N.M`:

1. Read `decision-graph.md`
2. Extract ADRs relevant to the scope
3. Display the graph and exit (no interactive session)

```
Decision Graph for Epic {N}:

  D-003: ky for HTTP client
    → 3.1: API client setup
    → 3.3: Data fetching
    → 4.1: Payment integration (other epic)

  D-012: Zod validation
    → 3.1: API client setup
    → 3.2: Form validation
```

---

## 12. Decision Graph

The decision graph is a persistent artifact at `SPEC_DIR/decision-graph.md` mapping architecture decisions to dependent stories.

### 12a. Format

```markdown
# Decision Graph

Maps architecture decisions to dependent stories. Updated by /sprint-plan, /refine, /replan.

## D-001: {decision title}
- 1.1: {story title}
- 1.3: {story title}

## D-003: {decision title}
- 2.1: {story title}
- ~4.3: Payment webhook handler~ *(inferred — uses HTTP client pattern)*
```

Tilde-wrapped entries are inferred from content scans (not frontmatter) and should be confirmed or removed.

### 12b. Maintenance

Updated by:
- **`/sprint-plan`** (Phase 5): Seeds the initial graph from story frontmatter
- **`/refine`**: Updates when ADRs are revised or enrichment discovers new dependencies
- **`/replan`**: Updates when decisions are revised mid-sprint
- **`/reconcile --decisions`**: Reads the graph to find affected stories

### 12c. Deep Scan

When the graph may be incomplete, scan all story files and ADRs:
1. Check `decisions` frontmatter references (explicit)
2. Search story body for D-NNN mentions
3. Search for library/pattern names from decisions
4. Mark entries as explicit or inferred (tilde-wrapped)

### 12d. Bootstrap

If `decision-graph.md` doesn't exist when `/refine --epic=N` is invoked:
1. Run deep scan to build the initial graph
2. Present to user for review
3. Write to `current/decision-graph.md`

---

## 13. Integration Points

- **`/sprint-plan`**: Inter-phase summary options include `refine {phase}` instead of `ral {phase}`
- **`/sprint-exec`**: Run `/refine --epic=N` before `/sprint-exec --epic=N` for prepared execution
- **`/replan`**: For mid-execution ADR breakage. `/refine` is for deliberate pre-execution refinement. Both update the decision graph.
- **`/reconcile`**: `/refine --propagate` auto-triggers reconcile. Manual: `/reconcile --decisions` after refinement.
- **`/audit`**: After executing a refined epic, `/audit` can verify that prep notes and enrichments were followed.
- **`/status`**: Dashboard shows decision graph and prep status per epic.
