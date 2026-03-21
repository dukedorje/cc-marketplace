---
name: help
description: Show complete morphist-tools skill reference. Triggers on "help", "how do I use morphist-tools", "what skills are available", or "sprint-plan help".
user-invocable: true
argument-hint: ""
---

Display the following usage guide directly to the user. Do NOT run any agents or tools — just output this text as-is:

---

# Morphist Tools — Complete Skill Reference

## Lifecycle at a Glance

```
  PLAN                PREPARE           EXECUTE            VERIFY & REVIEW       CLOSE
  ────                ───────           ───────            ───────────────       ─────
  /prd           ──>  /sprint-plan ──>  /epic-prep   ──>  /sprint-exec    ──>  /retro
                                        (optional)         │ (per epic)
                                                           ├─> /verify         (auto, inline)
                                                           ├─> /sprint-review  (auto, background)
                                                           ├─> /reconcile
                                                           └─> /review-fix

  AT ANY TIME:  /status  /ral  /audit-story  /replan  /log  /doc  /update-status  /ultraresearch
```

**Typical happy path**: `/prd` -> `/sprint-plan` -> `/sprint-exec` -> `/retro`

Everything else is for refinement, course correction, or quality assurance.

---

## 1. `/prd` — Create a Product Requirements Document

**When**: You have an idea but no formal spec yet. Always start here.

```
/prd "build a real-time dashboard for sensor data"
/prd path/to/notes.md
/prd --fast "quick prototype"
```

| Flag | Effect |
|------|--------|
| `--fast` | Skip interview, generate PRD in one pass |

Conducts a structured interview, researches the codebase for context, and produces a scope-tiered PRD at `.omc/sprint-plan/prd-{slug}.md`.

The PRD is the input to `/sprint-plan`.

---

## 2. `/sprint-plan` — Multi-Phase Sprint Planning

**When**: You have a PRD (or idea) and want implementation-ready stories.

```
/sprint-plan "your idea"                       # From an idea (runs lightweight PRD internally)
/sprint-plan path/to/prd.md                    # From an existing PRD
/sprint-plan --fast "quick prototype"          # Single-pass, no consensus
/sprint-plan --continue                        # Resume from last incomplete phase
/sprint-plan --continue=architecture           # Resume from after architecture
/sprint-plan --restart-from=architecture       # Redo architecture and all downstream
```

| Flag | Effect |
|------|--------|
| `--fast` | Single-pass mode, no RALPLAN-DR consensus loops |
| `--thorough` | Explicit thorough mode (default) |
| `--skip-ux` | Skip the optional UX Design phase |
| `--continue` | Resume from next incomplete phase |
| `--continue=<phase>` | Resume from after specified phase |
| `--restart-from=<phase>` | Re-run a phase and all downstream |

### Phases (in order)

| # | Phase | What happens |
|---|-------|-------------|
| 0 | Discovery | Scans codebase, inventories existing code, finds planning artifacts |
| 1 | Requirements | Expands PRD into FRs, NFRs, constraints (parallel agents) |
| 1.5 | UX Design (optional) | Component specs, user flows for frontend projects |
| 2A | Architecture | Architecture decisions with RALPLAN-DR consensus |
| 2B | Epic Design | Groups requirements into user-value-focused epics |
| 3 | Stories | Breaks epics into dev-agent-sized stories with BDD criteria |
| 4 | Enrichment | Adds technical details, testing requirements, file lists per story |
| 5 | Validation | Checks FR coverage, dependencies, story quality |

### Output

All artifacts in `.omc/sprint-plan/sprint-NNN/` (symlinked as `current/`):

| File | Contents |
|------|----------|
| `discovery.md` | Project context |
| `requirements.md` | Functional and non-functional requirements |
| `ux-design.md` | Component specs and user flows (optional) |
| `architecture-decisions.md` | ADR-lite architecture decisions (D-001, D-002, ...) |
| `epics.md` | Epic structure with story summaries |
| `stories/E{N}-S{M}.md` | Enriched story files ready for dev agents |
| `readiness-report.md` | Final validation summary |

---

## 3. `/epic-prep` — Pre-Execution Deep Dive (Optional)

**When**: Before executing an epic, you want to enrich stories, revise decisions, or drill into specifics. Especially useful for complex or risky epics.

```
/epic-prep --epic=2                # Deep dive into Epic 2
/epic-prep --story=3.2             # Drill into a specific story
/epic-prep --graph                 # View the decision dependency graph
```

| Flag | Effect |
|------|--------|
| `--epic=N` | Target epic N for deep dive |
| `--story=N.M` | Drill into a specific story |
| `--graph` | Display the decision dependency graph |

Interactive session. Re-runnable — if new information surfaces (library changes, API discoveries, user feedback), run `/epic-prep --epic=N` again to add updated prep notes. Previous notes are preserved so executors see the full context.

If ADRs are revised during prep, suggests `/reconcile --decisions` to propagate changes to other epics.

---

## 4. `/sprint-exec` — Execute Stories

**When**: Planning is complete and you're ready to implement. This is where code gets written.

```
/sprint-exec                       # Execute NEXT incomplete epic (default)
/sprint-exec --full-auto           # Execute ALL remaining epics, no stopping
/sprint-exec --next-story          # Execute just the next unfinished story
/sprint-exec --epic=2              # Execute only Epic 2
/sprint-exec --story=2.3           # Execute only story 2.3 (infers epic)
/sprint-exec --dry-run             # Preview execution plan without running agents
/sprint-exec --concurrency=3       # Limit to 3 parallel agents per epic
```

| Flag | Effect |
|------|--------|
| *(no flags)* | Execute the next incomplete epic only (incremental) |
| `--full-auto` | Execute all remaining epics, auto-resolve failures and blockers |
| `--next-story` | Execute just the next unfinished story |
| `--epic=N` | Execute only epic N |
| `--story=N.M` | Execute only story N.M (uses opus model for retries) |
| `--dry-run` | Show plan without dispatching agents |
| `--concurrency=N` | Max parallel executor agents per epic |

**Behavior**:
- **Default is incremental**: just the next epic, so you stay in control
- `--full-auto` runs everything and auto-accepts partial failures / blockers
- Epics run sequentially, stories within an epic run in parallel
- After each epic: `/verify` runs inline (gate), then `/sprint-review` in background
- Resume-safe: re-running skips already-done stories
- Failed stories can be retried individually with `--story=N.M` (upgrades to opus)

---

## 5. `/ral` — Refine Any Phase Artifact

**When**: A phase is complete but you want higher quality. Runs a Planner -> Architect -> Critic adversarial consensus pass.

```
/ral architecture                  # Refine architecture decisions
/ral epics --epic=2                # Refine only Epic 2's design
/ral enrichment --story=3.1        # Refine a single story's enrichment
/ral prd                           # Refine the PRD
/ral retro                         # Refine the retrospective
/ral stories --force               # Override the 2-pass limit
```

| Flag | Effect |
|------|--------|
| `--epic=N` | Scope to epic N (valid for: `epics`, `stories`, `enrichment`) |
| `--story=N.M` | Scope to story N.M (valid for: `enrichment` only) |
| `--force` | Bypass the 2-pass-per-scope limit |

Valid phases: `requirements`, `architecture`, `epics`, `stories`, `enrichment`, `prd`, `retro`

Each scope gets 2 refinement passes before diminishing-returns warning. If the artifact changes, downstream phases are marked stale.

---

## 6. `/sprint-review` — Review Completed Work

**When**: An epic (or all epics) has been executed and you want a quality review against specs and architecture decisions. Runs automatically after each epic in `/sprint-exec`, but can also be called manually.

```
/sprint-review                     # Review the most recently completed epic
/sprint-review --epic=2            # Review Epic 2
/sprint-review --all               # Review all completed epics
```

| Flag | Effect |
|------|--------|
| `--epic=N` | Review specific epic |
| `--all` | Review all completed epics |

Output lands in `.omc/sprint-plan/current/reviews/`.

---

## 7. `/reconcile` — Fix Style Drift Across Agents

**When**: Parallel agent execution produced inconsistent naming, patterns, or conventions across stories/epics. Also use after `/epic-prep` revises ADRs to propagate changes.

```
/reconcile --epic=2                # Reconcile within Epic 2
/reconcile --all                   # Full-sprint cross-epic reconciliation
/reconcile --decisions             # Propagate ADR changes to dependent story specs
/reconcile --all --auto            # Auto-resolve everything (majority-wins)
```

| Flag | Effect |
|------|--------|
| `--epic=N` | Scope to epic N |
| `--all` | Full-sprint reconciliation |
| `--decisions` | Propagate ADR changes via decision graph (different mode) |
| `--auto` | Auto-resolve without asking (majority-wins rule) |

Dispatched automatically by `/sprint-review` (per-epic) and `/retro` (full-sprint).

---

## 8. `/review-fix` — Fix Issues from Reviews

**When**: You have findings from `/sprint-review` or `/reconcile` and want to validate and fix them. Checks each finding against actual code, discards false positives, fixes real issues.

```
/review-fix                        # Pick from recent review files
/review-fix path/to/review.md     # Fix findings in specific review
/review-fix --dry-run              # Preview what would be fixed
/review-fix --auto                 # Fix without asking per-finding
/review-fix --tdd                  # Generate failing tests as validation
```

| Flag | Effect |
|------|--------|
| `--auto` | Fix all real issues without per-finding confirmation |
| `--tdd` | Generate failing tests before fixing |
| `--dry-run` | Show plan without making changes |

---

## 9. `/verify` — Quick Epic Completion Check

**When**: After an epic executes, you want a fast independent check that files exist, ACs were addressed, and architecture was followed. Auto-runs between epics in `/sprint-exec`. For deep analysis, use `/audit-story` instead.

```
/verify                            # Verify most recently completed epic
/verify --epic=2                   # Verify Epic 2
/verify --story=2.3                # Verify a single story
/verify --all                      # Verify all done stories in sprint
```

| Flag | Effect |
|------|--------|
| `--epic=N` | Verify all done stories in epic N |
| `--story=N.M` | Verify a single story |
| `--all` | Verify all done stories across all epics |

Checks: file existence, import health, AC spot-check (YES/PARTIAL/NO), architecture compliance. Verdicts: PASS / CONCERNS / FAIL.

Auto-runs after each epic in `/sprint-exec`. Failures pause execution (unless `--full-auto`).

---

## 10. `/audit-story` — Deep Story Completion Audit

**When**: You want to verify that implemented stories actually meet their acceptance criteria. Especially useful after library changes, mid-sprint pivots, or before marking work as done.

```
/audit-story --story=3.2           # Audit a specific story
/audit-story --epic=2              # Audit all stories in Epic 2
/audit-story --all                 # Audit everything
/audit-story --all --context="switched from Eden Treaty to ky"
/audit-story --story=3.2 --tdd    # Generate failing tests as gates
/audit-story --dry-run             # Preview audit plan
```

| Flag | Effect |
|------|--------|
| `--story=N.M` | Audit specific story |
| `--epic=N` | Audit all stories in epic N |
| `--all` | Audit all stories |
| `--context="..."` | New facts to check against (library changes, etc.) |
| `--tdd` | Generate failing tests as validation gates |
| `--dry-run` | Preview without making changes |

Produces a gap analysis and actionable work plan. Updates story metadata so the next implementor (or `/sprint-exec --story=N.M` retry) knows what's left.

---

## 11. `/replan` — Mid-Sprint Course Correction

**When**: An architecture assumption, library choice, or dependency breaks mid-sprint. Surgically updates affected artifacts instead of restarting planning.

```
/replan --story=3.2 --reason="Eden Treaty doesn't support SSE"
/replan --epic=2 --reason="need to switch from SQLite to Postgres"
/replan --decision=D-005 --reason="chosen auth library is deprecated"
/replan --decision=D-005 --dry-run
```

| Flag | Effect |
|------|--------|
| `--story=N.M` | Replan starting from a specific story |
| `--epic=N` | Replan an entire epic |
| `--decision=D-NNN` | Replan around a specific architecture decision |
| `--reason="..."` | What broke and why (required context) |
| `--dry-run` | Show impact without making changes |

Updates architecture decisions, propagates changes to affected story specs, and marks impacted stories for re-execution.

---

## 12. `/status` / `/update-status` — View/Update Statuses

**When**: You want to see where things stand — what phase you're in, what artifacts exist, epic/story progress. Also use to fix statuses after manual work.

`/status` is a quick alias for `/update-status --show`.

```
/status                            # Sprint overview + dashboard
/update-status                     # Same as /status (default is --show)
/update-status --show              # Same as above
/update-status --sync              # Recalculate epic statuses from stories
/update-status --epic=2 --status=done
/update-status --story=3.1 --status=done
```

| Flag | Effect |
|------|--------|
| `--show` | Display status dashboard with decision graph (default) |
| `--sync` | Recalculate epic statuses from their story statuses |
| `--epic=N --status=VALUE` | Set epic status |
| `--story=N.M --status=VALUE` | Set story status |

---

## 13. `/retro` — Sprint Retrospective

**When**: Sprint execution is complete (or use `--force` for a partial retro). Analyzes git history, Dev Agent Records, and ADR adherence. Produces cross-sprint intelligence for the next sprint's Phase 0.

```
/retro                             # Generate retrospective
/retro --force                     # Generate even if stories are incomplete
```

Output: `.omc/sprint-plan/current/retrospective.md`

Automatically dispatches `/reconcile --all` for full-sprint code style reconciliation.

---

## 14. `/log` — Work Log Annotations

**When**: You want to record a decision, discovery, or note. Auto-detects sprint context and enriches entries with epic/story/decision refs.

```
/log "figured out auth needs refresh tokens"
/log "SSE requires custom adapter" --story=3.2 --decision=D-005
/log "decided Zustand over Redux" --tag=state-management --doc
/log "webhook retry logic" --doc=architecture/webhook-retries
```

| Flag | Effect |
|------|--------|
| `--story=N.M` | Associate with a story |
| `--epic=N` | Associate with an epic |
| `--decision=D-NNN` | Reference an architecture decision |
| `--tag=TAG` | Free-form tag (repeatable) |
| `--doc` | Also create permanent doc in `docs/` |
| `--doc=PATH` | Create doc at specific `docs/PATH.md` |

Log file: `.omc/sprint-plan/current/work-log.md` (sprint) or `.omc/work-log.md` (project).

Cross-references entries in story files. Consumed by `/retro` for retrospective context.

---

## 15. `/doc` — Permanent Documentation

**When**: You want to create lasting documentation in `docs/`. Can derive content from sprint artifacts (stories, epics, decisions) or standalone topics.

```
/doc "authentication flow"                        # From topic
/doc architecture/auth-flow                       # Explicit path
/doc api/webhooks --from-story=2.3                # From a story's implementation
/doc architecture/state --from-decision=D-007     # From a decision
/doc --from-epic=2                                # Document an entire epic
```

| Flag | Effect |
|------|--------|
| `--from-story=N.M` | Derive from story spec + implementation |
| `--from-epic=N` | Derive from all stories/decisions in epic |
| `--from-decision=D-NNN` | Document a specific architecture decision |
| `--update` | Update existing doc instead of creating new |

Docs are standalone — readable without sprint artifacts. Cross-references logged in the work log.

---

## 16. `/ultraresearch` — Multi-Agent Research Swarm

**When**: You have a question that needs broad exploration, multiple perspectives, or deep investigation. Standalone — not part of the sprint lifecycle.

```
/ultraresearch "what are the tradeoffs of SSR vs SSG for our use case?"
/ultraresearch "compare auth solutions" --depth 3 --mode engineering
```

| Flag | Effect |
|------|--------|
| `--depth 1\|2\|3` | How deep to explore (default: 2) |
| `--mode research\|engineering\|quick` | Research style |
| `--max-branches N` | Limit parallel hypothesis branches |
| `--output-dir PATH` | Custom output location |

---

## Quick Reference: When to Use What

| Situation | Skill |
|-----------|-------|
| "I have an idea, where do I start?" | `/prd` then `/sprint-plan` |
| "Planning is done, let's build" | `/sprint-exec` (next epic) or `--full-auto` (all) |
| "This epic is complex, let me prepare" | `/epic-prep --epic=N` |
| "The architecture phase feels weak" | `/ral architecture` |
| "A library we chose doesn't work" | `/replan --decision=D-NNN --reason="..."` |
| "Quick check — did this epic actually get built?" | `/verify --epic=N` |
| "Is this story actually done? (deep)" | `/audit-story --story=N.M` |
| "The agents used different naming styles" | `/reconcile --epic=N` or `--all` |
| "Review says there are issues" | `/review-fix` |
| "Where am I? What phase is this?" | `/status` |
| "I did manual work, update the status" | `/update-status --story=N.M --status=done` |
| "I want to note why I made this choice" | `/log "reason" --story=N.M` |
| "Document this for future devs" | `/doc topic --from-story=N.M` or `/log "note" --doc` |
| "Sprint is done, what did we learn?" | `/retro` |
| "I need to research a technical question" | `/ultraresearch "question"` |
