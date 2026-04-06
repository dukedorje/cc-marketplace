---
name: log
description: Append timestamped entries to a work log with auto-detected sprint/epic/story context. Creates a running record of decisions, discoveries, and notes.
user-invocable: true
argument-hint: "\"<message>\" [--story=N.M] [--epic=N] [--decision=D-NNN] [--tag=TAG] [--sprint=ID]"
---

# log: Work Log Annotation System

Append timestamped, context-aware entries to a work log. Automatically detects sprint context when available and enriches entries with epic/story/decision references.

---

## 1. Argument Parsing

Parse `$ARGUMENTS`:

| Argument | Required | Description |
|----------|----------|-------------|
| `<message>` | Yes | The log entry text (first positional argument, quoted string) |
| `--story=N.M` | No | Associate with a specific story |
| `--epic=N` | No | Associate with a specific epic |
| `--decision=D-NNN` | No | Reference an architecture decision |
| `--tag=TAG` | No | Free-form tag (can be repeated: `--tag=auth --tag=perf`) |
| `--doc` | No | Also create/update a permanent doc from this entry (see section 5) |
| `--doc=SUBDIR/NAME` | No | Create permanent doc at `docs/SUBDIR/NAME.md` |

If no message is provided:
```
Usage: /log "your message here" [--story=N.M] [--epic=N] [--decision=D-NNN] [--tag=TAG] [--doc[=PATH]]

Examples:
  /log "figured out auth needs refresh tokens"
  /log "SSE requires custom adapter for ky" --story=3.2 --decision=D-005
  /log "decided to use Zustand over Redux" --tag=state-management --doc
  /log "webhook retry logic" --doc=architecture/webhook-retries
```

---

## 2. Auto-Detect Sprint Context

### Sprint Resolution

Resolve the target sprint directories:
1. If `--sprint=<id>` was provided in `$ARGUMENTS`, set `STATE_DIR` = `.omc/sprint-plan/<id>/`
2. Else if `state_read` is available, read key `morphist.active_sprint`. If set, `STATE_DIR` = `.omc/sprint-plan/<value>/`
3. Else if `.omc/sprint-plan/current` symlink exists, `STATE_DIR` = `.omc/sprint-plan/current/`
4. Otherwise: no sprint context (proceed to section 2b).

If `STATE_DIR` was resolved, verify `STATE_DIR/phase-state.json` exists. Read it and set `SPEC_DIR` from the `spec_dir` field.
If `SPEC_DIR` does not exist, treat as no sprint context (proceed to section 2b).

### 2a. If Sprint Exists

Extract:
- `sprint_number`
- `current_phase`
- `execution_status` (if present)

If `--story` or `--epic` was provided, validate they exist:
- Read `SPEC_DIR/epics.md` to confirm the epic exists
- If `--story=N.M`, find the story file and extract its title

If neither `--story` nor `--epic` was provided, attempt to infer context:
- If `execution_status` is `"in-progress"`, check `execution_log` for the most recent story being worked on
- If found, auto-tag the entry with that epic/story (note it as "inferred" in the entry)

If `--decision` was provided, read the decision title from `SPEC_DIR/architecture-decisions.md`.

### 2b. If No Sprint

No sprint context is added. The entry is project-level only. This is fine — the log works outside sprints too.

---

## 3. Determine Log File

The work log file depends on context:

1. **Sprint-scoped** (sprint exists): `STATE_DIR/work-log.md`
2. **Project-scoped** (no sprint): `.omc/work-log.md`

If the log file doesn't exist, create it with a header:

```markdown
# Work Log

Running record of decisions, discoveries, and annotations.
```

---

## 4. Write Log Entry

Append the entry to the log file. Format:

```markdown
---

### {ISO 8601 timestamp}

{message}

{if sprint context}
**Sprint**: {sprint_number} | **Phase**: {current_phase}
{/if}
{if epic}**Epic {N}**: {epic_title}{/if}
{if story}**Story {N.M}**: {story_title}{/if}
{if decision}**Decision {D-NNN}**: {decision_title}{/if}
{if tags}**Tags**: {comma-separated tags}{/if}
{if doc_created}**Doc**: [{doc_path}]({relative_path_to_doc}){/if}
```

### 4a. Cross-Reference in Story File

If `--story=N.M` was specified (not inferred), also append a brief reference in the story file under a `## Work Log` section (create if missing, insert before `## Dev Agent Record`):

```markdown
## Work Log

- [{timestamp}] {first line of message} → [full entry](../../work-log.md#{anchor})
```

This creates a bidirectional link: the log references the story, the story references the log.

---

## 5. Permanent Documentation (`--doc`)

When `--doc` is specified, create or update a permanent document in the project's `docs/` directory.

### 5a. Determine Doc Path

- `--doc=SUBDIR/NAME` → `docs/SUBDIR/NAME.md`
- `--doc` (no path) → infer from tags and content:
  - If `--tag` provided: `docs/{first_tag}/{slugified_message_summary}.md`
  - If `--decision` provided: `docs/decisions/{D-NNN}-{slug}.md`
  - Otherwise: `docs/notes/{slugified_message_summary}.md`

### 5b. Find Docs Directory

Look for a `docs/` directory:
1. Check the working directory root
2. Check any additional working directories provided in the environment
3. If no `docs/` directory exists anywhere, create one at the working directory root

### 5c. Create or Update Doc

If the doc file **doesn't exist**, create it:

```markdown
# {Title derived from message}

{expanded message content}

{if sprint context}
## Context

- Sprint: {sprint_number}
- Epic: {N} — {epic_title}
- Story: {N.M} — {story_title}
- Decision: {D-NNN} — {decision_title}
{/if}

## References

- Work log entry: {timestamp}
{if story}- Story spec: `SPEC_DIR/stories/{story_file}`{/if}
{if decision}- Architecture decision: `SPEC_DIR/architecture-decisions.md#{D-NNN}`{/if}
```

If the doc file **already exists**, append a new section with the update:

```markdown

## Update: {ISO 8601 timestamp}

{message content}

{references as above}
```

### 5d. Expand Content

When `--doc` is used, the log message may be brief but the doc should be more complete. Dispatch a writer agent (haiku) to expand the message into proper documentation:

```
You are writing a brief technical document based on a work log annotation.

Message: {message}
{if sprint context}Sprint context: {sprint/epic/story/decision details}{/if}

Write a clear, concise document (3-10 paragraphs) that:
1. Explains the concept, decision, or discovery
2. Provides enough context for someone unfamiliar to understand
3. Includes any technical details implied by the message
4. References the sprint artifacts where applicable

Keep it practical and direct. No filler.
```

---

## 6. Confirm to User

After writing the entry:

```
Logged: {first 80 chars of message}...
  {timestamp}
  {if epic}Epic {N}{/if} {if story}Story {N.M}{/if} {if decision}{D-NNN}{/if}
  {if tags}Tags: {tags}{/if}
  → {log_file_path}
  {if doc_created}
  Doc created: {doc_path}
  {/if}
```

---

## 7. Viewing the Log

The work log is a plain markdown file. Users can:
- Read it directly: `cat STATE_DIR/work-log.md`
- Search it: grep for tags, story refs, dates
- It's consumed by `/retro` for retrospective generation

### Integration with `/retro`

The retro skill should read `STATE_DIR/work-log.md` if it exists and pass it to the story execution analysis agent. Work log entries provide context that Dev Agent Records don't capture — human decisions, discoveries, course corrections.

---

## 8. Integration Points

- **`/sprint-exec`**: After a blocker triage decision, auto-log the decision with the story context
- **`/replan`**: After replanning completes, auto-log the reason and affected stories
- **`/refine`**: After revising an ADR, auto-log the revision with decision context
- **`/retro`**: Reads the work log for additional sprint context
- **`/doc`**: The `--doc` flag bridges ephemeral log entries to permanent documentation
