---
name: ral
description: On-demand RALPLAN-DR refinement pass for any sprint-plan phase output. Triggers Planner→Architect→Critic consensus on a phase artifact, with optional --epic=N or --story=N.M scoping.
user-invocable: true
argument-hint: "<phase> [--epic=N] [--story=N.M] [--force]"
---

# RAL: Refinement at Level

You are the `ral` skill for the sprint-plan plugin. You trigger a RALPLAN-DR consensus pass (Planner → Architect → Critic) on a specified phase's output artifact.

---

## 1. Argument Parsing

Parse `$ARGUMENTS` for:

1. **Phase name** (required): First positional argument. Valid values: `requirements`, `architecture`, `epics`, `stories`, `enrichment`, `prd`, `retro`
2. **`--epic=N`** (optional): Scope refinement to a specific epic. Valid for phases: `epics`, `stories`, `enrichment`
3. **`--story=N.M`** (optional): Scope refinement to a specific story. Valid for phase: `enrichment` only. When provided, `--epic` is inferred from N.
4. **`--force` flag** (optional): Bypasses the 2-pass limit on ral invocations

**Scope validation**:
- `--epic` is only valid for `epics`, `stories`, and `enrichment` phases. If used with other phases, respond: `--epic is not supported for the "{phase}" phase.`
- `--story` is only valid for `enrichment`. If used with other phases, respond: `--story is not supported for the "{phase}" phase. Use --epic=N to scope epics or stories phases.`
- `--epic` and `--story` are mutually exclusive. If both are provided, `--story` wins and `--epic` is inferred.

If no phase is provided, respond:
```
Usage: ral <phase> [--epic=N] [--story=N.M] [--force]

Valid phases: requirements, architecture, epics, stories, enrichment, prd, retro

Scoping:
  --epic=N      Refine only epic N (epics, stories, enrichment phases)
  --story=N.M   Refine only story N.M (enrichment phase only)

Examples:
  ral architecture
  ral epics --epic=2
  ral enrichment --story=3.1
  ral prd
  ral retro
```

If an invalid phase is provided, respond with the same usage message and list valid phases.

---

## 2. Validate Phase Completion

Read `.omc/sprint-plan/current/phase-state.json`.

If the file does not exist, respond:
```
No active sprint found. Run /sprint-plan first to initialize a sprint.
```

Check that the specified phase has been completed. Use this mapping to determine which `current_phase` values indicate completion:

| Phase Argument | Completed When `current_phase` Is... |
|----------------|--------------------------------------|
| prd | Always valid (PRD is a standalone artifact, not gated by current_phase) |
| requirements | requirements, architecture, epic-design, story-decomposition, story-enrichment, validation |
| architecture | architecture, epic-design, story-decomposition, story-enrichment, validation |
| epics | epic-design, story-decomposition, story-enrichment, validation |
| stories | story-decomposition, story-enrichment, validation |
| enrichment | story-enrichment, validation |
| retro | Always valid (retrospective is a terminal artifact, check `retrospective_status: "complete"` in phase-state.json) |

If the phase has NOT been completed, respond:
```
Phase "{phase}" has not been completed yet. Complete this phase first before refining it.
```

---

## 3. Check RAL Pass Limit

Read `ral_passes` from `phase-state.json`.

**Pass tracking key**: When a scope flag is used, track passes with a scoped key instead of the bare phase name:
- `--epic=N` → key is `{phase}:epic-{N}` (e.g., `epics:epic-2`)
- `--story=N.M` → key is `enrichment:story-{N.M}` (e.g., `enrichment:story-3.1`)
- No scope → key is `{phase}` (unchanged from current behavior)

Look up `ral_passes[key]`. If `ral_passes[key] >= 2` AND the `--force` flag is NOT present, respond:
```
This {scope description} has been refined 2 times. Further refinement has diminishing returns. Proceed to the next phase, or override with `ral --force {phase}`.
```

If `--force` is present, proceed regardless of the count.

---

## 4. Load Phase Artifact

Load ONLY the artifact file for the specified phase. Do not rely on memory of prior agent responses. This is context shedding — the artifact is the only input to the refinement pass.

**Phase-to-artifact mapping**:

| Phase | Artifact Path |
|-------|--------------|
| prd | `.omc/sprint-plan/prd-{slug}.md` (most recent by mtime if no path given) |
| requirements | `.omc/sprint-plan/current/requirements.md` |
| architecture | `.omc/sprint-plan/current/architecture-decisions.md` |
| epics | `.omc/sprint-plan/current/epics.md` |
| stories | `.omc/sprint-plan/current/epics.md` (stories sections) |
| enrichment | All files in `.omc/sprint-plan/current/stories/` |
| retro | `.omc/sprint-plan/current/retrospective.md` |

### Scoped artifact loading

When `--epic=N` or `--story=N.M` is provided, load only the relevant portion of the artifact:

**`epics` phase with `--epic=N`**:
- Load `.omc/sprint-plan/current/epics.md`
- Extract only the `## Epic N: ...` section (from the `## Epic N` heading up to but not including the next `## Epic` heading or end of file)
- Pass this extracted section as the artifact content to the agents

**`stories` phase with `--epic=N`**:
- Load `.omc/sprint-plan/current/epics.md`
- Extract only the `## Epic N: ...` section (same extraction as above — story details are nested within epic sections)

**`enrichment` phase with `--epic=N`**:
- Load only story files matching `E{N}-S*.md` from `.omc/sprint-plan/current/stories/`
- Concatenate them with `---` separators between files, preserving filenames as headers

**`enrichment` phase with `--story=N.M`**:
- Load only `.omc/sprint-plan/current/stories/E{N}-S{M}.md`
- This is the sole artifact content

If the scoped artifact cannot be found (e.g., no `## Epic 3` heading exists, or `E2-S1.md` doesn't exist), respond:
```
Could not find {scope description} in the artifact. Check that the epic/story number is correct.
```

If the artifact file itself does not exist, respond:
```
Artifact file for phase "{phase}" not found at expected path. The phase may not have produced output yet.
```

---

## 5. Run RALPLAN-DR Consensus

Run three sequential agent passes. Each agent receives the artifact content (full or scoped) and the outputs from previous agents in the chain.

When a scope is active, include a scope line in each agent prompt so the agents understand they are reviewing a subset:
- `Scope: Epic {N}` or `Scope: Story {N.M}`
- If no scope: omit the scope line

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

Dispatch to `architect` (opus), passing the artifact content AND the planner's proposed improvements:

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

Dispatch to `critic` (opus), passing the artifact, planner improvements, AND architect challenges:

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

### If consensus is clear (critic produced APPLY or MODIFY verdicts):

Apply all APPLY and MODIFY changes to the artifact. Do not change sections that were not targeted by the consensus.

**Scoped write-back**: When a scope was used, write changes back to the correct location:
- **`--epic=N` on `epics` or `stories`**: Replace only the `## Epic N` section in `epics.md`, leaving all other epic sections untouched.
- **`--epic=N` on `enrichment`**: Write each modified story file back to its original path in `stories/`.
- **`--story=N.M`**: Write the modified content back to `stories/E{N}-S{M}.md`.
- **No scope**: Write the updated artifact back to the same path it was loaded from (unchanged behavior).

**Decision graph maintenance**: If the phase is `architecture` and changes were applied (decisions added, removed, or renamed), update `current/decision-graph.md` if it exists:
- If a decision was renamed: update the `## D-NNN: {title}` heading in the graph
- If a decision was removed: remove its section from the graph
- If a new decision was added: add a section with an empty story list (stories will be linked during the next `/epic-prep` or validation run)
- If the graph file doesn't exist, skip this step.

Similarly, if the phase is `enrichment` and the `decisions` frontmatter of a story changed (decision references added or removed), update the corresponding entries in `current/decision-graph.md` if it exists.

### If no consensus (critic found no APPLY or MODIFY changes):

Do not modify the artifact. Report to the user:
```
RAL pass complete. The critic found no changes to apply — the artifact is already at high quality for this iteration.
```

### If the agents produce irreconcilable disagreements on a specific point:

Present the disagreement to the user using the Decision Steering elicitation format:

```
## Decision Point: [Disagreement Title]

**What is being decided**: [1-sentence framing of the conflict]
**Why it matters**: [Impact on artifact quality or downstream phases]

### Planner's Position
[Summary of proposed change and rationale]

### Architect's Objection
[Summary of the challenge]

### Your Options
1. **Apply the change** — accept the planner's improvement
2. **Reject the change** — accept the architect's objection
3. **Propose a different resolution** — describe what you want
```

Wait for user input before proceeding on disagreements.

---

## 7. Update State

After the refinement pass completes (regardless of whether changes were applied):

1. Read `phase-state.json`.
2. Increment `ral_passes[key]` by 1 (using the same scoped key from section 3).
3. If the artifact WAS modified, mark downstream phases as stale in `stale_phases`. Scoped refinements use the same stale-marking rules as unscoped — changing one epic's design still potentially affects downstream phases.

**Downstream stale marking**:

| Phase Refined | Phases Marked Stale |
|--------------|-------------------|
| prd | None (PRD is upstream of sprint-plan, does not mark sprint phases stale) |
| requirements | architecture, epic-design, story-decomposition, story-enrichment, validation |
| architecture | epic-design, story-decomposition, story-enrichment, validation |
| epics | story-decomposition, story-enrichment, validation |
| stories | story-enrichment, validation |
| enrichment | validation |
| retro | None (retrospective is a terminal artifact) |

Write the updated `phase-state.json` back to `.omc/sprint-plan/current/phase-state.json`.

---

## 8. Report Results

After applying changes and updating state, report to the user:

```
--- RAL Pass Complete: {phase}{if scoped} ({scope_description}){/if} ---
Pass number: {ral_passes[key]} of 2 (use --force to exceed limit)

Changes applied: {count}
Changes rejected: {count}

Applied changes:
{list each applied/modified change with a 1-line description}

Rejected changes:
{list each rejected change with a 1-line reason}

{if artifact was modified}
Downstream phases marked stale: {list}
Re-run those phases or use --restart-from=<phase> to regenerate downstream artifacts.
{/if}

{if ral_passes[key] == 2}
Note: This {scope_description or phase} has now been refined 2 times. Further `ral {phase}` calls for this scope will require --force.
{/if}
```

If no changes were applied:
```
--- RAL Pass Complete: {phase}{if scoped} ({scope_description}){/if} ---
Pass number: {ral_passes[key]} of 2

Result: No changes applied. The artifact is at high quality for this iteration.
No downstream phases marked stale.
```
