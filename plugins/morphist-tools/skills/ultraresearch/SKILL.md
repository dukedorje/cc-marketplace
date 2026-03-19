---
name: ultraresearch
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
1. Use `decomposer` to generate 3 hypotheses max
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

Spawn a `decomposer` agent (opus):

```
Agent(
  subagent_type="morphist-tools:decomposer",
  model="opus",
  prompt="""
  Research question: {{QUESTION}}
  MAX_BRANCHES: {{MAX_BRANCHES}}
  {{#if CONTEXT}}
  ## Caller Context
  {{CONTEXT}}
  {{/if}}

  Decompose this question into ranked hypotheses following your protocol.
  Write the output as a single markdown document — I will save it to the output directory.
  """
)
```

Save the agent's output to `{{OUTPUT_DIR}}/decomposition.md`.

**Failure handling**: If the decomposer fails or returns empty, retry once with a
simplified prompt. If it fails again, report the error and stop gracefully.

### Stage 2 — Exploration (parallel, depth-bounded)

For each selected hypothesis from decomposition.md, spawn an `investigator` agent
**in parallel** (respecting MAX_AGENTS cap — queue excess and launch as slots free):

```
# For each hypothesis, fire in parallel:
Agent(
  subagent_type="morphist-tools:investigator",
  model="sonnet",
  prompt="""
  ## Hypothesis Assignment
  **Title**: {{hypothesis.title}}
  **Investigation Type**: {{hypothesis.investigation_type}}
  **Plausibility**: {{hypothesis.plausibility}}
  **Info Value**: {{hypothesis.information_value}}
  **Context**: {{hypothesis.rationale}}

  DEPTH_REMAINING: {{DEPTH - current_level}}
  {{#if DEPTH_REMAINING > 0}}
  You may identify up to 3 sub-hypotheses if needed. Write sub-hypothesis findings
  to separate documents and note them in your Sub-Hypotheses section.
  {{else}}
  Do NOT generate sub-hypotheses. This is the maximum depth.
  {{/if}}

  Investigate this hypothesis following your protocol. The investigation_type tells
  you whether to focus on web search, codebase exploration, analysis, or a hybrid.

  Write your output as a single markdown document — I will save it.
  """
)
```

Save each agent's output to `{{OUTPUT_DIR}}/hypotheses/{{hypothesis-slug}}/findings.md`.

If sub-hypotheses are generated and DEPTH allows, spawn additional investigator agents
for each sub-hypothesis. Save sub-findings to
`{{OUTPUT_DIR}}/hypotheses/{{parent-slug}}/sub/{{child-slug}}/findings.md`.

**Time-box**: If an investigator has not produced findings after exploring 4+ sources,
it must write what it has and move on — do not block the swarm.

### Stage 3 — Synthesis & Verification

Spawn a `synthesizer` agent (opus). Pass it the decomposition and all findings:

```
Agent(
  subagent_type="morphist-tools:synthesizer",
  model="opus",
  prompt="""
  ## Synthesis Task
  **Original Question**: {{QUESTION}}
  **Mode**: {{MODE}}
  **Output Directory**: {{OUTPUT_DIR}}

  ## Decomposition
  {{contents of decomposition.md}}

  ## Findings
  {{for each hypothesis, include the full findings.md content with its path}}

  Follow your synthesis protocol:
  1. Cross-reference all findings by theme
  2. Draft the narrative synthesis with inline citations
  3. Run verification checks (citation audit, coverage, claim validation)
  4. Write TWO files:
     - synthesis.md (narrative output with verification section)
     - summary.json (machine-readable summary)
  """
)
```

The synthesizer handles both synthesis AND verification in a single pass.
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

At the end of Stage 3 (Synthesis), the synthesizer agent MUST also write `summary.json`
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
The skill uses morphist-tools:decomposer, morphist-tools:investigator, and
morphist-tools:synthesizer agents. Orchestrate them as described.
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
