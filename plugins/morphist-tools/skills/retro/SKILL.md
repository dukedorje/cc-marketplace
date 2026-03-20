---
name: retro
description: Generate a sprint retrospective from execution artifacts. Analyzes git history, story Dev Agent Records, and architecture decision adherence to produce a rich retrospective with cross-sprint intelligence for the next sprint's Phase 0.
user-invocable: true
argument-hint: "[--force]"
---

# retro: Sprint Retrospective Generator

You are the `retro` skill for the sprint-plan plugin. You analyze sprint execution artifacts and generate a structured retrospective that closes the learning loop and feeds the next sprint's Phase 0.

---

## 1. Precondition Check

Read `.omc/sprint-plan/current/phase-state.json`.

If the file does not exist, halt:
```
No active sprint found. Run /sprint-plan first to initialize a sprint.
```

Check `execution_status` in `phase-state.json`.

If `execution_status` is NOT `"complete"`:
- If `--force` is NOT in `$ARGUMENTS`, halt:
  ```
  Not all stories have been executed (execution_status: {current_value}).
  Run /sprint-exec to complete execution first.

  To generate a partial retrospective anyway, use: /retro --force
  ```
- If `--force` IS in `$ARGUMENTS`, continue with a note:
  ```
  Note: Generating partial retrospective — not all stories have been executed.
  ```

---

## 2. Load Sprint Context

Read the following files before dispatching analysis agents:

- `.omc/sprint-plan/current/phase-state.json` — sprint number, execution log
- `.omc/sprint-plan/current/discovery.md` — sprint start date (use `created` frontmatter field as git log date anchor), tech stack, project type
- `.omc/sprint-plan/current/requirements.md` — original requirements and scope
- `.omc/sprint-plan/current/architecture-decisions.md` — all architecture decisions
- `.omc/sprint-plan/current/epics.md` — epic and story structure

List all story files in `.omc/sprint-plan/current/stories/` to pass to analysis agents.

---

## 3. Parallel Data Collection

Dispatch three agents in parallel to analyze different aspects of the sprint:

### Agent 1: Git Log Analysis (explore, haiku)

```
You are analyzing git history for a completed sprint.

Sprint start date: {created_date_from_discovery_md_frontmatter}
Working directory: {working_directory}

Story files and their file lists:
{for each story: story_id, title, and File List from Dev Agent Record}

Tasks:
1. Run: git log --oneline --since="{created_date}" to get all commits since sprint start
2. For each commit, attempt to categorize it to a story by:
   - Matching story references in the commit message (e.g., "story 1.2", "1.2", "Epic 1")
   - File overlap: check which story's File List includes the files changed in the commit
3. Produce a summary with:
   - Total commits since sprint start
   - Commits per story (story_id: count)
   - Files changed count (total and per story if attributable)
   - Any commits that could not be attributed to a story
   - Commit velocity: commits per day (total_commits / sprint_days)
```

### Agent 2: Story Execution Analysis (analyst, opus)

```
You are analyzing sprint story execution results.

Story files directory: .omc/sprint-plan/current/stories/
Story file list: {list of all story file paths}

Tasks:
1. Read ALL story files listed above
2. For each story, extract from frontmatter: status, started_at, completed_at
3. For each story, extract from Dev Agent Record: Agent Model Used, Completion Notes, problems encountered, File List
4. Produce the following analysis:

   a. Story-by-story outcomes:
      - status (done/failed/blocked/in-progress)
      - duration (completed_at - started_at, if both present)
      - problems encountered (from Dev Agent Record)
      - files created vs files listed in story spec (scope creep indicators)

   b. Cross-story patterns:
      - Shared problems that appeared in multiple stories
      - Repeated solutions or patterns that emerged
      - Stories that completed cleanly with no problems
      - Stories that required significant problem-solving

   c. Scope creep analysis:
      - For each story, compare the File List in Dev Agent Record to the Files/file list in the story spec
      - Flag any files created that were NOT in the original story file list
      - Estimate impact (how many extra files, what kind)

   d. Technical debt indicators:
      - Note any "TODO", "FIXME", "workaround", "hack", or "temporary" language in Dev Agent completion notes
      - List each with: what it is, which story created it, which file(s), and severity estimate

   e. Velocity metrics:
      - Total stories planned vs completed vs failed vs blocked
      - Stories by complexity if complexity data is present in story frontmatter
```

### Agent 3: Architecture Decision Review (critic, opus)

```
You are reviewing how well architecture decisions held up during sprint execution.

Architecture decisions file: .omc/sprint-plan/current/architecture-decisions.md
Story files: {list of all story file paths}

Tasks:
1. Read the architecture decisions file — extract each decision with its ID (D-NNN), statement, and rationale
2. Read each story file — extract the Architecture Compliance section content and Dev Agent Record notes
3. For each architecture decision:
   - Did stories follow it? (cite specific story IDs)
   - Did any story deviate from it? (cite story ID and describe deviation)
   - Did execution reveal that the decision was wrong, incomplete, or needed revision?
   - Verdict: "held" | "partially held" | "revised" | "not applicable"
4. Identify decisions that need revision based on execution experience:
   - What happened that contradicted or required modifying the decision?
   - Proposed revision (brief description)
5. Produce:
   - Decisions that held: count and list
   - Decisions that needed revision: count and list with details
   - New patterns that emerged during execution that should become architecture decisions in the next sprint
```

Wait for all three agents to complete before proceeding.

---

## 4. Full-Sprint Reconciliation

Before generating the retrospective, run a full-sprint code style reconciliation. This detects cross-epic inconsistencies introduced by parallel agent execution.

Equivalent to dispatching `/reconcile --all`. This runs in **GUIDED mode** — contentious choices (high severity) are presented to the user as elicitations, similar to Decision Steering in sprint-plan.

After reconciliation completes, read the report at `.omc/sprint-plan/current/reviews/reconciliation-full-sprint.md` and pass it as additional input to the retrospective writer (section 5). The findings should appear in the retrospective's "Patterns Discovered" section.

If no inconsistencies are found, note "Code reconciliation: no cross-epic style issues detected" and proceed.

---

## 5. Generate Retrospective

Dispatch a `writer` (sonnet) agent with all collected analysis outputs to generate the retrospective document:

```
You are writing a sprint retrospective document.

Sprint: {sprint_number}
Sprint start: {created_date}

## Input Data

### Git Analysis
{git_analysis_output}

### Story Execution Analysis
{story_execution_analysis_output}

### Architecture Decision Review
{architecture_decision_review_output}

### Code Reconciliation
{reconciliation_report_from_section_4_or_"no cross-epic style issues detected"}

### Original Requirements Summary
{brief summary of requirements.md}

## Your Task

Write a comprehensive sprint retrospective to `.omc/sprint-plan/current/retrospective.md`.

The document must have YAML frontmatter:
---
sprint: {sprint_number}
generated_at: {ISO 8601 timestamp}
execution_status: {complete|partial}
stories_done: {count}
stories_failed: {count}
stories_blocked: {count}
---

Then include ALL of the following sections in this order:

### Executive Summary
3-5 sentences summarizing: what was built, how execution went, overall sprint health, and the primary learning.

### What Worked
Specific things that went well — cite story IDs. Include:
- Stories that completed cleanly
- Architecture decisions that proved sound
- Patterns or approaches that worked well
- Team/process observations if evident from commit patterns

### What Didn't Work
Specific things that went poorly — cite story IDs and decision IDs. Include:
- Stories that failed or had significant problems
- Architecture decisions that needed revision or caused friction
- Scope creep incidents
- Repeated problems across multiple stories

### Patterns Discovered
Two subsections:

**Coding Patterns**
- Patterns that emerged in implementation (e.g., "every story that touched auth required X")
- Reusable patterns that should be documented for future stories
- Anti-patterns that appeared (things done wrong that should be avoided)

**Process Patterns**
- Patterns in how stories were executed (e.g., "stories without clear file lists took longer")
- Observations about story quality and its impact on execution
- Observations about epic ordering (did sequential epics depend on each other as expected?)

### Technical Debt Created
For each piece of technical debt identified:
| Item | File(s) | Origin Story | Why Created | Priority |
|------|---------|-------------|-------------|---------|
List each debt item with suggested resolution priority: High/Medium/Low

### Architecture Decision Review
Table format:
| Decision ID | Statement (brief) | Held? | Notes |
|-------------|------------------|-------|-------|
Followed by a paragraph on decisions needing revision.

### Velocity Analysis
- Planned: {N} stories across {M} epics
- Completed (done): {count} ({percentage}%)
- Failed: {count} ({percentage}%)
- Blocked/Skipped: {count} ({percentage}%)
- Average story duration: {if timestamps available}
- Commits: {total}, averaging {per_day} per day
- Scope creep: {N} files created outside original story specs

### Learnings for Next Sprint
Concrete, actionable recommendations for the next sprint's Phase 0. Be specific — not "write better stories" but "stories that create new API endpoints should include the full request/response schema in Technical Requirements."

At least 3-5 recommendations.

### Next Epic Preparation
This section is critical — it makes the retrospective directly actionable for sprint N+1.

**Unfinished Work**
For each failed or blocked story:
- Story ID and title
- What was attempted
- What needs to happen before retrying
- Suggested approach for next sprint

**Deferred Requirements**
Review the Growth and Vision FRs from requirements.md that were not included in this sprint's scope. For each:
- Which deferred FRs are now ready to promote to MVP for the next sprint? (Criteria: dependencies from this sprint are now done)
- Suggested priority order for next sprint

**Architecture Evolution**
For each architecture decision that needed revision:
- D-NNN (revised): [original statement]
- Proposed revision: [updated statement]
- Status: proposed-revision

Also include: new patterns from execution that should become architecture decisions.

**Recommended Focus**
1-3 sentences: What should the next sprint's primary objective be, given what was learned?

**Technical Debt Priority**
Ordered list of debt items with the recommended sprint to address each:
1. [Item] — Sprint N+1 (High priority: {reason})
2. [Item] — Sprint N+2 (Medium priority: {reason})
...
```

Write the completed retrospective to `.omc/sprint-plan/current/retrospective.md`.

---

## 6. Update Phase State

After the retrospective is written:

1. Read `phase-state.json`
2. Add `retrospective_status: "complete"` as a sibling field
3. Write `phase-state.json` back

---

## 7. Report Completion

Print the completion summary:

```
--- Sprint {sprint_number} Retrospective Complete ---

Retrospective written to: .omc/sprint-plan/current/retrospective.md

Highlights:
  Stories done:    {count}/{total} ({percentage}%)
  Stories failed:  {count}
  Stories blocked: {count}
  Technical debt:  {count} items identified
  Arch decisions:  {held_count} held, {revised_count} needed revision

Key learnings: {first 1-2 learnings from the Learnings section}

Next sprint prep: {N} unfinished stories, {M} deferred requirements ready to promote

The retrospective is ready for Phase 0 of sprint {N+1}.
Run /sprint-plan to start the next sprint — it will automatically consume this retrospective.
```

---

## 8. Cross-Sprint Intelligence Note

The retrospective at `.omc/sprint-plan/current/retrospective.md` is automatically consumed by Phase 0 of the next sprint. When the next `/sprint-plan` run completes Phase 0 (discovery), it reads `sprint-{N-1}/retrospective.md` (after the current directory is archived) and uses it to:
- Pre-populate the requirements phase with unfinished work
- Carry forward architecture evolution proposals
- Inform velocity estimates for story sizing
- Surface promoted deferred requirements

No additional wiring is needed.

---

## 9. RAL Support

The `/ral retro` command targets this artifact at `.omc/sprint-plan/current/retrospective.md`. The `retro` phase is terminal — refining it does NOT mark any downstream phases stale.
