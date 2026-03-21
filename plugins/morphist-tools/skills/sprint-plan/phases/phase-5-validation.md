# Phase 5: Validation Gate

**Purpose**: Adversarial review of the complete artifact chain, including epic health metrics from Phase 3.

**Agents**:
- `critic` (opus): Full coverage validation
- `verifier` (sonnet): Mechanical correctness checks

**Input**: All artifacts from the current sprint directory:
- `current/requirements.md`
- `current/architecture-decisions.md`
- `current/epics.md`
- `current/stories/*.md` (all story files)
- `current/phase-state.json` (epic health metrics from Phase 3)

---

## Process

### Step 1: Fire Critic and Verifier in Parallel

Both agents receive the full artifact set simultaneously. Neither waits on the other.

```
critic (opus)  ──────────────────────────────────────────────────> findings-critic.json
verifier (sonnet) ───────────────────────────────────────────────> findings-verifier.json
                                                                         |
                                                                         v
                                                               merge -> unified-findings
```

**Agent dispatch**:
```python
# Fire both in parallel
critic_task = Agent(
    subagent_type="oh-my-claudecode:critic",
    model="opus",
    prompt=CRITIC_PROMPT  # see below
)
verifier_task = Agent(
    subagent_type="oh-my-claudecode:verifier",
    model="sonnet",
    prompt=VERIFIER_PROMPT  # see below
)
```

---

### Step 2: Critic Agent Validation (opus)

The critic performs adversarial, high-judgment validation. It is looking for gaps, not just surface errors.

#### 2a. FR Coverage Audit

- Load every FR from `requirements.md` (FR1, FR2, ..., FRN)
- Load the FR Coverage Map from `epics.md`
- For each FR, confirm at least one story exists that implements it
- **Orphan FR**: An FR that appears in requirements.md but is not covered by any story
- Output: FR coverage table with status per FR (Covered / Orphan)

#### 2b. Architecture Compliance Audit

- Load every decision from `architecture-decisions.md` (D-001, D-002, ...)
- For each decision, check whether at least one story's `## Architecture Compliance` section references it
- **Uncovered decision**: A decision with no story implementation coverage
- Note: Not every decision needs a story reference — only decisions with downstream implementation consequences. The critic uses judgment here.
- Output: Decision coverage table with status

#### 2c. Dependency Validation

- For each story, extract any dependencies it implies (e.g., "requires Story 1.2 to be complete first")
- Check: No story has a forward dependency within its own epic (e.g., Story 1.3 cannot depend on Story 1.4)
- Check: No story has a cross-epic forward dependency (e.g., a story in Epic 2 cannot depend on a story in Epic 3)
- **Forward dependency**: A story that requires a later story to be completed first
- Output: Dependency graph with any violations flagged

#### 2d. Story Quality Assessment

For each story file, verify all of the following:

| Check | Criterion |
|-------|-----------|
| Agent-completable | Story can be completed by a single dev agent without requiring human judgment mid-story |
| BDD acceptance criteria | At least 2 Given/When/Then acceptance criteria present and testable |
| Scope boundaries | Both "This story DOES" and "This story does NOT" sections present and non-empty |
| Technical requirements | Libraries/frameworks listed with versions (or "none required") |
| References | `frs` and `decisions` frontmatter fields populated |

#### 2e. Epic Health Assessment

Using story counts from `epics.md` and complexity data from `phase-state.json`:

| Flag | Condition | Advisory |
|------|-----------|---------|
| Too Large | Epic has >8 stories | Recommend splitting into two epics |
| Too Small | Epic has <2 stories | Recommend merging with adjacent epic |
| Cross-Epic Coupling | >30% of an epic's stories reference stories in another epic | Review dependency direction |
| Complexity Mismatch | High-complexity epic early in sequence | Review sequencing |

Epic health flags are **advisory only** — they appear in the readiness report but do not block a pass verdict unless the critic judges them severe.

---

### Step 3: Verifier Agent Checks (sonnet)

The verifier performs mechanical, deterministic correctness checks. No judgment required — these are binary pass/fail.

#### 3a. Frontmatter Consistency

For each story file:
- `frs` field lists FRs that are actually discussed in the story body (not phantom references)
- `decisions` field lists D-NNN codes that appear in the `## Architecture Compliance` section
- `epic` and `story` numbers match the filename

#### 3b. Section Completeness

Every story file must have all template sections present and non-placeholder:
- `## Story` — not "TODO" or empty
- `## Acceptance Criteria` — not "TODO" or empty
- `## Scope Boundaries` — not "TODO" or empty
- `## Tasks / Subtasks` — not "TODO" or empty
- `## Architecture Compliance` — not "TODO" or empty (may state "No specific decisions apply" explicitly)
- `## Technical Requirements` — not "TODO" or empty
- `## Dev Agent Record` — may be empty (not yet implemented), but section header must be present

#### 3c. File Naming

Every story file must match pattern: `{epic_num}-{story_num}-{slug}.md`

Examples of valid names: `1-1-user-auth.md`, `2-3-dashboard-layout.md`

Examples of invalid names: `story-1.md`, `epic1-story2.md`, `1.1-auth.md`

#### 3d. FR Coverage Map Consistency

The FR Coverage Map table in `epics.md` must match actual story file existence:
- Every story listed in the map must have a corresponding file in `stories/`
- Every story file in `stories/` must appear in the map

#### 3e. Decision Reference Integrity

Every D-NNN reference in any story file must point to a real decision in `architecture-decisions.md`.

No dangling references (e.g., story references D-007 but no D-007 exists).

#### 3f. Story Number Uniqueness

Within each epic, no two stories share the same story number.

Valid: `1-1-auth.md`, `1-2-profile.md`
Invalid: `1-1-auth.md`, `1-1-login.md`

#### 3g. Map-File Consistency

Every story referenced in `epics.md` must have a corresponding file in `stories/`. Every file in `stories/` must be referenced in `epics.md`. No orphaned files, no missing files.

---

### Step 4: Merge Findings

After both agents complete, the orchestrator merges findings into a single unified list:

```python
findings = {
    "critical": [],    # Blocks readiness verdict
    "warnings": [],    # Advisory, does not block
    "auto_fixable": [] # Can be corrected without user input
}
```

**Classification rules**:

| Finding Type | Classification |
|-------------|---------------|
| Orphan FR (no story coverage) | Critical |
| Architecture decision with no implementation reference | Warning (unless decision is CRITICAL significance → Critical) |
| Forward dependency violation | Critical |
| Story missing BDD criteria | Critical |
| Story missing scope boundaries | Warning |
| Story missing technical requirements | Warning |
| Broken D-NNN reference | Critical |
| Frontmatter mismatch (frs/decisions) | Auto-fixable |
| Placeholder text in story section | Auto-fixable if minor, Warning if major |
| File naming violation | Auto-fixable |
| Map-file mismatch | Critical |
| Duplicate story number | Critical |
| Epic health flag (too large/small) | Warning |

---

### Step 5: Auto-Fix Loop

**Trigger**: One or more `auto_fixable` findings exist.

**Max iterations**: 2. After 2 iterations, remaining auto-fixable issues are reclassified as warnings and presented to the user.

**Auto-fixable actions**:
- Correct frontmatter `frs` and `decisions` fields to match story body content
- Fix file naming violations (rename files to match pattern)
- Fill placeholder text in minor sections (e.g., add "No specific decisions apply" to Architecture Compliance when no D-NNN refs exist in story body)
- Correct Map-file references after file renames

**Not auto-fixable** (requires user):
- Missing story for an orphan FR (scope decision — must a new story be created, or is the FR out of scope?)
- Forward dependency violations (requires restructuring the epic or story sequence)
- Missing BDD acceptance criteria (requires domain knowledge to write correctly)
- Architecture decisions with no story coverage (may indicate an architectural gap)

After auto-fixes are applied, the verifier re-runs its mechanical checks to confirm fixes are clean.

---

### Step 6: Final Verdict and Readiness Report

**If critical findings remain after auto-fix loop**:
- Status: `fail`
- Present unresolved critical issues to user with actionable guidance
- User must address them and re-run Phase 5 (`ral validation` not applicable here — re-run the phase)

**If only warnings remain (no criticals)**:
- Status: `pass-with-warnings`
- Present readiness report with warnings highlighted

**If no findings remain**:
- Status: `pass`
- Present readiness report

**Decision Steering**: Dormant. The final readiness report is ALWAYS shown to the user regardless of steering mode (GUIDED or AUTONOMOUS). This is the final user checkpoint before implementation begins.

---

## Output: `current/readiness-report.md`

```markdown
---
sprint: sprint-[NNN]
validation_status: pass | pass-with-warnings | fail
timestamp: [datetime]
---

# Sprint Readiness Report

## Summary
- **Status**: [READY / READY WITH WARNINGS / NOT READY]
- **Epics**: [count]
- **Stories**: [count]
- **Total FRs**: [count]
- **FR Coverage**: [percentage]%
- **Architecture Compliance**: [percentage]%

## FR Coverage Audit
| FR | Description | Epic | Story | Status |
|----|-------------|------|-------|--------|
| FR1 | [brief description] | Epic 1 | Story 1.1 | Covered |
| FR2 | [brief description] | Epic 1 | Story 1.2 | Covered |
| FR3 | [brief description] | - | - | ORPHAN — no story coverage |

## Architecture Decision Audit
| Decision | Significance | Stories Implementing | Status |
|----------|-------------|---------------------|--------|
| D-001 | HIGH | Story 1.1, Story 2.1 | Covered |
| D-002 | CRITICAL | Story 1.2 | Covered |
| D-003 | MEDIUM | (none) | Not referenced — verify intentional |

## Epic Health
| Epic | Stories | Complexity | Flags |
|------|---------|------------|-------|
| Epic 1: [Title] | 4 | Medium | None |
| Epic 2: [Title] | 9 | High | TOO LARGE — recommend splitting |
| Epic 3: [Title] | 1 | Low | TOO SMALL — recommend merging |

## Dependency Graph
[Textual map of epic ordering and cross-epic dependencies]

Epic 1 (Foundation)
  └─> Epic 2 (Core Features) [depends on: 1.2, 1.3]
        └─> Epic 3 (Advanced Features) [depends on: 2.1]

No circular dependencies detected.

## Issues Found

### Critical (blocks readiness)
[List or "None"]

### Warnings (advisory)
[List or "None"]

### Auto-Fixed
[List of issues that were automatically corrected, or "None"]

## Recommendations
[Any suggestions for improvement before starting implementation. Omit section if none.]

## Next Steps
- [ ] Review this report
- [ ] Address any critical issues listed above
- [ ] For epic health warnings, consider `ral epics` to restructure if warranted
- [ ] Begin implementation with `/team ralph` or equivalent
```

---

## Critic Prompt Template

```
You are performing adversarial validation of a sprint planning artifact chain.

Your role is to find REAL problems that would cause a developer agent to fail or produce incorrect output. Do not invent problems. Do not flag style preferences. Flag only substantive issues.

## Artifacts Provided
- requirements.md: [contents]
- architecture-decisions.md: [contents]
- epics.md: [contents]
- stories/: [all story file contents]

## Your Tasks

1. FR COVERAGE: List every FR from requirements.md. For each, identify whether a story implements it. Output a table.

2. ARCHITECTURE COMPLIANCE: List every decision from architecture-decisions.md. For each, assess whether any story's Architecture Compliance section reflects it. Decisions with no implementation consequences (e.g., purely naming decisions) may be marked "Not applicable." Use judgment.

3. DEPENDENCY VALIDATION: For each story, identify any implied prerequisites. Flag any forward dependencies (story requires a later story).

4. STORY QUALITY: For each story, check: (a) completable by single agent, (b) has BDD criteria, (c) has scope boundaries, (d) has technical requirements. Flag failures.

5. EPIC HEALTH: Count stories per epic. Flag epics with >8 stories (too large) or <2 stories (too small). Flag any cross-epic coupling concerns.

## Output Format
Return a JSON object:
{
  "fr_coverage": [...],
  "architecture_compliance": [...],
  "dependency_violations": [...],
  "story_quality_issues": [...],
  "epic_health_flags": [...],
  "summary": {
    "critical_count": N,
    "warning_count": N
  }
}
```

## Verifier Prompt Template

```
You are performing mechanical correctness checks on sprint planning artifacts. These are binary pass/fail checks — no judgment required.

## Artifacts Provided
- requirements.md: [contents]
- architecture-decisions.md: [contents]
- epics.md: [contents]
- stories/: [all story file contents with filenames]

## Your Tasks

For each story file, check ALL of the following:

1. FRONTMATTER: Does `frs` match FRs discussed in body? Does `decisions` match D-NNN refs in Architecture Compliance? Do epic/story numbers match filename?

2. COMPLETENESS: Are all template sections present? Are any sections placeholder text ("TODO", empty string, "TBD")?

3. FILE NAMING: Does filename match `{epic_num}-{story_num}-{slug}.md`?

4. FR MAP CONSISTENCY: Does every story in epics.md FR Coverage Map have a file? Does every story file appear in the map?

5. DECISION REFERENCES: Does every D-NNN ref in any story point to a real decision in architecture-decisions.md?

6. DUPLICATE STORY NUMBERS: Are story numbers unique within each epic?

7. MAP-FILE CONSISTENCY: Are all story files in stories/ referenced in epics.md? Are all stories in epics.md present as files?

## Output Format
Return a JSON object:
{
  "frontmatter_issues": [...],
  "completeness_issues": [...],
  "naming_violations": [...],
  "fr_map_inconsistencies": [...],
  "broken_decision_refs": [...],
  "duplicate_story_numbers": [...],
  "map_file_mismatches": [...]
}
```

---

## Decision Graph Generation

After a passing verdict (before state updates), build the decision graph at `current/decision-graph.md`.

This is the initial seed — `/epic-prep` and `/replan` will maintain it going forward.

### Process

1. Read all story files from `current/stories/`
2. Read all decisions from `current/architecture-decisions.md`
3. For each decision (D-NNN):
   - Find all stories whose `decisions` frontmatter field references this ID
   - Also scan story body `## Architecture Compliance` sections for the decision ID
4. Write the graph:

```markdown
# Decision Graph

Maps architecture decisions to dependent stories. Updated by /sprint-plan, /epic-prep, /replan.

## D-001: {decision title}
- 1.1: {story title}
- 1.3: {story title}
- 2.2: {story title}

## D-003: {decision title}
- 2.1: {story title}
- 2.3: {story title}
```

Decisions with no story references are included with an empty list and a note: `(no stories reference this decision)`.

Skip this step if the verdict is `fail` (the sprint needs fixes first).

---

## State Updates

On phase completion, update `current/phase-state.json`:

```json
{
  "current_phase": "validation",
  "validation_status": "pass | pass-with-warnings | fail",
  "validation_iterations": 1
}
```

If status is `pass` or `pass-with-warnings`, the sprint is ready for implementation.
