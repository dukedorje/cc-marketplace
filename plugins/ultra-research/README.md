# Ultra-Research

A multi-agent research swarm that decomposes questions into ranked hypotheses, fans out parallel specialist agents to explore each branch, and synthesizes findings into a referenced document with inline source citations.

## Installation

```bash
claude plugin add ./plugins/ultra-research
```

## Usage

```
/ultra-research "How does the authentication system handle token refresh?"
/ultra-research "What are the tradeoffs between SSR and CSR for our dashboard?" --mode engineering
/ultra-research "Why are API response times spiking?" --depth 1 --mode quick
```

### Parameters

| Parameter       | Default                            | Description                                    |
| --------------- | ---------------------------------- | ---------------------------------------------- |
| `question`      | *(required)*                       | The research question or goal                  |
| `--depth`       | `2`                                | Max tree depth (1=flat, 2=one sub-level, 3=deep) |
| `--mode`        | `research`                         | `research`, `engineering`, or `quick`          |
| `--max-branches`| `5`                                | Max parallel branches per level                |
| `--output-dir`  | *(context-dependent)*              | Where to write output artifacts (see below)    |

### Output Directory

The default output location depends on how ultra-research is invoked:

- **User-invoked** (`/ultra-research`) → `docs/research/<slug>` — project knowledge, committable
- **Agent-invoked** (from another workflow) → `.omc/research/<slug>` — ephemeral working data
- **Explicit `--output-dir`** → always wins

### Modes

- **research** — Produces a narrative synthesis document with cross-referenced findings, confidence assessments, and open questions. Best for understanding a topic.
- **engineering** — Produces an action plan with risk analysis and decision log. Best for deciding what to build or change.
- **quick** — Single-pass with 3 hypotheses max, no sub-branching, no verification. Best for lighter questions that need breadth but not depth.

## How It Works

1. **Decomposition** — An analyst agent breaks the question into ranked hypotheses scored by plausibility and information value.
2. **Exploration** — Specialist agents (document-specialist, architect, scientist, analyst, explore) investigate hypotheses in parallel with controlled sub-branching.
3. **Synthesis** — An architect agent cross-references all findings into a unified document with inline citations.
4. **Verification** — A verifier agent audits citations, checks coverage, and flags unsupported claims.

## Output Structure

```
docs/research/<question-slug>/
├── decomposition.md          # Ranked hypotheses
├── synthesis.md              # Final output with verification
└── hypotheses/
    ├── hypothesis-one/
    │   ├── findings.md
    │   └── sub/
    │       └── sub-hyp/
    │           └── findings.md
    └── hypothesis-two/
        └── findings.md
```

## Agent-Callable Interface

Ultra-research can be invoked programmatically by other skills and workflows. This makes it composable — e.g., sprint-plan can fire ultra-research during architecture decisions.

### Dispatch

Spawn an agent with a structured `## Ultra-Research Request` block in the prompt:

```
## Ultra-Research Request
- QUESTION: What are the tradeoffs between X and Y?
- MODE: engineering
- DEPTH: 2
- OUTPUT_DIR: .omc/sprint-plan/current/research/x-vs-y
- MAX_BRANCHES: 4
- CONTEXT: |
    Any domain context the caller already has.
    This seeds the decomposition stage.
```

### Output Contract

After completion, the caller can read `{{OUTPUT_DIR}}/summary.json` for structured results:

```json
{
  "question": "...",
  "mode": "engineering",
  "status": "complete",
  "confidence": "high",
  "executive_summary": "3-5 sentence answer",
  "key_findings": [{ "finding": "...", "confidence": "high", "sources": ["..."] }],
  "open_questions": ["..."],
  "contradictions": ["..."],
  "verification_status": "PASS"
}
```

Full narrative output is always available in `{{OUTPUT_DIR}}/synthesis.md`.

## Agents Used

| Agent                | Model  | Role                                      |
| -------------------- | ------ | ----------------------------------------- |
| `analyst`            | opus   | Decomposition, requirements analysis      |
| `document-specialist`| sonnet | External docs, web search                 |
| `architect`          | opus   | Design tradeoffs, synthesis               |
| `scientist`          | sonnet | Data analysis, benchmarking               |
| `explore`            | haiku  | Codebase search, file mapping             |
| `verifier`           | sonnet | Citation audit, coverage check            |
