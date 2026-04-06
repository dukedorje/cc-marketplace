# Phase 3: Story Decomposition

**Purpose**: Break each epic into implementation-ready stories with BDD acceptance criteria.

**Decision Steering**: Dormant. All significant decisions were made in Phases 1-2.

---

## Agents

- `planner` (opus): Decomposes each epic into stories (sequential per epic)
- `writer` (sonnet) x N: Parallel BDD acceptance criteria writing within each epic

---

## Input

Load from `current/` at phase start (context shedding — load artifacts only, not prior agent memory):

- `current/epics.md` — epic list and FR assignments
- `current/architecture-decisions.md` — decisions that constrain story scope
- `current/requirements.md` — FR/NFR reference for coverage validation

---

## Process

### Step 1: Parallel Epic Decomposition (fan-out)

Process all epics **in parallel**. Each planner agent receives the **full** `epics.md` (all epics, not just its own) so it can see scope boundaries and avoid overlap. Architecture decisions and requirements are also shared.

Dispatch one `planner` (opus) agent per epic, all simultaneously:

```
# For each epic (ALL in parallel):
Agent(
  subagent_type="oh-my-claudecode:planner",
  model="opus",
  run_in_background=True,
  prompt="""
You are decomposing Epic {epic_num}: {epic_title} into implementation-ready story stubs.

## YOUR Epic Details
{epic_content_from_epics_md_for_this_epic}

## ALL Epics (for context — understand scope boundaries, avoid overlap)
Read the full epic plan from: SPEC_DIR/epics.md

## Architecture Decisions
Read the full architecture decisions from: SPEC_DIR/architecture-decisions.md

## Requirements
Read the full requirements from: SPEC_DIR/requirements.md

## Rules
- Size each story for single dev agent completion (~1-4 hours of focused work)
- No forward dependencies within this epic — each story must be independently completable
- Database/entity creation only when actually needed by the story, not as a separate setup story
- Each story must reference which FRs it implements
- Use user story format: "As a [role], I want [action], so that [benefit]"
- Assign estimated complexity: small | medium | large
- Be aware of what OTHER epics cover — don't duplicate their scope into your stories
- If your epic depends on entities/patterns from another epic, reference them by name but don't recreate them
- Assign a test_tier to each story based on risk and complexity:
  - "yolo": Small, clear, low-risk, UI tweaks, config changes. Build check only — if it compiles/runs, ship it.
  - "smoke": Medium complexity, standard CRUD, familiar patterns. Happy path tests only. Default tier.
  - "thorough": Large, novel, security-sensitive, data integrity, complex state. Full TDD with edge cases and error handling.
- Default to "smoke" unless there's a clear reason to go higher or lower.
- Stories touching auth, payments, data migration, or security → always "thorough"
- Stories that are pure UI layout, config, or documentation → "yolo"

## Output Format
Return a JSON array of story stubs:
[
  {
    "story_num": 1,
    "title": "...",
    "user_story": "As a [role], I want [action], so that [benefit].",
    "frs": ["FR1", "FR3"],
    "decisions": ["D-001", "D-003"],
    "complexity": "small|medium|large",
    "test_tier": "yolo|smoke|thorough",
    "technical_notes": "Brief technical guidance from architecture decisions"
  }
]
"""
)
```

### Step 2: Parallel BDD Criteria Writing

After all planner agents return, fire BDD writers for **all stories across all epics** in parallel:

```
# For each story across ALL epics (parallel):
Agent(
  subagent_type="oh-my-claudecode:writer",
  model="sonnet",
  run_in_background=True,
  prompt="""
Write BDD acceptance criteria for this story.

## Story
Title: {story_title}
User Story: {user_story}
FRs: {frs}
Decisions: {decisions}
Technical Notes: {technical_notes}

## Architecture Decisions (for context)
Read from: SPEC_DIR/architecture-decisions.md

## Requirements (for FR details)
Read from: SPEC_DIR/requirements.md

## Rules
- Use Given/When/Then format exactly
- One BDD scenario per acceptance criterion
- Cover: happy path, edge cases, error cases
- Reference specific FRs and architecture decisions by ID
- Include technical constraints from architecture-decisions.md
- Minimum 3 ACs per story (happy path + at least one edge case + at least one error case)

## Output Format
Return a JSON array of acceptance criteria:
[
  {
    "ac_num": 1,
    "title": "Happy path: [brief title]",
    "given": "...",
    "when": "...",
    "then": "...",
    "type": "happy|edge|error"
  }
]
"""
)
```

### Step 3: Merge & Reconciliation

#### 3a. Merge Story Stubs with BDD Criteria

After all writer agents return, merge each story stub with its BDD output into the story stub format (see Output Schema below).

#### 3b. Cross-Epic Reconciliation Scan

Dispatch a verifier agent (sonnet) to scan all story stubs across all epics for issues introduced by parallel decomposition:

```
Agent(
  subagent_type="oh-my-claudecode:verifier",
  model="sonnet",
  prompt="""
Scan all story stubs for cross-epic consistency issues.

## All Story Stubs
{all_story_stubs_across_all_epics}

## Epic Plan
Read from: SPEC_DIR/epics.md

## Requirements
Read from: SPEC_DIR/requirements.md

Check for:
1. **Duplicate stories**: Two stories across different epics covering the same FR or building the same feature
2. **FR coverage gaps**: Any FR assigned to an epic that has no story implementing it
3. **Cross-epic dependency conflicts**: Story in Epic N assumes an entity/pattern from Epic M that Epic M doesn't actually create
4. **Scope boundary violations**: Story scope overlaps with another epic's declared scope
5. **Entity naming conflicts**: Same concept named differently across epics

## Output Format
{
  "status": "clean|warnings|issues",
  "findings": [
    {
      "type": "duplicate|gap|dependency|overlap|naming",
      "severity": "info|warning|error",
      "description": "...",
      "affected_stories": ["1.2", "3.1"],
      "suggested_fix": "..."
    }
  ]
}
"""
)
```

#### 3c. Auto-Fix Reconciliation Issues

- **info**: Log only, no action
- **warning**: Apply suggested fix if unambiguous (e.g., rename for consistency), log the change
- **error**: Apply fix if possible. If not auto-fixable, flag to the orchestrator for the inter-phase summary.

### Step 4: Epic Health Metrics

After processing all stories for all epics, compute and record health metrics:

| Metric | Check | Flag Condition |
|--------|-------|---------------|
| Story count | Count stories per epic | Flag if >8 (too large) or <2 (too small) |
| Complexity distribution | Count small/medium/large per epic | Flag if all large (epic may need splitting) |
| FR coverage | Verify every FR assigned to this epic has at least one story | Flag any uncovered FRs |
| Cross-epic dependencies | Scan story technical notes for references to future epic entities | Flag any detected forward deps |
| Test tier distribution | Count yolo/smoke/thorough per epic | Flag if >50% thorough (may be over-testing) |

### Step 5: Update epics.md

Append stories and health metrics directly to `SPEC_DIR/epics.md` under each epic section.

---

## Output Schema

### Story stub format (written into epics.md)

```markdown
### Story {epic}.{story}: {Title}

**User Story**: As a [role], I want [action], so that [benefit].
**FRs**: [FR1, FR3]
**Decisions**: [D-001, D-003]
**Complexity**: small | medium | large
**Test Tier**: yolo | smoke | thorough

#### Acceptance Criteria
- **AC1**: Given [precondition], When [action], Then [expected result]
- **AC2**: Given [precondition], When [action], Then [expected result]
- **AC3**: [error case] Given [precondition], When [invalid action], Then [error handling]

#### Technical Notes
[Brief technical guidance from architecture decisions]

#### Scope Boundaries
- This story DOES: [explicit list]
- This story does NOT: [explicit list]
```

### Epic health metrics format (appended to each epic in epics.md)

```markdown
## Epic {N} Health Metrics
- **Story count**: {N} {flag if >8 or <2}
- **Complexity**: {X small, Y medium, Z large} {flag if all large}
- **FR coverage**: {all FRs covered | FR5, FR7 uncovered — flag}
- **Dependency flags**: {none | "Story 2.3 references Epic 3 entity UserProfile — forward dep flag"}
```

---

## State Updates

After this phase completes, update `STATE_DIR/phase-state.json`:
- `current_phase`: `"story-enrichment"`
- `stories_total`: total story count across all epics

---

## Agent Dispatch Reference

```
# Step 1: All epics in parallel
Agent(subagent_type="oh-my-claudecode:planner", model="opus", prompt="...")  # per-epic, parallel

# Step 2: All stories across all epics in parallel (after all planners return)
Agent(subagent_type="oh-my-claudecode:writer", model="sonnet", prompt="...")  # per-story, parallel

# Step 3b: Cross-epic reconciliation (after all writers return)
Agent(subagent_type="oh-my-claudecode:verifier", model="sonnet", prompt="...")
```

Fire all planner calls simultaneously. After all return, fire all writer calls across all epics simultaneously. After all return, run reconciliation scan.
