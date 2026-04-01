# Phase 1B: Sprint Scoping

Analyze the full requirement surface from Phase 1 and negotiate the sprint boundary with the user. Determines how much to bite off this sprint based on requirement complexity, velocity data (sprint N>1), and user intent.

This phase sits between Requirements (Phase 1) and Architecture (Phase 2A). Architecture decisions should only be made for in-scope requirements — designing systems you won't build this sprint wastes effort and produces stale decisions.

---

## Inputs

Read:
- `current/requirements.md` — all FRs, NFRs, TCs, scope boundaries, assumptions
- `current/discovery.md` — tech stack, project type, velocity data (sprint N>1)
- `.omc/backlog.md` — open backlog items (if exists)
- `.omc/backlog-promoted.md` — promoted items for this sprint (if exists)

## Sprint Size Parameter

The `sprint_size` value comes from:
1. `--sprint-size` flag (explicit user choice) — highest priority
2. Velocity-calibrated estimate (sprint N>1, from retro data)
3. Default: `standard`

| Sprint Size | Epics | Stories | Intent |
|------------|-------|---------|--------|
| `focused` | 1-2 | 3-8 | High uncertainty, new domain, learning sprint, maximum architectural steering |
| `standard` | 2-4 | 8-18 | Moderate confidence, known stack, balanced scope |
| `ambitious` | 4-6 | 15-30 | High confidence, well-understood domain, full feature delivery |

### Velocity Calibration (Sprint N>1)

When previous sprint data is available (from `discovery.md` → Previous Sprint Intelligence):

1. Extract from prior retro:
   - `stories_done` / `stories_total` → completion rate
   - Scope creep percentage (files outside original specs)
   - Replan count (mid-sprint course corrections)
   - Post-mortem root cause categories (if any)

2. Calculate **effective capacity**:
   ```
   raw_capacity = stories_done from last sprint
   divergence_factor = 1 - (scope_creep_pct / 100) - (replan_count * 0.05)
   effective_capacity = raw_capacity * max(divergence_factor, 0.5)
   ```

3. Map effective capacity to sprint size:
   - ≤8 stories → suggest `focused`
   - 9-18 stories → suggest `standard`
   - ≥19 stories → suggest `ambitious`

4. If `--sprint-size` was explicit, use it regardless — but note if it conflicts with velocity data:
   ```
   Note: --sprint-size=ambitious selected, but previous sprint velocity
   suggests standard capacity (12 stories done, 20% scope creep).
   Proceeding with ambitious — monitor closely.
   ```

---

## Analysis Agent

Dispatch to `analyst` (opus):

```
You are analyzing requirements to determine the right sprint scope.

## All Requirements
{requirements.md content — full FR/NFR/TC list}

## Project Context
{discovery.md summary — tech stack, project type, new_repo flag}

## Sprint Size Target
{sprint_size} ({stories_min}-{stories_max} stories, {epics_min}-{epics_max} epics)

{if velocity_data}
## Previous Sprint Velocity
- Stories: {done}/{total} ({completion_rate}%)
- Scope creep: {scope_creep_pct}%
- Replans: {replan_count}
- Effective capacity: {effective_capacity} stories
{/if}

{if backlog_promoted}
## Promoted Backlog Items (must include)
{promoted items list}
{/if}

{if backlog_open}
## Open Backlog ({count} items)
{summary by type and priority — for awareness}
{/if}

## Your Task

### Step 1: Cluster Requirements
Group FRs by natural cohesion and dependency. Each cluster is a proto-epic.
For each cluster:
- List the FRs it contains
- Estimate story count (based on FR complexity and AC count)
- Estimate complexity: low (CRUD/config), medium (integration/logic), high (new patterns/external deps)
- Note dependencies on other clusters

### Step 2: Assess Total Surface Area
- Total estimated stories across all clusters
- How does this compare to the sprint size target?
- Which clusters are foundation (must come first)?
- Which clusters are independent (can be deferred without blocking future work)?

### Step 3: Propose Sprint Boundary
Given the sprint size target, propose which clusters are:
- **IN**: Must be in this sprint (foundation + highest value)
- **STRETCH**: Could fit if IN clusters are clean (moderate risk)
- **DEFER**: Save for next sprint (independent or lower priority)

For each IN/STRETCH/DEFER classification, explain WHY:
- Foundation dependency (other clusters need this)
- User value density (high impact per story)
- Risk profile (new vs known patterns)
- Independence (can wait without blocking anything)

### Step 4: Identify Deferred FR Disposition
For each deferred cluster:
- Can it be built independently in a future sprint?
- Does deferring it create any risk (e.g., late integration)?
- Suggested sprint for pickup (N+1, N+2, or "when ready")

### Step 5: Flag Risks
- Any FR that spans multiple clusters (coupling risk)
- Any cluster where complexity estimate is uncertain
- Any assumption that could blow up scope if wrong

Return structured output:

```json
{
  "clusters": [
    {
      "id": "A",
      "name": "descriptive name",
      "frs": ["FR1", "FR2"],
      "estimated_stories": 5,
      "complexity": "low|medium|high",
      "dependencies": ["B"],
      "classification": "in|stretch|defer",
      "rationale": "why this classification"
    }
  ],
  "sprint_boundary": {
    "in_clusters": ["A", "B"],
    "stretch_clusters": ["C"],
    "defer_clusters": ["D"],
    "estimated_stories_in": 10,
    "estimated_stories_stretch": 14,
    "estimated_stories_total": 22
  },
  "promoted_backlog_mapped": [
    {
      "backlog_id": "BL-NNN",
      "mapped_to_cluster": "A",
      "rationale": "..."
    }
  ],
  "risks": [
    {
      "description": "...",
      "affected_clusters": ["A", "C"],
      "mitigation": "..."
    }
  ]
}
```
```

---

## User Negotiation

Present the scoping proposal as an interactive decision point:

```
═══════════════════════════════════════════════════
  Sprint Scoping: {total_frs} FRs across {cluster_count} clusters
═══════════════════════════════════════════════════

  Sprint size: {sprint_size}
  Target: {stories_min}-{stories_max} stories, {epics_min}-{epics_max} epics
  {if velocity_data}
  Previous sprint: {stories_done} stories done ({completion_rate}%),
    {scope_creep_pct}% scope creep
  {/if}

  ─────────────────────────────────────────────────

  IN (foundation + highest value):
    {cluster_id}. {name} — {fr_count} FRs, ~{story_count} stories [{complexity}]
       FRs: {fr_list}
       Why: {rationale}

    {cluster_id}. {name} — {fr_count} FRs, ~{story_count} stories [{complexity}]
       FRs: {fr_list}
       Why: {rationale}

  {if stretch_clusters}
  STRETCH (fits if core is clean):
    {cluster_id}. {name} — {fr_count} FRs, ~{story_count} stories [{complexity}]
       FRs: {fr_list}
       Why: {rationale}
  {/if}

  DEFER to next sprint:
    {cluster_id}. {name} — {fr_count} FRs, ~{story_count} stories [{complexity}]
       FRs: {fr_list}
       Why: {rationale}

  {if promoted_backlog}
  BACKLOG (promoted, included in scope):
    {for each promoted item}
    BL-{NNN}: {title} → mapped to cluster {id}
    {/for}
  {/if}

  ─────────────────────────────────────────────────

  Estimated scope:
    Core (IN only):     ~{in_stories} stories across {in_epics} clusters
    With stretch:       ~{stretch_stories} stories across {stretch_epics} clusters
    Full (everything):  ~{total_stories} stories across {total_clusters} clusters

  {if risks}
  Risks:
    {for each risk}
    - {description} (affects: {clusters})
    {/for}
  {/if}

  ─────────────────────────────────────────────────

  Options:
    continue       — accept IN clusters, defer the rest
    include C      — move cluster C from STRETCH/DEFER to IN
    exclude B      — move cluster B from IN to DEFER
    stretch        — include IN + STRETCH clusters
    all            — include everything (sprint size guardrails still apply)
    focused        — switch to focused sprint size, re-scope
    ambitious      — switch to ambitious sprint size, re-scope
═══════════════════════════════════════════════════
```

Wait for user input. If the user adjusts scope:
- Re-classify clusters based on their input
- If they change sprint size, re-run the analysis with the new target
- If they include/exclude specific clusters, adjust without re-running

---

## Output

Write to `SPRINT_DIR/sprint-scope.md`:

```markdown
---
sprint: sprint-{NNN}
sprint_size: {focused|standard|ambitious}
created: {ISO 8601}
total_frs: {count}
in_scope_frs: {count}
deferred_frs: {count}
estimated_stories: {count}
estimated_epics: {count}
{if velocity_data}
velocity_basis: sprint-{N-1}
velocity_stories_done: {count}
velocity_completion_rate: {pct}
effective_capacity: {count}
{/if}
---

# Sprint Scope

## Sprint Size
**{sprint_size}** — {rationale for this size choice}

{if velocity_data}
### Velocity Basis
Previous sprint: {stories_done}/{total} stories ({completion_rate}%), {scope_creep}% scope creep.
Effective capacity: ~{effective_capacity} stories.
{/if}

## In-Scope Clusters

{for each IN cluster}
### Cluster {id}: {name}
- **FRs**: {fr_list}
- **Estimated stories**: {count}
- **Complexity**: {level}
- **Rationale**: {why in scope}
{if dependencies}- **Depends on**: Cluster {deps}{/if}
{/for}

## Stretch Goals

{for each STRETCH cluster, or "None — scope is tight."}
### Cluster {id}: {name}
- **FRs**: {fr_list}
- **Estimated stories**: {count}
- **Include if**: {condition — e.g., "core clusters complete cleanly"}
{/for}

## Deferred to Future Sprints

{for each DEFER cluster}
### Cluster {id}: {name}
- **FRs**: {fr_list}
- **Estimated stories**: {count}
- **Suggested pickup**: Sprint {N+X}
- **Rationale**: {why deferred}
- **Risk of deferral**: {any integration risk, or "None — fully independent"}
{/for}

## Backlog Items Included

{for each promoted backlog item}
- **BL-{NNN}**: {title} → Cluster {id}
{/for}

## Scope Risks

{for each risk}
- {description} — affects {clusters}, mitigation: {mitigation}
{/for}

## FR Disposition Summary

| FR | Cluster | Status | Rationale |
|----|---------|--------|-----------|
| FR1 | A | IN | {brief} |
| FR7 | D | DEFER | {brief} |
```

### Deferred FRs → Backlog

For each deferred FR cluster, automatically add items to `.omc/backlog.md`:

```markdown
### BL-{NNN}: {cluster_name} — deferred from sprint {N}

- **Type**: deferred
- **Priority**: {medium if independent, high if blocking future work}
- **Status**: open
- **Added**: {timestamp}
- **Source**: scoping:sprint-{N}
- **Tags**: deferred, {cluster_id}

{cluster_description}. Contains FRs: {fr_list}.
Estimated {story_count} stories. Suggested pickup: sprint {N+X}.

> **Evidence**: Sprint scoping classified this cluster as DEFER — {rationale}
```

Make backlog capture graceful — skip if `.omc/backlog.md` doesn't exist.

---

## State Update

Update `phase-state.json`:
- Set `current_phase` to `"sprint-scoping"`
- Add `sprint_size: "{focused|standard|ambitious}"`
- Add `sprint_scope`:
  ```json
  {
    "in_scope_frs": ["FR1", "FR2", ...],
    "deferred_frs": ["FR7", "FR8", ...],
    "stretch_frs": ["FR5", ...],
    "estimated_stories": 12,
    "estimated_epics": 3,
    "clusters": [
      {"id": "A", "name": "...", "classification": "in", "frs": ["FR1", "FR2"]},
      {"id": "D", "name": "...", "classification": "defer", "frs": ["FR7"]}
    ]
  }
  ```

---

## Decision Steering

**Active — sprint scope is a CRITICAL decision.**

In GUIDED mode, the entire negotiation above IS the steering interaction.

In AUTONOMOUS mode (fast): auto-accept the analyst's proposed boundary without user negotiation. Log the scoping decision with `[AUTO-DECIDED]` marker.

---

## Inter-Phase Summary

Phase-specific content for section 3a:

| Field | Content |
|-------|---------|
| "What was produced" | Sprint scope: {in_count} FRs in scope (~{story_estimate} stories), {defer_count} FRs deferred, sprint size: {size} |
| "Key decisions made" | Which FR clusters are in vs deferred, sprint size selection |
| "Things to consider" | Stretch cluster viability, risks, velocity-vs-target alignment |
