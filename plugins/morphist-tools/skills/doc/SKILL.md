---
name: doc
description: Create or update permanent documentation in docs/. Auto-enriches with sprint context, architecture decisions, and code references when available.
user-invocable: true
argument-hint: "<topic-or-path> [--from-story=N.M] [--from-epic=N] [--from-decision=D-NNN] [--sprint=ID]"
---

# doc: Permanent Documentation Writer

Create or update permanent technical documentation in the project's `docs/` directory. When sprint context is available, enriches docs with architecture decisions, story specs, and implementation details.

---

## 1. Argument Parsing

Parse `$ARGUMENTS`:

| Argument | Required | Description |
|----------|----------|-------------|
| `<topic-or-path>` | Yes | Either a topic string or a path like `architecture/auth-flow` |
| `--from-story=N.M` | No | Derive doc content from a completed story's spec and implementation |
| `--from-epic=N` | No | Derive doc content from an epic's stories and decisions |
| `--from-decision=D-NNN` | No | Document a specific architecture decision in detail |
| `--update` | No | Update an existing doc rather than creating new |

If no arguments provided:
```
Usage: /doc <topic-or-path> [--from-story=N.M] [--from-epic=N] [--from-decision=D-NNN]

Creates permanent documentation in docs/.

Examples:
  /doc "authentication flow"                    # New doc from topic
  /doc architecture/auth-flow                   # Explicit path
  /doc api/webhooks --from-story=2.3            # From a story's implementation
  /doc architecture/state-management --from-decision=D-007
  /doc --from-epic=2                            # Document everything in Epic 2
```

---

## 2. Determine Doc Path

### 2a. Explicit Path

If the argument looks like a path (contains `/`):
- Target: `docs/{argument}.md`
- Title: derived from the last path segment

### 2b. Topic String

If the argument is a plain string:
- Slugify: lowercase, replace spaces with hyphens
- If `--from-decision`: `docs/decisions/{slug}.md`
- If `--from-epic`: `docs/epics/epic-{N}-{slug}.md`
- If `--from-story`: `docs/stories/{slug}.md`
- Otherwise: `docs/{slug}.md`

### 2c. Find Docs Directory

Search for `docs/` in:
1. Current working directory
2. Additional working directories from environment
3. If not found, create `docs/` at the working directory root

Create any necessary subdirectories.

---

## 3. Sprint Resolution

Resolve the target sprint directories:
1. If `--sprint=<id>` was provided in `$ARGUMENTS`, set `STATE_DIR` = `.omc/sprint-plan/<id>/`
2. Else if `state_read` is available, read key `morphist.active_sprint`. If set, `STATE_DIR` = `.omc/sprint-plan/<value>/`
3. Else if `.omc/sprint-plan/current` symlink exists, `STATE_DIR` = `.omc/sprint-plan/current/`
4. Otherwise halt: "No active sprint found. Run `/sprint-plan` first, or pass `--sprint=<id>`."

Verify `STATE_DIR/phase-state.json` exists. Read it and set `SPEC_DIR` from the `spec_dir` field.
Verify `SPEC_DIR` exists. If not, halt: "Sprint spec directory not found at {spec_dir}. Sprint may need re-initialization."

Skip this section if no `--from-story`, `--from-epic`, or `--from-decision` flag was provided — sprint context is optional for `/doc`.

---

## 4. Gather Source Material

### 4a. If `--from-story=N.M`

Read:
- Story file: `SPEC_DIR/stories/{story_file}`
- Architecture decisions referenced by the story
- Files listed in the Dev Agent Record (read each file for implementation details)
- Any existing work log entries referencing this story

### 4b. If `--from-epic=N`

Read:
- All story files in the epic
- Architecture decisions referenced by any story in the epic
- Epic section from `SPEC_DIR/epics.md`
- Decision graph entries for this epic (if `SPEC_DIR/decision-graph.md` exists)
- Work log entries referencing this epic

### 4c. If `--from-decision=D-NNN`

Read:
- The decision from `SPEC_DIR/architecture-decisions.md`
- Decision graph: which stories depend on this decision
- Story files that reference this decision (Architecture Compliance sections)
- Replan log entries for this decision (if any)

### 4d. Topic Only (no `--from-*`)

Read the codebase for relevant context:
- Search for files related to the topic (Glob/Grep)
- Check if sprint artifacts exist and contain relevant information
- Check work log for related entries

---

## 5. Generate Documentation

Dispatch a writer agent (sonnet) to create the document:

```
You are writing permanent technical documentation for a software project.

## Topic
{topic_or_title}

## Source Material
{all gathered context from section 3}

## Instructions

Write a clear, well-structured technical document. This is PERMANENT documentation
that will live in the project's docs/ directory and be read by developers who may
not have access to sprint planning artifacts.

Guidelines:
1. Lead with what the reader needs to know, not how you learned it
2. Include code examples where they clarify (extract from actual implementation files)
3. Document the WHY behind decisions, not just the WHAT
4. Reference source files with relative paths from the project root
5. If derived from sprint artifacts, translate sprint jargon into permanent language
   (e.g., "Story 2.3" becomes "the webhook retry implementation")
6. Include a ## References section at the end linking to relevant files

Structure:
- Start with a one-paragraph summary
- Use ## sections for major topics
- Use ### for subtopics
- Include code blocks with language tags
- Keep it concise — aim for completeness without verbosity

{if from_story or from_epic}
IMPORTANT: This doc should stand alone. A reader should NOT need to look at
sprint planning artifacts to understand it. Translate all story/epic context
into plain technical documentation.
{/if}
```

---

## 6. Write the Document

Write the generated content to the determined path.

If `--update` was specified and the file exists:
- Read the existing document
- Pass both the existing content and the new source material to the writer
- Instruct the writer to update/extend the existing doc, not replace it

If the file already exists and `--update` was NOT specified:
- Ask the user:
  ```
  Doc already exists at {path}. Options:
  1. Update it (merge new content)
  2. Overwrite it
  3. Choose a different path
  ```

---

## 7. Cross-Reference

### 7a. Log the Doc Creation

If a sprint is active, append to the work log (`STATE_DIR/work-log.md`):

```markdown
---

### {timestamp}

Created documentation: [{doc_title}]({doc_path})

{if from_story}**Story {N.M}**: {story_title}{/if}
{if from_epic}**Epic {N}**: {epic_title}{/if}
{if from_decision}**Decision {D-NNN}**: {decision_title}{/if}
**Tags**: documentation
```

### 7b. Reference in Story/Epic

If `--from-story` or `--from-epic` was used, add a reference in the story file(s) under `## Work Log` (create if needed):

```markdown
- [{timestamp}] Doc created: [{doc_title}]({relative_path_to_doc})
```

---

## 8. Confirm to User

```
Doc created: {doc_path}
  Title: {title}
  {if from_story}Source: Story {N.M} — {story_title}{/if}
  {if from_epic}Source: Epic {N} — {epic_title} ({story_count} stories){/if}
  {if from_decision}Source: Decision {D-NNN} — {decision_title}{/if}
  {if work_log_updated}Work log updated{/if}
```

---

## 8. Batch Documentation

When `--from-epic=N` is used, the writer may produce a single comprehensive doc or suggest splitting into multiple docs. If the epic is large (>4 stories), recommend splitting:

```
Epic {N} has {count} stories. Options:
1. Single comprehensive doc covering the entire epic
2. One doc per major topic area (suggested split: {list})
3. One doc per story
```

For option 2 or 3, create multiple docs in `docs/epics/epic-{N}/`.

---

## 9. Integration Points

- **`/log --doc`**: The `/log` skill's `--doc` flag delegates to this skill's document generation logic (section 4)
- **`/retro`**: The retro could suggest `/doc --from-epic=N` for completed epics that produced significant learnings
- **`/sprint-review`**: Reviews could suggest documentation for complex implementations
- **Outside sprints**: `/doc "topic"` works without any sprint context — it searches the codebase and writes docs from what it finds
