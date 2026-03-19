---
name: ultra-research
description: >
  Multi-agent research swarm with tree-structured hypothesis exploration and synthesized output.
  Use when a question requires broad exploration, multiple perspectives, or deep investigation.
  Triggers on: ultraresearch, research swarm, deep research, explore hypotheses.
user-invocable: true
argument-hint: "<question> [--depth 1|2|3] [--mode research|engineering|quick] [--max-branches N] [--output-dir PATH]"
model: opus
---

# Ultra-Research: Tree-Structured Research Swarm

A multi-agent research orchestrator. Decomposes a question into ranked hypotheses,
fans out parallel specialist agents to explore each branch, allows controlled sub-branching,
then synthesizes findings into a referenced document with inline source citations.

## Parameters

Parse these from the user's input. Apply defaults for anything not specified.

| Parameter        | Default                                    | Description                                    |
| ---------------- | ------------------------------------------ | ---------------------------------------------- |
| `QUESTION`       | *(required)*                               | The research question or goal                  |
| `DEPTH`          | `2`                                        | Max tree depth (1 = flat, 2 = one sub-level, 3 = deep) |
| `MODE`           | `research`                                 | `research` (synthesis doc), `engineering` (action plan), `quick` (single-pass summary) |
| `OUTPUT_DIR`     | *(context-dependent, see below)*           | Root directory for all output artifacts        |
| `MAX_BRANCHES`   | `5`                                        | Max parallel hypothesis branches per level     |
| `MAX_AGENTS`     | `8`                                        | Hard cap on total concurrent agents            |

### Output Directory Resolution

The default `OUTPUT_DIR` depends on how ultra-research was invoked:

| Invocation | Default OUTPUT_DIR | Rationale |
| ---------- | ------------------ | --------- |
| **User** (`/ultra-research "question"`) | `docs/research/{{slug(QUESTION)}}` | User-initiated research is project knowledge — visible, committable |
| **Agent** (prompt contains `## Ultra-Research Request`) | `.omc/research/{{slug(QUESTION)}}` | Workflow supporting data — ephemeral, not part of the repo |
| **Explicit** (`--output-dir` or `OUTPUT_DIR` set) | Whatever was specified | Caller knows best — always wins |

**Detection rule**: If the prompt contains `## Ultra-Research Request`, treat this as an agent invocation. Otherwise, treat it as user-invoked.

### Quick Mode

When `MODE = quick`, collapse the workflow to a single pass:
1. Use `analyst` to decompose into 3 hypotheses max
2. Explore each with a single agent (no sub-branching, DEPTH forced to 1)
3. Produce a condensed synthesis (skip verification stage)

This is appropriate for questions that need breadth but not depth.

## Progress Tracking

Use `notepad_write_working` at each stage transition to persist progress:

```
## Ultra-Research: {{QUESTION}}
Stage: [1-Decomposition | 2-Exploration | 3-Synthesis | 4-Verification]
Hypotheses: N total, N explored, N pending
Output: {{OUTPUT_DIR}}
```

This ensures state survives context compaction. On resume, read notepad first
and skip completed stages.

## Workflow

### Stage 1 — Decomposition

Spawn an `analyst` agent (opus):

1. Parse QUESTION into atomic sub-questions
2. Generate N candidate hypotheses — err toward inclusion over exclusion
3. For each hypothesis, assign:
   - **plausibility**: low / medium / high
   - **information_value**: what we learn if true vs. false (low / medium / high)
   - **agent_type**: which specialist is best suited (see agent routing below)
   - **estimated_effort**: light / medium / heavy
4. Rank by `information_value × plausibility`, but **do not discard** low-plausibility
   hypotheses with high information value — these are often the most important
5. Select top MAX_BRANCHES hypotheses
6. Write `{{OUTPUT_DIR}}/decomposition.md`:

```markdown
# Decomposition: {{QUESTION}}

## Sub-Questions
1. [atomic question]
2. [atomic question]
...

## Hypotheses (ranked)

### H1: [hypothesis title]
- **Plausibility**: high | **Info Value**: high | **Agent**: document-specialist
- **Rationale**: Why this hypothesis matters
- **What we learn if true**: ...
- **What we learn if false**: ...

### H2: ...
```

**Failure handling**: If the analyst fails or returns empty, retry once with a
simplified prompt. If it fails again, report the error and stop gracefully.

### Stage 2 — Exploration (parallel, depth-bounded)

For each selected hypothesis, spawn the appropriate agent **in parallel**
(respecting MAX_AGENTS cap — queue excess and launch as slots free):

#### Agent Routing

| Agent Type             | Model  | Use When                                           |
| ---------------------- | ------ | -------------------------------------------------- |
| `document-specialist`  | sonnet | External docs, web search, reference lookup         |
| `architect`            | opus   | System design implications, tradeoff analysis       |
| `scientist`            | sonnet | Data analysis, experimentation, benchmarking        |
| `analyst`              | opus   | Requirements, constraints, hidden assumptions       |
| `explore`              | haiku  | Codebase search, symbol/file mapping                |

#### Each Agent Must:

1. Investigate its assigned hypothesis thoroughly
2. Write findings to `{{OUTPUT_DIR}}/hypotheses/{{hypothesis-slug}}/findings.md`
3. **Sub-branching** (only if DEPTH > current level):
   - Agent may identify at most 3 sub-hypotheses that warrant deeper investigation
   - Each sub-hypothesis must include a one-sentence justification for why it can't
     be resolved from current findings
   - Children write to `{{OUTPUT_DIR}}/hypotheses/{{parent-slug}}/sub/{{child-slug}}/findings.md`
   - Sub-branches do NOT spawn further children (hard stop at DEPTH)
4. **Time-box**: If an agent has not produced findings after exploring 3+ sources,
   it must write what it has and move on — do not block the swarm

#### Findings Document Format

Every `findings.md` MUST include these sections:

```markdown
# Hypothesis: {{title}}

## Summary
2-3 sentence verdict. Was the hypothesis supported, refuted, or inconclusive?

## Evidence
What was found, with specifics. Include code snippets, data points, or quotes
where applicable.

## Confidence
low | medium | high — with one sentence justifying the confidence level.

## Sources
Each source must be typed:

- [1] **file**: `src/auth/middleware.ts:45-67` — "relevant excerpt"
- [2] **url**: https://docs.example.com/auth — "relevant excerpt or summary"
- [3] **doc**: hypotheses/sibling-slug/findings.md §Evidence — "cross-reference"

## Open Questions
What remains unknown or needs human judgment.

## Sub-Hypotheses (if any)
- [child-slug]: one-sentence description and justification
```

### Stage 3 — Synthesis

Spawn an `architect` agent (opus):

1. Read ALL findings docs from the tree (walk `{{OUTPUT_DIR}}/hypotheses/` recursively)
2. Cross-reference and identify:
   - **Convergent findings**: multiple hypotheses point the same direction
   - **Contradictions**: flag explicitly for human attention
   - **Gaps**: important questions that no hypothesis addressed
   - **Surprise findings**: unexpected results with high signal value
3. Produce `{{OUTPUT_DIR}}/synthesis.md` in the appropriate format:

#### Research Mode Output

```markdown
# {{QUESTION}}

## Executive Summary
3-5 sentence answer to the original question with confidence level.

## Key Findings
Narrative synthesis with inline citations: [1], [2], ...
Organize by theme, not by hypothesis. Each finding should draw from
multiple sources where possible.

## Analysis
Cross-cutting themes, contradictions, and confidence assessment.
Explicitly state what the evidence supports vs. what is speculative.

## Open Questions
What remains unresolved. Prioritize by impact.

## Methodology
Brief description of how many hypotheses explored, agents used, depth reached.

## References
[1] hypotheses/foo/findings.md §Evidence — "relevant quote or summary"
[2] hypotheses/bar/sub/baz/findings.md §Summary — "relevant quote"
[3] https://docs.example.com/page — "external source summary"
...
```

#### Engineering Mode Output

```markdown
# {{QUESTION}}

## Recommendation
Clear directional recommendation with rationale and confidence level.

## Action Plan
1. [Step] — supported by [1], [2]
2. [Step] — supported by [3]
...
Each step must reference at least one finding.

## Risks & Mitigations
Derived from contradictions and open questions.
| Risk | Likelihood | Impact | Mitigation |
| ---- | ---------- | ------ | ---------- |

## Decision Log
Key decisions made during research and why.

## References
[Same citation format as research mode]
```

### Stage 4 — Verification

Spawn a `verifier` agent (sonnet):

1. **Citation audit**: Spot-check that every inline citation [N] in synthesis
   actually exists in the referenced source doc at the referenced section
2. **Coverage check**: Verify no hypothesis from Stage 1 was silently dropped —
   every hypothesis must appear in synthesis or be explicitly noted as unexplored
3. **Claim validation**: Flag any claims in synthesis that lack supporting citations
4. **Append** a `## Verification` section to synthesis.md:

```markdown
## Verification
- **Citations checked**: N/N valid
- **Hypotheses covered**: N/N
- **Unsupported claims**: [list or "none"]
- **Issues found**: [list or "none"]
- **Verification status**: PASS | PASS_WITH_WARNINGS | FAIL
```

If verification status is FAIL, report issues to the user rather than silently
patching — the human should decide how to proceed.

## Output Structure

```
{{OUTPUT_DIR}}/
├── decomposition.md          # Stage 1: ranked hypotheses
├── synthesis.md              # Stage 3+4: final output with verification
└── hypotheses/
    ├── hypothesis-one/
    │   ├── findings.md       # Stage 2: agent findings
    │   └── sub/
    │       └── sub-hyp/
    │           └── findings.md
    ├── hypothesis-two/
    │   └── findings.md
    └── ...
```

## Agent-Callable Interface

Ultra-research can be invoked by other skills/workflows via agent dispatch. When called
programmatically (not by a human typing `/ultra-research`), the caller provides structured
parameters and receives structured output.

### Dispatch Contract

Other skills invoke ultra-research by spawning an agent with these parameters in the prompt:

```
## Ultra-Research Request
- QUESTION: <the research question>
- MODE: research | engineering | quick
- DEPTH: 1 | 2 | 3
- OUTPUT_DIR: <caller-specified path, e.g., .omc/sprint-plan/current/research/topic-slug>
- MAX_BRANCHES: <number>
- CONTEXT: <optional — any context the caller wants to inject, e.g., existing requirements, constraints>
```

The `CONTEXT` field allows callers to seed the decomposition stage with domain knowledge
they already have, reducing redundant exploration.

### Output Contract

After completion, ultra-research guarantees these files exist at `{{OUTPUT_DIR}}`:

1. **`synthesis.md`** — the full narrative output (always present)
2. **`summary.json`** — a machine-readable summary for agent callers:

```json
{
  "question": "...",
  "mode": "research|engineering|quick",
  "status": "complete|partial|failed",
  "confidence": "low|medium|high",
  "executive_summary": "3-5 sentence answer",
  "key_findings": [
    {
      "finding": "one-sentence finding",
      "confidence": "low|medium|high",
      "sources": ["hypotheses/foo/findings.md", "https://..."]
    }
  ],
  "open_questions": ["..."],
  "contradictions": ["..."],
  "hypotheses_explored": 5,
  "hypotheses_total": 5,
  "verification_status": "PASS|PASS_WITH_WARNINGS|FAIL|SKIPPED",
  "output_dir": "..."
}
```

3. **`decomposition.md`** — the hypothesis tree (always present)
4. **`hypotheses/*/findings.md`** — individual findings (always present)

### Synthesis Stage: summary.json Generation

At the end of Stage 3 (Synthesis), the architect agent MUST also write `summary.json`
alongside `synthesis.md`. Extract the structured fields from the narrative synthesis.
This is mandatory — agent callers depend on it to decide whether to consume the full
synthesis or just the summary.

### Example: Sprint-Plan Calling Ultra-Research

```python
# In Phase 2A, when a CRITICAL architecture decision needs deeper research:
Agent(
    subagent_type="general-purpose",
    model="opus",
    prompt="""
## Ultra-Research Request
- QUESTION: What are the tradeoffs between Drizzle ORM and Prisma for a multi-tenant SaaS with row-level security?
- MODE: engineering
- DEPTH: 2
- OUTPUT_DIR: .omc/sprint-plan/current/research/orm-tradeoffs
- MAX_BRANCHES: 4
- CONTEXT: |
    Project uses PostgreSQL (D-001). Multi-tenant with row-level security (FR7).
    Existing codebase has no ORM — greenfield decision.

Follow the ultra-research workflow exactly as defined in the skill.
Write all output to the specified OUTPUT_DIR.
""",
)

# After agent completes, the caller reads:
# .omc/sprint-plan/current/research/orm-tradeoffs/summary.json
# to get structured findings, then injects them into the architecture decision.
```

---

## Error Handling

- **Agent spawn failure**: Log the error, skip that hypothesis, note it as "unexplored" in synthesis
- **Empty findings**: If an agent returns no useful findings, record "inconclusive" rather than omitting
- **Timeout**: If Stage 2 takes too long, proceed to synthesis with whatever findings are available
- **All agents fail**: Report to user with error details; do not produce a hollow synthesis. Write `summary.json` with `"status": "failed"`
