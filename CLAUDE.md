# Agent Plugin Bazaar

A marketplace of Claude Code plugins — skills, agents, and MCP integrations.

## Repository Structure

```
.claude-plugin/marketplace.json   # Marketplace registry (versions must match plugin.json)
plugins/
  morphist-tools/                 # Consolidated toolkit: sprint planning, PRD, research
    .claude-plugin/plugin.json
    agents/                       # Auto-discovered agent definitions
    skills/                       # All skills (sprint-plan, prd, ultra-research, etc.)
    templates/                    # Shared templates for sprint planning
validate.sh                       # Pre-flight validation script
```

## Pre-Push Checklist

Before pushing any update, ALWAYS:

1. Bump the version (default to **patch** — see below)
2. Run `./validate.sh`
3. Fix all errors before pushing

## Version Bumping

When making changes, bump the plugin version. Default to a **patch** bump unless the change warrants more:

- **Patch** (1.0.0 → 1.0.1): Bug fixes, wording changes, small adjustments
- **Minor** (1.0.1 → 1.1.0): New skills, new agents, significant feature additions
- **Major** (1.1.0 → 2.0.0): Breaking changes to skill interfaces or agent contracts

Update the version in **both** files:
1. `plugins/morphist-tools/.claude-plugin/plugin.json`
2. `.claude-plugin/marketplace.json`

Then run `./validate.sh` to confirm they match.

## Validation

```bash
./validate.sh                    # Validate all plugins
./validate.sh morphist-tools     # Validate a specific plugin
```

Checks:
- plugin.json validity and required fields
- Skill SKILL.md frontmatter (name, description)
- Agent .md frontmatter (name, description)
- Version consistency between plugin.json and marketplace.json

## Plugin Development

### Testing locally (without publishing)

```bash
claude --plugin-dir ./plugins/morphist-tools
```

This loads the plugin from source. The local copy takes precedence over any installed marketplace version. Use `/reload-plugins` inside the session to pick up changes without restarting.

### Adding a new skill

1. Create `plugins/morphist-tools/skills/<skill-name>/SKILL.md` with frontmatter
2. If the skill needs custom agents, add them to `plugins/morphist-tools/agents/`
3. Bump version and run `./validate.sh`

## Conventions

- Skills dispatch plugin agents via `subagent_type="morphist-tools:<agent-name>"`
- Agent .md files use YAML frontmatter with `name`, `description`, `model`, and optional `disallowedTools`
- Skill SKILL.md files use YAML frontmatter with `name`, `description`, and optional `user-invocable`, `argument-hint`, `model`
- OMC agent types (e.g., `oh-my-claudecode:executor`) are acceptable dependencies. Make OMC infrastructure calls (state_write, etc.) graceful — skip if unavailable.

## Skills Reference

| Skill | Description |
|-------|-------------|
| `sprint-plan` | Multi-phase sprint planning workflow |
| `prd` | Interactive PRD workshop |
| `sprint-exec` | Execute validated sprint stories |
| `ral` | RALPLAN-DR refinement pass on any phase |
| `retro` | Sprint retrospective generation |
| `ultraresearch` | Multi-agent research swarm |
| `help` | Sprint-plan usage guide |
