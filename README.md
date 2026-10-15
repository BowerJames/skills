# Skills

My personal AI coding agent skills, following the [Agent Skills](https://agentskills.io) standard. Layout and conventions inspired by [mattpocock/skills](https://github.com/mattpocock/skills).

## Skills

### Engineering

A prerequisite chain — install together:

- **[codebase-design](skills/engineering/codebase-design/SKILL.md)** — shared vocabulary for designing modules: interface, constructor, port, driving and driven adapter, entrypoint, factory, behaviour, resource, and how errors cross an interface.
- **[dependency-injection](skills/engineering/dependency-injection/SKILL.md)** — how entities get their dependencies: a constructor takes the module's one port and its configuration, and module and adapter factories only wire, reaching no resource. *Requires codebase-design.*

Testing:

- **[testing](skills/engineering/testing/SKILL.md)** — testing entities through their constructors and interfaces: one behaviour per test, a test adapter for every port, unit tests by default. *Requires codebase-design.*
- **[test-driven-development](skills/engineering/test-driven-development/SKILL.md)** — driving development through tests: the red → green → refactor loop, one behaviour per cycle. *Requires codebase-design, testing and project-structure.*

Structure:

- **[project-structure](skills/engineering/project-structure/SKILL.md)** — where code lives on disk: self-contained modules in the core, data carriers alone in the domain, driving and driven adapters, factories and startup code in the infrastructure, imports pointing inward, tests mirroring the source tree, legacy code reached through ports. *Requires codebase-design and testing.*

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
