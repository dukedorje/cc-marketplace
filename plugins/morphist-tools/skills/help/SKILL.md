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
  /prd           ──>  /sprint-plan ──>  /refine      ──>  /sprint-exec    ──>  /retro
                                        (optional)         │ (per epic)
                                                           ├─> /verify         (auto, inline)
                                                           ├─> /sprint-review  (auto, background)
                                                           ├─> /reconcile
                                                           └─> /review-fix

  AT ANY TIME:  /status  /refine  /audit  /replan  /scope  /log  /doc  /update-status
                /sprint-validate  /done-validate  /blocker-triage  /ultraresearch
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
/sprint-plan "your idea"                       # Pauses at decision points for review
/sprint-plan path/to/prd.md                    # From an existing PRD
/sprint-plan --sprint-size=focused "careful"   # Small sprint, max steering (3-8 stories)
/sprint-plan --sprint-size=ambitious "big"     # Full delivery mode (15-30 stories)
/sprint-plan --step "careful planning"         # Pause after EVERY phase
/sprint-plan --auto "build the dashboard"      # Run all phases without pausing
/sprint-plan --fast "quick prototype"          # Single-pass, no consensus, no pauses
/sprint-plan --continue                        # Resume from last incomplete phase
/sprint-plan --restart-from=sprint-scoping     # Re-scope the sprint boundary
```

| Flag | Effect |
|------|--------|
| *(default)* | Pause at decision points: after requirements, scoping, architecture, validation |
| `--sprint-size=SIZE` | `focused` (3-8 stories), `standard` (8-18, default), `ambitious` (15-30) |
| `--step` | Pause after every phase for maximum control |
| `--auto` | Run all phases without pausing (stops only for Decision Steering) |
| `--fast` | Single-pass mode, no consensus, no pauses |
| `--thorough` | Explicit thorough mode (default quality level) |
| `--skip-ux` | Skip the optional UX Design phase |
| `--continue` | Resume from next incomplete phase |
| `--continue=<phase>` | Resume from after specified phase |
| `--restart-from=<phase>` | Re-run a phase and all downstream |

### Phases (in order)

| # | Phase | What happens |
|---|-------|-------------|
| 0 | Discovery | Scans codebase, inventories existing code, finds planning artifacts |
| 1 | Requirements | Expands PRD into FRs, NFRs, constraints (parallel agents) |
| 1B | Sprint Scoping | Clusters FRs, proposes sprint boundary, negotiates scope with user |
| 1.5 | UX Design (optional) | Component specs, user flows for frontend projects |
| 2A | Architecture | Architecture decisions (only for in-scope FRs) |
| 2B | Epic Design | Groups in-scope requirements into user-value-focused epics |
| 3 | Stories | Breaks epics into dev-agent-sized stories with BDD criteria |
| 4 | Enrichment | Adds technical details, testing requirements, file lists per story |
| 5 | Validation | Checks FR coverage, dependencies, story quality, sprint size compliance |

### Output

Spec artifacts in `SPEC_DIR` (e.g. `docs/sprints/NNN-slug/`):

| File | Contents |
|------|----------|
| `discovery.md` | Project context |
| `requirements.md` | Functional and non-functional requirements |
| `ux-design.md` | Component specs and user flows (optional) |
| `architecture-decisions.md` | ADR-lite architecture decisions (D-001, D-002, ...) |
| `epics.md` | Epic structure with story summaries |
| `stories/E{N}-S{M}.md` | Enriched story files ready for dev agents |
| `readiness-report.md` | Final validation summary |

Ephemeral state in `STATE_DIR` (e.g. `.omc/sprint-plan/sprint-NNN/`, symlinked as `current/`):

| File | Contents |
|------|----------|
| `phase-state.json` | Phase tracking, epic/story statuses, execution log |
| `work-log.md` | Work log annotations |
| `reviews/` | Review and reconciliation reports |

---

## 2b. `/scope` — Sprint Scope Negotiation

**When**: You want to (re-)negotiate which FRs are in scope for this sprint. Runs automatically as Phase 1B of `/sprint-plan`, but also available standalone for mid-sprint re-scoping.

```
/scope                         # Re-negotiate sprint scope
/scope --show                  # View current IN/STRETCH/DEFER split
/scope --sprint-size=focused   # Re-scope as a focused sprint
/scope --auto                  # Auto-accept analyst's proposal
```

Clusters FRs by cohesion/dependency, estimates story counts, proposes a sprint boundary, and presents an interactive negotiation. Deferred items are added to the backlog.

---

## 2c. `/sprint-validate` — Full Sprint Validation

**When**: Sprint planning artifacts are complete (or partially complete) and you want adversarial validation. Runs automatically as Phase 5 of `/sprint-plan`, but also available standalone after manual edits.

```
/sprint-validate               # Validate current sprint artifacts
/sprint-validate --fix         # Validate and auto-fix issues (max 2 iterations)
/sprint-validate --force       # Validate even if enrichment is incomplete
```

Dispatches `critic` (opus) + `verifier` (sonnet) in parallel. Checks FR coverage, architecture compliance, dependency validation, story quality, epic health, and mechanical correctness. Produces a readiness report.

For a quick post-execution smoke test, use `/verify` instead.

---

## 3. `/refine` — Refine Artifacts & Epic Deep-Dive

**When**: A phase artifact needs improvement (consensus pass), OR you want to deep-dive into an epic before execution (interactive preparation).

### Consensus mode (refine a phase artifact)

```
/refine architecture               # Refine architecture decisions
/refine requirements                # Refine requirements
/refine sprint-scoping              # Re-evaluate sprint boundary
/refine epics --epic=2              # Refine Epic 2's design
/refine enrichment --story=3.1      # Refine a single story's spec
/refine prd                         # Refine the PRD
/refine retro                       # Refine the retrospective
```

Runs a Planner → Architect → Critic adversarial consensus pass. Each scope gets 2 passes before diminishing-returns warning. If the artifact changes, downstream phases are marked stale.

### Interactive deep-dive (pre-execution preparation)

```
/refine --epic=3                   # Deep dive into Epic 3
/refine --story=3.2                # Drill into a specific story
/refine --epic=3 --graph           # View decision graph for Epic 3
/refine --epic=3 --propagate       # Deep dive + auto-reconcile ADR changes
```

Interactive session for enriching stories, revising decisions, and drilling into specifics. Re-runnable — previous prep notes are preserved.

| Flag | Effect |
|------|--------|
| `<phase>` | Run consensus on that phase's artifact |
| `--epic=N` | Scope to epic N (with phase: scoped consensus; without: interactive deep-dive) |
| `--story=N.M` | Scope to story N.M |
| `--graph` | Display the decision dependency graph |
| `--propagate` | Auto-run reconcile after ADR changes |
| `--force` | Bypass the 2-pass-per-scope limit |

Valid phases: `requirements`, `sprint-scoping`, `architecture`, `epics`, `stories`, `enrichment`, `prd`, `retro`

---

## 4. `/sprint-exec` — Execute Stories

**When**: Planning is complete and you're ready to implement. This is where code gets written.

Sprint-exec is a **thin dispatcher** — it reads story specs, builds a task manifest, and delegates to OMC execution primitives. For maximum autonomy, run under ralph: `ralph /sprint-exec --auto`.

```
/sprint-exec                       # Execute NEXT incomplete epic (stops on high+ severity)
/sprint-exec --auto                # All remaining epics, stops on high+ severity
/sprint-exec --auto --stop-at=critical  # All remaining, only stops on critical
/sprint-exec --full-auto           # All remaining, never stops
/sprint-exec --next-story          # Just the next unfinished story
/sprint-exec --epic=2              # Only Epic 2
/sprint-exec --story=2.3           # Only story 2.3 (uses opus for retries)
/sprint-exec --dry-run             # Preview execution plan
/sprint-exec --concurrency=3       # Limit to 3 parallel agents per epic
```

| Flag | Effect |
|------|--------|
| *(no flags)* | Execute the next incomplete epic, stop on high+ severity |
| `--auto` | All remaining epics, stop on high+ severity decisions |
| `--full-auto` | All remaining epics, auto-resolve everything |
| `--stop-at=LEVEL` | Override stop threshold: `critical`, `high`, `medium`, `all` |
| `--next-story` | Execute just the next unfinished story |
| `--epic=N` | Execute only epic N |
| `--story=N.M` | Execute only story N.M (uses opus model for retries) |
| `--dry-run` | Show plan without dispatching agents |
| `--concurrency=N` | Max parallel executor agents per epic |

**Behavior**:
- **Default is incremental**: just the next epic, stops on high+ severity
- `--auto` runs all epics, stops on high+ severity (arch blockers, epic failures)
- `--full-auto` runs everything, auto-resolves all decisions
- `--stop-at` overrides the threshold for any mode (e.g., `--stop-at=all` for maximum control)
- Epics run sequentially, stories within an epic run in parallel
- After each epic: `/done-validate` → `/blocker-triage` (if needed) → `/verify` (gate) → `/sprint-review` (background)
- Context checkpoint after each epic for seamless resume
- Failed stories can be retried individually with `--story=N.M` (upgrades to opus)

---

## 5. `/sprint-review` — Review Completed Work

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

Output lands in `STATE_DIR/reviews/`.

---

## 6. `/reconcile` — Fix Style Drift Across Agents

**When**: Parallel agent execution produced inconsistent naming, patterns, or conventions across stories/epics. Also use after `/refine` revises ADRs to propagate changes (or use `/refine --propagate` to auto-trigger).

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

## 7. `/review-fix` — Fix Issues from Reviews

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

## 8. `/verify` — Quick Epic Completion Check

**When**: After an epic executes, you want a fast independent check that files exist, ACs were addressed, and architecture was followed. Auto-runs between epics in `/sprint-exec`. For deep analysis, use `/audit` instead.

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

## 8b. `/done-validate` — Post-Execution Completion Validation

**When**: After executor agents return, validates they actually produced work. Auto-runs within `/sprint-exec`, but also available standalone.

```
/done-validate --epic=2            # Validate all stories in Epic 2
/done-validate --story=2.3         # Validate a single story
/done-validate --story=2.3 --update  # Validate and write results to frontmatter
```

Checks: Dev Agent Record filled, files exist on disk, AC checkboxes addressed, blocker classification. Determines outcome: `done` | `failed` | `blocked`.

---

## 8c. `/blocker-triage` — Architectural Blocker Resolution

**When**: A story reports an architectural blocker (`library_incompatible`, `architecture_mismatch`, `dependency_missing`). Auto-runs within `/sprint-exec`, but also available standalone.

```
/blocker-triage --epic=2           # Triage blockers in Epic 2
/blocker-triage --story=2.3        # Triage a specific story
/blocker-triage --epic=2 --auto-accept  # Accept partial for all
```

Analyzes downstream impact on upcoming epics, presents 4 options: swap approach, update architecture decision, accept partial, halt sprint.

---

## 9. `/audit` — Deep Story Investigation & Fix Planning

**When**: Something went wrong — a story failed, code shifted, a library changed, a story was blocked. You need to understand the real state of the implementation against its ACs and get a concrete plan to fix it.

```
/audit --story=3.2                 # Investigate a specific story
/audit --epic=2                    # Investigate all stories in Epic 2
/audit --all                       # Investigate everything
/audit --all --context="switched from Eden Treaty to ky"
/audit --story=3.2 --tdd          # Generate failing tests as gates
/audit --dry-run                   # Preview audit plan
```

| Flag | Effect |
|------|--------|
| `--story=N.M` | Investigate specific story |
| `--epic=N` | Investigate all stories in epic N |
| `--all` | Investigate all stories |
| `--context="..."` | New facts to check against (library changes, etc.) |
| `--tdd` | Generate failing tests as validation gates |
| `--dry-run` | Preview without making changes |

Reads every implementation file, checks each AC (MET/PARTIAL/NOT_MET), produces a gap analysis and actionable fix plan. Updates story metadata so the next implementor (or `/sprint-exec --story=N.M` retry) knows what's left.

---

## 10. `/replan` — Mid-Sprint Course Correction

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

## 11. `/status` / `/update-status` — View/Update Statuses

**When**: You want to see where things stand — what phase you're in, what artifacts exist, epic/story progress. Also use to fix statuses after manual work.

`/status` is a quick alias for `/update-status --show`.

```
/status                            # Sprint overview + dashboard (current sprint)
/status --sprint=2                 # Status for a specific sprint
/update-status                     # Same as /status (default is --show)
/update-status --show              # Same as above
/update-status --show --sprint=2   # Show a specific sprint
/update-status --sync              # Recalculate epic statuses from stories
/update-status --epic=2 --status=done
/update-status --story=3.1 --status=done
```

| Flag | Effect |
|------|--------|
| `--show` | Display status dashboard with decision graph (default) |
| `--sprint=ID` | Target a specific sprint instead of current |
| `--sync` | Recalculate epic statuses from their story statuses |
| `--epic=N --status=VALUE` | Set epic status |
| `--story=N.M --status=VALUE` | Set story status |

---

## 12. `/retro` — Sprint Retrospective

**When**: Sprint execution is complete (or use `--force` for a partial retro). Analyzes git history, Dev Agent Records, and ADR adherence. Produces cross-sprint intelligence for the next sprint's Phase 0.

```
/retro                             # Generate retrospective
/retro --force                     # Generate even if stories are incomplete
```

Output: `SPEC_DIR/retrospective.md`

Automatically dispatches `/reconcile --all` for full-sprint code style reconciliation.

---

## 13. `/log` — Work Log Annotations

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

Log file: `STATE_DIR/work-log.md` (sprint) or `.omc/work-log.md` (project).

Cross-references entries in story files. Consumed by `/retro` for retrospective context.

---

## 14. `/doc` — Permanent Documentation

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

## 15. `/release` — Project Release Orchestrator

**When**: You're ready to publish a release. Reads `.release.json` from the repo root to know which files to bump, what validation to run, and how to create the GitHub release.

```
/release minor                     # Bump minor version, full release
/release patch                     # Bump patch version
/release 2.0.0                     # Set exact version
/release --init                    # Create .release.json for this repo
/release --dry-run minor           # Preview without making changes
/release --notes-only              # Generate release notes only
/release --no-push patch           # Local only, don't push or create GH release
```

| Flag | Effect |
|------|--------|
| `patch` / `minor` / `major` | Semver bump level |
| `<version>` | Exact version |
| `--init` | Create or update `.release.json` |
| `--dry-run` | Show what would happen |
| `--notes-only` | Generate release notes only |
| `--no-push` | Don't push or create GitHub release |

Steps: pre-flight checks → generate release notes → bump versions → validate → commit & tag → push → GitHub release → post-release hooks.

Each repo defines its own process in `.release.json` (version files, validation scripts, CI triggers, release note format).

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
| "This epic is complex, let me prepare" | `/refine --epic=N` |
| "What's in scope for this sprint?" | `/scope --show` |
| "Let me re-scope to a smaller sprint" | `/scope --sprint-size=focused` |
| "Re-validate planning after edits" | `/sprint-validate` or `/sprint-validate --fix` |
| "The architecture phase feels weak" | `/refine architecture` |
| "A library we chose doesn't work" | `/replan --decision=D-NNN --reason="..."` |
| "Quick check — did this epic actually get built?" | `/verify --epic=N` |
| "Did the executor actually produce files?" | `/done-validate --epic=N` |
| "A story hit an architecture blocker" | `/blocker-triage --story=N.M` |
| "Something's wrong with a story, what's the real state?" | `/audit --story=N.M` |
| "A library changed, what broke?" | `/audit --all --context="library X changed"` |
| "The agents used different naming styles" | `/reconcile --epic=N` or `--all` |
| "Review says there are issues" | `/review-fix` |
| "Where am I? What phase is this?" | `/status` |
| "I did manual work, update the status" | `/update-status --story=N.M --status=done` |
| "I want to note why I made this choice" | `/log "reason" --story=N.M` |
| "Document this for future devs" | `/doc topic --from-story=N.M` or `/log "note" --doc` |
| "Sprint is done, what did we learn?" | `/retro` |
| "I need to research a technical question" | `/ultraresearch "question"` |
| "What follow-ups are piling up?" | `/backlog --show` or `/backlog --scan` |

---

## Workflows: When Things Go Wrong

When a story fails, gets blocked, or the codebase shifts, here's the natural sequence:

### Verify → Audit → Fix → Post-Mortem

```
/verify --epic=3                   # Quick: did it work?
  ↓ (FAIL or CONCERNS)
/audit --epic=3                    # Deep: what's broken and how to fix it?
  ↓ (produces fix plan)
/sprint-exec --story=3.4           # Re-execute with the fix plan
  ↓ (done)
/post-mortem --story=3.4           # Document what went wrong for future agents
/backlog --scan --story=3.4        # Catch any TODOs or workarounds left behind
```

### When the Problem Is Bigger

```
/audit --story=3.4 --context="we switched to ky"
  ↓ (reveals architecture decision is broken)
/replan --decision=D-005 --reason="Eden Treaty doesn't support SSE"
  ↓ (updates specs, resets affected stories)
/sprint-exec --epic=3              # Re-execute affected stories
/post-mortem --epic=3              # Document the whole incident
```

### Quick Reference

| Tool | Question | Speed |
|------|----------|-------|
| `/done-validate` | "Did the agent actually produce files?" (completion check) | ~10s |
| `/verify` | "Is it done?" (pass/fail gate) | ~30s |
| `/blocker-triage` | "An arch decision broke — what's the impact?" (triage) | ~1min |
| `/audit` | "What's broken and how do I fix it?" (investigation) | ~3min |
| `/post-mortem` | "Why did it fail? What should future agents know?" (lessons) | ~2min |
