# Skills

My personal AI coding agent skills, following the [Agent Skills](https://agentskills.io) standard. Layout and conventions inspired by [mattpocock/skills](https://github.com/mattpocock/skills).

## Skills

### Engineering

A prerequisite chain — install all three together:

- **[codebase-design](skills/engineering/codebase-design/SKILL.md)** — shared vocabulary for designing deep modules: interface, seam, adapter, resource.
- **[dependency-injection](skills/engineering/dependency-injection/SKILL.md)** — wiring modules together: code against interfaces, declare seams at construction. *Requires codebase-design.*
- **[factory-pattern](skills/engineering/factory-pattern/SKILL.md)** — construction wrapped behind an interface. *Requires the two above.*

Standalone:

- **[testing](skills/engineering/testing/SKILL.md)** — testing modules through their interfaces: behaviour, seams, test adapters, red → green. *Requires codebase-design.*

### Utils

- **[repo-explorer](skills/utils/repo-explorer/SKILL.md)** — explore codebases without cluttering the workspace, via a `/tmp/repos/` cache.

## Installation

### skills.sh (recommended, works with most agents)

Interactive — pick skills and agents:

```bash
npx skills@latest add BowerJames/skills
```

Or explicit:

```bash
npx skills@latest add BowerJames/skills --skill '*' --agent pi   # all skills, one agent
npx skills@latest add BowerJames/skills --all                    # all skills, all agents
npx skills@latest add BowerJames/skills --skill repo-explorer --agent claude-code
```

Installs are project-level by default and recorded in `skills-lock.json` (restore with `npx skills experimental_install`). Add `-g` for a global (user-level) install. Update later with `npx skills update`.

### Manual

```bash
git clone https://github.com/BowerJames/skills.git
cp -r skills/engineering/* skills/utils/* ~/.pi/agent/skills/    # pi
```

| Agent | Skills directory |
|---|---|
| pi | `~/.pi/agent/skills/` or `~/.agents/skills/` (global), `.pi/skills/` or `.agents/skills/` (project) |
| Claude Code | `~/.claude/skills/` (global), `.claude/skills/` (project) |
| Codex | `~/.codex/skills/` (global) |
