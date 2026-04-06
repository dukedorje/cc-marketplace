---
name: backlog
description: Persistent cross-sprint backlog for follow-ups, tech debt, ideas, and deferred work. Lives outside sprint artifacts so items survive archival. Scan stories for implicit follow-ups or add items manually.
user-invocable: true
argument-hint: "\"<item>\" [--type=TYPE] [--priority=P] | --scan [--story=N.M] [--epic=N] | --from-retro | --from-post-mortem[=N.M] | --show | --triage | --promote"
---

# backlog: Persistent Cross-Sprint Backlog

Manages a persistent backlog at `.omc/backlog.md` that survives sprint archival. Captures follow-ups, tech debt, ideas, deferred requirements, and preventive measures from any source — manual entry, story scanning, retro extraction, or post-mortem findings.

The backlog feeds into Phase 0 of the next `/sprint-plan` run, ensuring nothing falls through the cracks between sprints.

---

## 1. Argument Parsing

Parse `$ARGUMENTS`:

| Argument | Description |
|----------|-------------|
| `"<item>"` | Add a new backlog item (first positional argument, quoted string) |
| `--type=TYPE` | Item type: `follow-up`, `tech-debt`, `idea`, `deferred`, `preventive`, `bug` (default: `follow-up`) |
| `--priority=P` | Priority: `critical`, `high`, `medium`, `low` (default: `medium`) |
| `--story=N.M` | Associate with a story (for manual add) or scope a scan |
| `--epic=N` | Associate with an epic (for manual add) or scope a scan |
| `--tag=TAG` | Free-form tag (repeatable: `--tag=auth --tag=perf`) |
| `--scan` | Scan stories for implicit follow-up items (TODOs, workarounds, deferred work) |
| `--from-retro` | Extract items from the latest retrospective |
| `--from-post-mortem[=N.M]` | Extract preventive measures from post-mortems |
| `--show` | Display the current backlog |
| `--show=TYPE` | Filter display by type |
| `--triage` | Interactive prioritization session |
| `--promote` | Select backlog items to promote into the next sprint's requirements |
| `--done=ID` | Mark a backlog item as resolved |
| `--drop=ID` | Remove a backlog item (with reason) |

If no arguments:
```
Usage: /backlog "<item>" [--type=TYPE] [--priority=P]
       /backlog --scan [--epic=N]
       /backlog --from-retro | --from-post-mortem
       /backlog --show | --triage | --promote

Persistent backlog — survives sprint archival.

Quick add:
  /backlog "add retry logic for webhook failures"
  /backlog "explore caching strategy for API responses" --type=idea
  /backlog "fix N+1 query in user listing" --type=tech-debt --priority=high

Scan stories:
  /backlog --scan                          # Scan all stories in current sprint
  /backlog --scan --epic=2                 # Scan Epic 2 only
  /backlog --scan --story=2.3              # Scan a single story

Import:
  /backlog --from-retro                    # Pull from latest retrospective
  /backlog --from-post-mortem              # Pull from all post-mortems
  /backlog --from-post-mortem=2.3          # Pull from specific story's post-mortem

Manage:
  /backlog --show                          # List all items
  /backlog --show=tech-debt                # Filter by type
  /backlog --triage                        # Interactive prioritization
  /backlog --promote                       # Select items for next sprint
  /backlog --done=BL-007                   # Mark resolved
  /backlog --drop=BL-003                   # Remove with reason
```

---

## 2. Backlog File Location & Format

### 2a. File Location

The backlog lives at `.omc/backlog.md` — a sibling of `sprint-plan/`, **not** inside any sprint directory. This ensures it persists when sprints are archived.

If the file doesn't exist, create it:

```markdown
---
last_id: 0
item_count: 0
---

# Backlog

Persistent cross-sprint backlog. Items survive sprint archival and feed into Phase 0 of the next sprint.
```

### 2b. Item Format

Each item is a markdown section with structured fields:

```markdown
---

### BL-{NNN}: {title}

- **Type**: {follow-up|tech-debt|idea|deferred|preventive|bug}
- **Priority**: {critical|high|medium|low}
- **Status**: {open|promoted|resolved|dropped}
- **Added**: {ISO 8601 timestamp}
- **Source**: {manual|scan:story-N.M|retro:sprint-N|post-mortem:story-N.M}
- **Story**: {N.M — if associated}
- **Epic**: {N — if associated}
- **Tags**: {comma-separated, if any}

{description — 1-3 sentences of context}

{if source is scan or import, include evidence}
> **Evidence**: {quote or file:line reference that triggered this item}
```

### 2c. ID Assignment

Read the `last_id` from frontmatter, increment for each new item, update frontmatter after writing.

---

## 3. Manual Add (default action)

When a quoted string is provided as the first argument:

1. Read `.omc/backlog.md` (create if missing)
2. Parse `--type`, `--priority`, `--story`, `--epic`, `--tag` flags
3. Assign the next `BL-{NNN}` ID
4. Append the item to the backlog file
5. Update frontmatter (`last_id`, `item_count`)
6. If `--story` or `--epic` provided and sprint is active, cross-reference in the story file (add a line under `## Work Log` or create it):
   ```markdown
   - [{timestamp}] Backlog item added: BL-{NNN} — {title}
   ```

Report:
```
Added: BL-{NNN} — {title}
  Type: {type} | Priority: {priority}
  {if story}Story: {N.M}{/if} {if epic}Epic: {N}{/if}
  → .omc/backlog.md
```

---

## 4. Story Scanning (`--scan`)

Scan story files for implicit follow-up indicators that agents or humans left behind. This is the primary way to catch things that should be in the backlog but aren't.

### 4a. Determine Scan Scope

- `--scan --story=N.M`: scan one story
- `--scan --epic=N`: scan all stories in an epic
- `--scan` (no scope): scan all stories in the current sprint

Read `STATE_DIR/phase-state.json` to confirm a sprint exists (use sprint resolution: `--sprint` arg > `morphist.active_sprint` state > `current` symlink). Set `SPEC_DIR` from the `spec_dir` field in `phase-state.json`.

### 4b. Scan Patterns

For each story in scope, read the story file and scan for:

**In Dev Agent Record / Completion Notes:**

| Pattern | Backlog Type | Priority |
|---------|-------------|----------|
| `TODO`, `FIXME`, `HACK`, `XXX` | tech-debt | medium |
| `workaround`, `temporary`, `stopgap` | tech-debt | high |
| `deferred`, `skipped`, `punted`, `out of scope` | deferred | medium |
| `should also`, `would be nice`, `future improvement` | idea | low |
| `not implemented`, `partial`, `stub` | follow-up | high |
| `needs testing`, `untested`, `no tests` | follow-up | medium |
| `security concern`, `vulnerability`, `unsafe` | bug | critical |

**In Post-Mortem section (if present):**

| Pattern | Backlog Type | Priority |
|---------|-------------|----------|
| Preventive measures | preventive | inherits severity |
| Lessons of type `check` | preventive | medium |
| Lessons of type `avoid` | preventive | high |

**In Audit Report section (if present):**

| Pattern | Backlog Type | Priority |
|---------|-------------|----------|
| Work items with priority `must` | follow-up | high |
| Work items with priority `should` | follow-up | medium |
| Architecture issues | tech-debt | inherits severity |

**In implementation files (from Dev Agent Record file list):**

Read each file listed in the Dev Agent Record and scan for `TODO`, `FIXME`, `HACK`, `XXX`, `WORKAROUND` comments. These are direct evidence of deferred work.

### 4c. Deduplicate

Before adding scanned items:
1. Read existing backlog items
2. For each candidate, check if a similar item already exists (same story + same pattern match or substantially similar title)
3. Skip duplicates, noting them in the scan report

### 4d. Present Findings

Show findings to the user before committing:

```
==================================================
  BACKLOG SCAN: {scope}
==================================================

  Stories scanned: {count}
  Items found: {count}
  Duplicates skipped: {count}

  New items to add:
    [{type}] {title} — from Story {N.M}
      > {evidence quote}
      Priority: {priority}

    [{type}] {title} — from Story {N.M}
      > {evidence quote}
      Priority: {priority}

    ... (grouped by story)

  ---
  Add all {count} items to backlog? (yes / select / skip)
==================================================
```

**Options:**
- `yes` — add all items
- `select` — present each item for individual accept/reject
- `skip` — don't add any (scan report is still useful)

Add confirmed items to `.omc/backlog.md`.

---

## 5. Import from Retrospective (`--from-retro`)

Extract actionable items from the latest sprint retrospective.

### 5a. Find Retrospective

Read `SPEC_DIR/retrospective.md`. If not found, check the most recent sprint's spec directory for `retrospective.md`.

If no retrospective found:
```
No retrospective found. Run /retro first.
```

### 5b. Extract Items

Parse the retrospective for backlog-worthy items:

| Retro Section | Backlog Type | Priority |
|---------------|-------------|----------|
| Technical Debt Created (table rows) | tech-debt | from retro's Priority column |
| Architecture Decision Review — "needed revision" | preventive | high |
| Unfinished Work (failed/blocked stories) | follow-up | high |
| Deferred Requirements — "ready to promote" | deferred | medium |
| Learnings for Next Sprint (actionable items) | preventive | medium |
| Technical Debt Priority (ordered list) | tech-debt | from retro's priority |

### 5c. Present and Confirm

Same flow as scan (section 4d) — show extracted items, let user confirm.

Tag all imported items with `source: retro:sprint-{N}`.

---

## 6. Import from Post-Mortems (`--from-post-mortem`)

### 6a. Find Post-Mortems

- `--from-post-mortem=N.M`: read that story's Post-Mortem section
- `--from-post-mortem` (no scope): find all stories with `post_mortem: true` in frontmatter, plus all files in `STATE_DIR/post-mortems/`

### 6b. Extract Items

From each post-mortem:

| Post-Mortem Section | Backlog Type | Priority |
|---------------------|-------------|----------|
| Preventive Measures — `revise_decision` | preventive | high |
| Preventive Measures — `add_test` | follow-up | medium |
| Preventive Measures — `add_check` | preventive | medium |
| Preventive Measures — `process_change` | preventive | low |
| Lessons of type `avoid` | preventive | high |
| Lessons of type `check` or `specify` | preventive | medium |

### 6c. Present and Confirm

Same flow as scan (section 4d). Tag with `source: post-mortem:story-{N.M}`.

---

## 7. Display (`--show`)

### 7a. Basic Display

Read `.omc/backlog.md` and render a summary:

```
==================================================
  BACKLOG ({open_count} open, {total_count} total)
==================================================

  Critical ({count}):
    BL-{NNN}: {title} [{type}] — {source}
    BL-{NNN}: {title} [{type}] — {source}

  High ({count}):
    BL-{NNN}: {title} [{type}] — {source}

  Medium ({count}):
    BL-{NNN}: {title} [{type}] — {source}

  Low ({count}):
    BL-{NNN}: {title} [{type}] — {source}

  ---
  Resolved: {count} | Dropped: {count} | Promoted: {count}
==================================================
```

### 7b. Filtered Display

`--show=tech-debt` shows only items of that type. Same format, filtered.

---

## 8. Triage (`--triage`)

Interactive prioritization session for open backlog items.

### 8a. Load Items

Read all open items from `.omc/backlog.md`, sorted by current priority then age.

### 8b. Present Each Item

For each open item:

```
BL-{NNN}: {title}
  Type: {type} | Current priority: {priority}
  Source: {source}
  {if story}Story: {N.M}{/if}
  Added: {date}

  {description}
  {if evidence}> {evidence}{/if}

  Priority? (critical / high / medium / low / drop / skip)
```

- `critical/high/medium/low` — update priority
- `drop` — mark as dropped (prompts for reason)
- `skip` — leave unchanged, move to next

### 8c. Summary

After triage:
```
Triage complete:
  Reprioritized: {count}
  Dropped: {count}
  Unchanged: {count}
```

---

## 9. Promote (`--promote`)

Select backlog items to feed into the next sprint's planning.

### 9a. Show Candidates

Display all open items sorted by priority, with a recommendation:

```
==================================================
  PROMOTE TO NEXT SPRINT
==================================================

  Recommended (critical + high):
    [ ] BL-{NNN}: {title} [{type}]
    [ ] BL-{NNN}: {title} [{type}]

  Also consider (medium):
    [ ] BL-{NNN}: {title} [{type}]

  Select items to promote (comma-separated IDs, or "all-recommended"):
==================================================
```

### 9b. Mark as Promoted

For selected items:
1. Update status to `promoted`
2. Add `promoted_at: {ISO 8601}` and `promoted_to: sprint-{N+1}` to the item
3. Write a promotion manifest to `.omc/backlog-promoted.md`:

```markdown
---
promoted_at: {ISO 8601}
target_sprint: {N+1}
items_count: {count}
---

# Promoted Backlog Items

Items selected for inclusion in the next sprint's planning.

{for each promoted item}
## BL-{NNN}: {title}

- **Type**: {type}
- **Priority**: {priority}
- **Original source**: {source}
- **Description**: {description}
{/for}
```

This file is consumed by Phase 0 (Discovery) of the next `/sprint-plan` run.

---

## 10. Resolve and Drop (`--done`, `--drop`)

### 10a. Resolve (`--done=BL-NNN`)

1. Find the item in `.omc/backlog.md`
2. Update status to `resolved`
3. Add `resolved_at: {ISO 8601}`
4. Report: `Resolved: BL-{NNN} — {title}`

### 10b. Drop (`--drop=BL-NNN`)

1. Find the item
2. Ask for reason: `Why are you dropping BL-{NNN}? (brief reason)`
3. Update status to `dropped`
4. Add `dropped_at: {ISO 8601}` and `drop_reason: {reason}`
5. Report: `Dropped: BL-{NNN} — {title} (reason: {reason})`

---

## 11. Auto-Capture Integration

Other skills should call backlog to add items automatically. This section defines the integration points — each skill adds items by appending to `.omc/backlog.md` using the same format.

### 11a. From `/post-mortem`

After writing the Post-Mortem section (post-mortem skill section 5), for each preventive measure with type `revise_decision`, `add_test`, or `add_check`:

Append to backlog:
```markdown
### BL-{NNN}: {measure description}

- **Type**: preventive
- **Priority**: {high if revise_decision, medium otherwise}
- **Status**: open
- **Added**: {timestamp}
- **Source**: post-mortem:story-{N.M}
- **Story**: {N.M}
- **Tags**: post-mortem, {root_cause_category}

{measure description with context from the post-mortem}

> **Evidence**: Post-mortem for Story {N.M} — root cause: {category}
```

### 11b. From `/retro`

After writing the retrospective (retro skill section 5), for each item in the Technical Debt table and each unfinished story:

Append to backlog with `source: retro:sprint-{N}`.

### 11c. From `/audit`

After writing the audit report (audit skill section 6), for each work item with priority `must` in stories that are NOT going to be immediately re-executed:

Append to backlog with `source: audit:story-{N.M}`.

### 11d. From `/sprint-exec`

When a story completes with `blocker_type` set (partial completion), or when Completion Notes contain backlog-indicator patterns (TODO, workaround, deferred):

Append to backlog with `source: exec:story-{N.M}`.

### 11e. Auto-Capture Behavior

All auto-captured items:
- Are deduplicated against existing backlog items before adding
- Include an `> Evidence:` quote with the source text that triggered capture
- Are logged to the work log with tag `backlog`
- Print a brief notification: `Backlog: added BL-{NNN} — {title} (from {source})`

Skills should make auto-capture **graceful** — if `.omc/backlog.md` doesn't exist or can't be written, skip silently. The backlog is valuable but never blocks other workflows.

---

## 12. Phase 0 Integration

When `/sprint-plan` runs Phase 0 (Discovery), it should check for:

1. `.omc/backlog-promoted.md` — promoted items ready for this sprint
2. `.omc/backlog.md` — all open items (for awareness, not automatic inclusion)

The discovery agent should include in `discovery.md`:

```markdown
## Backlog Carry-Forward

### Promoted Items ({count})
{list promoted items — these MUST be included in requirements}

### Open Backlog ({count} items)
{summary by type and priority — for awareness during requirements phase}
```

Promoted items should flow into Phase 1 (Requirements) as pre-approved FRs or NFRs, tagged with their backlog ID for traceability.

After the sprint is initialized, clear `.omc/backlog-promoted.md` (the items are now tracked in sprint artifacts). The items remain in `.omc/backlog.md` with status `promoted`.

---

## 13. Report Completion

For add operations:
```
Added: BL-{NNN} — {title}
  Type: {type} | Priority: {priority}
  → .omc/backlog.md
```

For scan/import operations:
```
Backlog updated: {added_count} items added, {skipped_count} duplicates skipped
  → .omc/backlog.md ({total_open} open items)
```

---

## 14. Integration Summary

| Skill | Trigger | Backlog Action |
|-------|---------|----------------|
| `/post-mortem` | Preventive measures generated | Auto-add preventive items |
| `/retro` | Retrospective complete | Auto-add tech debt + unfinished work |
| `/audit` | Stories need rework but won't be re-executed immediately | Auto-add must-priority work items |
| `/sprint-exec` | Story completes with workarounds/TODOs | Auto-add follow-up items |
| `/sprint-plan` | Phase 0 discovery | Read promoted items + open backlog |
| `/backlog --scan` | Manual trigger | Scan stories for implicit follow-ups |
| `/backlog --from-retro` | Manual trigger | Import from retrospective |
| `/backlog --from-post-mortem` | Manual trigger | Import from post-mortems |
| `/backlog --promote` | Pre-sprint prep | Select items for next sprint |
