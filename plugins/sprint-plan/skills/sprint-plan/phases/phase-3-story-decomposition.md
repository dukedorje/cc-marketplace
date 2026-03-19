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

### Step 1: Sequential Epic Loop

Process epics **in order** (Epic 1 first, then Epic 2, etc.). Later epics reference patterns and entities established by earlier ones. Do not parallelize across epics.

### Step 2: Per-Epic Decomposition

For each epic, run the following sequence:

#### 2a. Planner Story Decomposition

Dispatch one `planner` (opus) agent per epic:

```
Agent(
  subagent_type="oh-my-claudecode:planner",
  model="opus",
  prompt="""
You are decomposing Epic {epic_num}: {epic_title} into implementation-ready story stubs.

## Epic Details
{epic_content_from_epics_md}

## Architecture Decisions
{full_content_of_architecture_decisions_md}

## Requirements
{full_content_of_requirements_md}

## Rules
- Size each story for single dev agent completion (~1-4 hours of focused work)
- No forward dependencies within this epic — each story must be independently completable
- Database/entity creation only when actually needed by the story, not as a separate setup story
- Each story must reference which FRs it implements
- Use user story format: "As a [role], I want [action], so that [benefit]"
- Assign estimated complexity: small | medium | large

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
    "technical_notes": "Brief technical guidance from architecture decisions"
  }
]
"""
)
```

#### 2b. Parallel BDD Criteria Writing

For each story stub returned by the planner, fire a `writer` (sonnet) agent **in parallel** (all stories within an epic at the same time):

```
Agent(
  subagent_type="oh-my-claudecode:writer",
  model="sonnet",
  prompt="""
Write BDD acceptance criteria for this story.

## Story
Title: {story_title}
User Story: {user_story}
FRs: {frs}
Decisions: {decisions}
Technical Notes: {technical_notes}

## Architecture Decisions (for context)
{full_content_of_architecture_decisions_md}

## Requirements (for FR details)
{full_content_of_requirements_md}

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

#### 2c. Merge Story Stubs with BDD Criteria

After all parallel writer calls return for an epic, merge each story stub with its BDD output into the story stub format (see Output Schema below).

### Step 3: Epic Health Metrics

After processing all stories for an epic, compute and record health metrics:

| Metric | Check | Flag Condition |
|--------|-------|---------------|
| Story count | Count stories per epic | Flag if >8 (too large) or <2 (too small) |
| Complexity distribution | Count small/medium/large per epic | Flag if all large (epic may need splitting) |
| FR coverage | Verify every FR assigned to this epic has at least one story | Flag any uncovered FRs |
| Cross-epic dependencies | Scan story technical notes for references to future epic entities | Flag any detected forward deps |

### Step 4: Update epics.md

Append stories and health metrics directly to `current/epics.md` under each epic section.

---

## Output Schema

### Story stub format (written into epics.md)

```markdown
### Story {epic}.{story}: {Title}

**User Story**: As a [role], I want [action], so that [benefit].
**FRs**: [FR1, FR3]
**Decisions**: [D-001, D-003]
**Complexity**: small | medium | large

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

After this phase completes, update `current/phase-state.json`:
- `current_phase`: `"story-enrichment"`
- `stories_total`: total story count across all epics

---

## Agent Dispatch Reference

```
# Per-epic planner call (sequential)
Agent(subagent_type="oh-my-claudecode:planner", model="opus", prompt="...")

# Per-story writer calls (parallel within each epic)
Agent(subagent_type="oh-my-claudecode:writer", model="sonnet", prompt="...")
```

Fire all writer calls for an epic's stories simultaneously. Wait for all writers to return before merging and moving to the next epic.
