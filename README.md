# Skills

My personal AI coding agent skills, following the [Agent Skills](https://agentskills.io) standard. Layout and conventions inspired by [mattpocock/skills](https://github.com/mattpocock/skills).

## Skills

### Engineering

A prerequisite chain — install together:

- **[codebase-design](skills/engineering/codebase-design/SKILL.md)** — shared vocabulary for designing deep modules: interface, behaviour, seam, adapter, factory, resource.
- **[dependency-injection](skills/engineering/dependency-injection/SKILL.md)** — wiring modules together: code against interfaces, declare seams in factories. *Requires codebase-design.*
- **[factory-pattern](skills/engineering/factory-pattern/SKILL.md)** — the code that creates an entity satisfying a module's interface. *Requires the two above.*

Testing:

- **[testing](skills/engineering/testing/SKILL.md)** — testing modules through their factories and interfaces: one behaviour per test, test adapters at the seams. *Requires codebase-design.*
- **[test-driven-development](skills/engineering/test-driven-development/SKILL.md)** — driving development through tests: the red → green loop, one behaviour per cycle. *Requires codebase-design and testing.*

Structure:

- **[project-structure](skills/engineering/project-structure/SKILL.md)** — where code lives on disk: one package and entry point per module, shared data carriers moved at second use, imports pointing one way. *Requires codebase-design.*

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
