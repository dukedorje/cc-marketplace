# Phase 1.5: UX Design (Optional)

## Purpose

Generate UX design artifacts — user flow diagrams, component hierarchy, screen specifications, and interaction patterns — from the functional requirements and discovery context. This phase bridges requirements and architecture by producing frontend design specs that inform component structure and API surface design.

This phase is **optional** and only runs when:
- `has_frontend: true` in `current/discovery.md` frontmatter
- `has_ux_artifacts: false` in `current/discovery.md` frontmatter
- `--skip-ux` flag is NOT set
- Mode is NOT `fast` (AUTONOMOUS steering skips this phase by default)

---

## Trigger Decision Point (GUIDED mode only)

Before running, present to the user:

```
## Decision Point: UX Design Phase

**What is being decided**: Whether to generate UX design artifacts before architecture
**Why it matters**: UX specs inform component architecture and API design
**Significance**: MEDIUM

### Options
1. **Run UX Design Phase** - generate component specs and user flows (~2-3 min)
2. **Skip** - proceed directly to architecture (can add UX later)

### Recommendation
Run UX Design Phase — frontend requirements were detected but no UX artifacts exist.
```

Wait for user response. If user chooses Skip (or says "2"), set `ux_design_phase: "skipped"` in `phase-state.json` and proceed to Phase 2A.

In AUTONOMOUS mode: skip this decision point and proceed directly to Phase 2A. Set `ux_design_phase: "skipped"`.

---

## Agent

| Agent | Model | Condition |
|---|---|---|
| `designer` | sonnet | Always (when phase runs) |

---

## Inputs

- `current/requirements.md` (Phase 1 output)
- `current/discovery.md` (Phase 0 output)

---

## Agent Prompt

### Designer Agent (sonnet)

```python
designer_result = Agent(
    subagent_type="oh-my-claudecode:designer",
    model="sonnet",
    prompt="""
You are the Designer agent running Phase 1.5: UX Design.

Read the following inputs:
- Requirements: .omc/sprint-plan/current/requirements.md
- Discovery: .omc/sprint-plan/current/discovery.md

From requirements.md, focus on:
- All functional requirements tagged as frontend/UI (any FR involving user interaction, display, forms, navigation)
- User personas (if present in requirements.md)

From discovery.md, focus on:
- Frontend framework and tech stack (e.g., React, Vue, Next.js)
- For brownfield: existing component patterns, design system, UI conventions noted in the codebase inventory

Your task: produce UX design artifacts in the following sections:

---

## User Flows

For each major user journey (derived from user personas and frontend FRs), produce a text-based flow diagram using mermaid or ASCII. Label each flow with the FRs it addresses.

Example format (mermaid):
```
flowchart TD
    A[Landing Page] --> B{Logged in?}
    B -- Yes --> C[Dashboard]
    B -- No --> D[Login Form]
    D --> E[Submit Credentials]
    E -- Valid --> C
    E -- Invalid --> F[Error State]
    F --> D
```

---

## Component Hierarchy

Produce a tree structure showing the component breakdown for the application. Group by page/view. Include component names, their purpose, and which FRs they implement.

Example format:
```
App
├── Layout
│   ├── Header (navigation, auth state)
│   └── Footer
├── Pages
│   ├── Dashboard (FR3, FR4)
│   │   ├── StatsPanel
│   │   ├── ActivityFeed
│   │   └── QuickActions
│   ├── Login (FR1)
│   │   ├── LoginForm
│   │   └── ErrorBanner
│   └── Settings (FR7)
│       ├── ProfileSection
│       └── PreferencesSection
└── Shared
    ├── Button
    ├── Modal
    └── LoadingSpinner
```

---

## Screen / View Specifications

For each key screen, specify:
- **Purpose**: What the user is trying to accomplish
- **Data displayed**: What information is shown (reference FRs)
- **Actions available**: What the user can do
- **States**: Normal, loading, empty, error states
- **Entry/exit points**: How users arrive at and leave this screen

---

## Interaction Patterns

Document recurring interaction patterns:
- **Forms**: Validation approach, error display, submission flow
- **Navigation**: Routing model, breadcrumbs, back navigation
- **Error states**: How errors are surfaced to the user
- **Loading states**: Skeleton screens, spinners, progressive loading
- **Confirmation dialogs**: When and how confirmations are shown

---

## Responsive & Accessibility Considerations

- Breakpoints and layout changes at each breakpoint
- Mobile-specific interaction patterns (touch targets, gestures)
- Accessibility requirements: ARIA roles, keyboard navigation, focus management, color contrast
- Any constraints from the detected tech stack (e.g., CSS framework breakpoints)

---

Return your complete structured output. Do not write any files — the orchestrator will write the output.
""",
    allowed_tools=["Read"],
)
```

---

## Output

Write the designer agent's output to `.omc/sprint-plan/current/ux-design.md` with this frontmatter:

```markdown
---
project: [name from requirements.md]
sprint: sprint-[NNN]
created: [date]
phase: 1.5-ux-design
source_requirements: .omc/sprint-plan/current/requirements.md
source_discovery: .omc/sprint-plan/current/discovery.md
---

# UX Design

[designer agent output]
```

---

## State Update

Update `current/phase-state.json`:

```json
{
  "current_phase": "ux-design",
  "ux_design_phase": "complete"
}
```

If phase was skipped (user chose Skip or AUTONOMOUS mode), set:

```json
{
  "ux_design_phase": "skipped"
}
```

Do NOT update `current_phase` when skipping — it remains `"requirements"` until Phase 2A sets it to `"architecture"`.

---

## Decision Steering

**Active — UX scope decisions.**

After the designer produces output, evaluate for significant decisions:
- Component architecture choices that affect backend API shape (e.g., "this UI requires real-time updates") → escalate to Phase 2A as a constraint
- Tech stack UX choices (e.g., "component library selection") → MEDIUM significance, present in GUIDED mode
- Accessibility requirements beyond baseline → flag as constraints for architecture

In AUTONOMOUS mode: all decisions auto-decided and logged with `[AUTO-DECIDED]`.

---

## Integration with Downstream Phases

**Phase 2A (Architecture Decisions)**:
- If `current/ux-design.md` exists, include it in architect context
- UX component hierarchy informs frontend architecture decisions (state management, routing, API surface)
- Pass relevant UX constraints as additional context when dispatching architect agent

**Phase 4 (Story Enrichment)**:
- If `current/ux-design.md` exists and a story implements a frontend FR, include the relevant component specs and screen specifications in that story's enrichment context
- Add a `## UX Specifications` section to frontend stories referencing the applicable component specs and interaction patterns

---

## Phase Completion

Phase 1.5 is complete when `current/ux-design.md` exists and `phase-state.json` has `ux_design_phase: "complete"`.

Proceed to Phase 2A: Architecture Decisions.
