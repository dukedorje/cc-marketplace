---
name: post-mortem
description: Incident post-mortem for a story or epic — analyzes what went wrong, documents root cause and lessons learned directly in story docs so future agents learn from past failures.
user-invocable: true
argument-hint: "[--story=N.M] [--epic=N] [--incident=\"...\"] [--dry-run]"
---

# post-mortem: Story-Level Incident Documentation

Analyzes a failure or significant incident in a story or epic, determines root cause, and writes the findings **directly into the story docs**. Unlike `/retro` (which produces a sprint-wide retrospective), `/post-mortem` enriches the specific story documentation so that future agents reading those docs inherit the hard-won lessons.

---

## 1. Argument Parsing

Parse `$ARGUMENTS`:

| Flag | Required | Description |
|------|----------|-------------|
| `--story=N.M` | No | Post-mortem for a specific story |
| `--epic=N` | No | Post-mortem for all failed/problematic stories in an epic |
| `--incident="..."` | No | Free-text description of what happened (supplements story data) |
| `--dry-run` | No | Show findings without modifying story files |

At least one of `--story` or `--epic` is required.

If neither is provided:
```
Usage: /post-mortem --story=N.M [--incident="what happened"]
       /post-mortem --epic=N

Analyzes failures and writes lessons directly into story docs for future agents.

Examples:
  /post-mortem --story=2.3                              # Analyze a failed story
  /post-mortem --story=2.3 --incident="SSE broke in prod despite passing tests"
  /post-mortem --epic=3                                 # All problematic stories in Epic 3
  /post-mortem --epic=3 --dry-run                       # Preview without modifying files
```

---

## 2. Gather Context

### 2a. Read Sprint Artifacts

Read:
- `.omc/sprint-plan/current/phase-state.json`
- `.omc/sprint-plan/current/architecture-decisions.md`
- `.omc/sprint-plan/current/epics.md`
- `.omc/sprint-plan/current/requirements.md`
- `.omc/sprint-plan/current/decision-graph.md` (if exists)
- `.omc/sprint-plan/current/replan-log.md` (if exists)
- `.omc/sprint-plan/current/work-log.md` (if exists)

### 2b. Identify Stories in Scope

**If `--story=N.M`**: Read that single story file from `current/stories/`.

**If `--epic=N`**: Read all story files in the epic. Select stories that are candidates for post-mortem — any story with:
- `status: failed`
- `status: blocked`
- `blocker_type` set in frontmatter
- `audit_verdict` of `INCOMPLETE`, `NEEDS_REWORK`, or `NOT_STARTED`
- `replan_reason` set (was reset due to mid-sprint replan)

If no problematic stories found in the epic:
```
No failed, blocked, or replanned stories found in Epic {N}.
All stories appear healthy. Nothing to post-mortem.

To force a post-mortem on a specific story: /post-mortem --story=N.M
```

### 2c. Collect Implementation Evidence

For each story in scope, gather:
1. The full story file (spec + Dev Agent Record)
2. Files listed in the Dev Agent Record — read each to check current state
3. Git log for files in the story's file list: `git log --oneline --follow -- {file}` for each
4. Any audit report sections already in the story
5. Replan notes if present
6. Work log entries referencing this story

---

## 3. Root Cause Analysis

For each story in scope, dispatch an analyst agent to determine root cause.

Process stories in parallel (up to 3 concurrent agents).

```python
Agent(
    subagent_type="oh-my-claudecode:analyst",
    model="opus",
    prompt="""
You are performing a root cause analysis on a story that experienced a failure or significant incident.

## Story Specification
{story_file_content}

## Architecture Decisions (relevant to this story)
{relevant_decisions_text}

## Implementation Evidence
{file_contents_from_dev_agent_record_file_list}

## Git History for Story Files
{git_log_output_for_story_files}

{if replan_log_entries}
## Replan Events Affecting This Story
{replan_entries}
{/if}

{if work_log_entries}
## Work Log Entries
{work_log_entries_for_this_story}
{/if}

{if incident_description}
## Incident Description (from user)
{incident_description}
{/if}

## Analysis Task

Perform a thorough root cause analysis. Work through the following:

### Step 1: Establish the Timeline
Reconstruct what happened in order:
- When was the story started?
- What was implemented (from Dev Agent Record)?
- When/how did the failure manifest?
- Was there a replan that affected this story?
- What was the final state?

### Step 2: Identify Root Cause
Classify the root cause into one or more categories:
- **Spec gap**: Story spec was missing critical information or acceptance criteria
- **Wrong assumption**: An architecture decision or technical assumption proved incorrect
- **Dependency failure**: A dependency (library, API, other story) didn't work as expected
- **Scope underestimate**: Story was larger/more complex than anticipated
- **Environment issue**: Build, tooling, or environment problem
- **Implementation error**: The agent made an avoidable mistake
- **External change**: Something outside the sprint changed (API, requirements, etc.)

For each root cause:
- What specifically went wrong
- What was the incorrect assumption or missing information
- When could this have been caught earlier

### Step 3: Assess Impact
- What work was wasted or needs redoing?
- Were other stories affected?
- Was there a cascade (one failure causing others)?

### Step 4: Extract Lessons
For each root cause, derive a concrete lesson:
- **What to check**: A specific verification step that would catch this
- **What to specify**: Information that should be in future story specs
- **What to avoid**: Anti-pattern or approach that doesn't work here
- **What to prefer**: Better approach that should be used instead

Lessons must be specific and actionable — not "write better specs" but
"stories involving SSE must specify the exact event format and include
a test for connection drop/reconnect."

### Step 5: Recommend Preventive Measures
- Should an architecture decision be revised?
- Should the story spec template be updated?
- Are there guard rails (tests, checks) that should exist?

Return structured output:

```json
{
  "story": "N.M",
  "title": "...",
  "timeline": [
    {"when": "...", "what": "..."}
  ],
  "root_causes": [
    {
      "category": "spec_gap|wrong_assumption|dependency_failure|scope_underestimate|environment_issue|implementation_error|external_change",
      "description": "what went wrong",
      "incorrect_assumption": "what was believed vs reality",
      "earliest_catchable": "when/how this could have been caught",
      "severity": "critical|major|minor"
    }
  ],
  "impact": {
    "wasted_work": "description",
    "other_stories_affected": ["N.M", ...],
    "cascade": true|false,
    "cascade_description": "..."
  },
  "lessons": [
    {
      "type": "check|specify|avoid|prefer",
      "lesson": "concrete actionable lesson",
      "applies_to": "this story|stories like this|all stories|specific pattern"
    }
  ],
  "preventive_measures": [
    {
      "type": "revise_decision|update_template|add_test|add_check|process_change",
      "description": "what to do",
      "target": "D-NNN|story template|CI|etc."
    }
  ]
}
```
""",
)
```

---

## 4. Compile Findings

### 4a. Aggregate Results

Collect results from all analyst agents and build a summary:

```
==================================================
  POST-MORTEM ANALYSIS
==================================================

  Stories analyzed: {count}

  Root causes found:
    Spec gap:           {count}
    Wrong assumption:   {count}
    Dependency failure: {count}
    Scope underestimate:{count}
    Environment issue:  {count}
    Implementation error:{count}
    External change:    {count}

  ---

  {for each story}
  Story {N.M}: {title}
    Status: {status}
    Root causes: {list categories}
    Key lesson: {most important lesson, one line}
  {/for}
==================================================
```

### 4b. Detect Cross-Story Patterns

If multiple stories share the same root cause category or the same incorrect assumption, flag it as a systemic issue:

```
!! SYSTEMIC ISSUE: {count} stories failed due to {shared root cause}.
   This suggests a planning-level gap, not a story-level problem.
   Consider: /replan --decision=D-{NNN} --reason="{reason}"
```

If `--dry-run`, stop here.

---

## 5. Update Story Documentation

This is the core value of post-mortem vs retro — lessons are written **into the story docs** where future agents will read them.

### 5a. Write Post-Mortem Section

For each analyzed story, insert a `## Post-Mortem` section. Place it **after the Dev Agent Record** (at the end of the file) so it's the last thing an agent reads — the final word on what happened.

```markdown
## Post-Mortem

**Date**: {ISO 8601 timestamp}
**Incident**: {one-line summary of what went wrong}
**Severity**: {critical|major|minor}

### Timeline
{for each timeline entry}
- **{when}**: {what}
{/for}

### Root Causes
{for each root cause}
#### {category_human_readable}
{description}

- **Assumed**: {what was believed}
- **Reality**: {what actually happened}
- **Could have caught**: {earliest_catchable}
{/for}

### Impact
- **Wasted work**: {description}
{if other_stories_affected}- **Cascaded to**: {story list}{/if}

### Lessons for Future Agents

> **READ THIS** before re-implementing or working on similar stories.

{for each lesson}
- **{type}**: {lesson}
{/for}

### Preventive Measures
{for each measure}
- [{type}] {description}{if target} (target: {target}){/if}
{/for}
```

### 5b. Update Story Frontmatter

Add post-mortem metadata to the story frontmatter:

1. Add `post_mortem: true`
2. Add `post_mortem_date: {ISO 8601}`
3. Add `root_causes: ["{category1}", "{category2}"]`
4. Preserve all existing frontmatter fields

### 5c. Annotate Acceptance Criteria (if audit data exists)

If the story has an existing `## Audit Report` section, cross-reference:
- For each AC that was NOT MET or PARTIAL, add a note linking it to the relevant root cause
- This creates a direct connection: "AC failed because of root cause X"

---

## 6. Write Epic-Level Summary (if `--epic`)

When analyzing multiple stories in an epic, create or update an epic-level post-mortem summary at `.omc/sprint-plan/current/post-mortems/epic-{N}.md`:

```markdown
---
epic: {N}
title: {epic_title}
generated_at: {ISO 8601}
stories_analyzed: {count}
---

# Epic {N} Post-Mortem: {epic_title}

## Summary
{2-3 sentences: what the epic was supposed to deliver, what went wrong, overall assessment}

## Stories Analyzed
| Story | Status | Root Causes | Key Lesson |
|-------|--------|-------------|------------|
| {N.M} | {status} | {categories} | {primary lesson} |

## Systemic Issues
{cross-story patterns from section 4b}

## Lessons Learned
{deduplicated, prioritized list of all lessons across stories}

### Must-Check Before Re-execution
{lessons of type "check" — these are pre-flight validations}

### Spec Improvements
{lessons of type "specify" — these improve story quality}

### Anti-Patterns
{lessons of type "avoid" — approaches that failed}

### Preferred Approaches
{lessons of type "prefer" — what works instead}

## Recommended Actions
{prioritized list of preventive measures}
```

Create the `post-mortems/` directory if it doesn't exist.

---

## 7. Cross-Reference and Log

### 7a. Update Work Log

Append to the sprint work log (`.omc/sprint-plan/current/work-log.md`):

```markdown
---

### {ISO 8601 timestamp}

Post-mortem completed for {story N.M | Epic N}

**Root causes**: {comma-separated categories}
**Key lesson**: {most impactful lesson}
{if epic}**Epic {N}**: {epic_title}{/if}
{if story}**Story {N.M}**: {story_title}{/if}
**Tags**: post-mortem
```

### 7b. Update Phase State

Append to `execution_log` in `phase-state.json`:

```json
{
  "type": "post-mortem",
  "scope": "story N.M" | "epic N",
  "date": "...",
  "stories_analyzed": 1,
  "root_causes": ["spec_gap", "wrong_assumption"],
  "lessons_count": 4,
  "preventive_measures_count": 2
}
```

---

## 8. Report to User

```
==================================================
  POST-MORTEM COMPLETE
==================================================

  Analyzed:  {count} stories
  Root causes found: {total_count}
    {for each category with count > 0}
    {category}: {count}
    {/for}

  Story docs updated:
    {for each story}
    stories/{story_file} — Post-Mortem section added
    {/for}

  {if epic_summary}
  Epic summary: post-mortems/epic-{N}.md
  {/if}

  Key lessons:
    {top 3 lessons, one line each}

  {if systemic_issues}
  !! Systemic issues detected:
    {list each with suggested action}
  {/if}

  {if preventive_measures contain revise_decision}
  Suggested next steps:
    {list /replan commands for decisions that need revision}
  {/if}

  Future agents reading these story docs will now see:
    - What went wrong and why
    - What assumptions were incorrect
    - What to check, avoid, and prefer
==================================================
```

---

## 9. Integration Points

- **After `/sprint-exec`**: When a story fails or gets blocked, run `/post-mortem --story=N.M` to document why before retrying
- **After `/audit-story`**: If audit reveals NEEDS_REWORK or INCOMPLETE, a post-mortem captures the deeper context about what went wrong
- **Before `/sprint-exec --story=N.M` (re-run)**: The executor reads the story doc and sees the Post-Mortem section — it knows what failed before and what to avoid
- **Feeds `/retro`**: The retro skill should check for `post-mortems/` directory and incorporate per-story findings into the sprint retrospective
- **Feeds `/replan`**: If post-mortem identifies a broken architecture decision, suggests `/replan` commands
- **Feeds next sprint Phase 0**: Post-mortem findings in story docs persist through sprint archival. When a story is carried forward, its post-mortem travels with it.

### How Future Agents Benefit

The Post-Mortem section is placed at the **end** of the story file intentionally. When an executor agent reads a story doc for re-implementation:

1. It reads the spec (what to build)
2. It reads the Dev Agent Record (what was tried before)
3. It reads the Audit Report (what's missing, if present)
4. It reads the Post-Mortem (what went wrong and why)

This gives the agent a complete picture: the plan, the attempt, the gaps, and the lessons — in that order. The "Lessons for Future Agents" subsection is written in direct imperative style specifically so agents treat it as implementation guidance.
