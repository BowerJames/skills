# Skills

My personal AI coding agent skills, following the [Agent Skills](https://agentskills.io) standard. Layout and conventions inspired by [mattpocock/skills](https://github.com/mattpocock/skills).

## Skills

### Engineering

- **[codebase-design](skills/engineering/codebase-design/SKILL.md)** — designing a program as an assembly: components with one interface and a port per dependency, adapters joining them, contracts, foreign components met only by inbound and outbound edge components, views, entrypoints, factories, composites, and how errors cross an interface. One vocabulary, with a file for each topic built on it:
  - **[dependency-injection](skills/engineering/codebase-design/dependency-injection.md)** — how entities get their dependencies: a constructor takes a component's ports and configuration and reaches no resource; factories and composites only wire, return the interface, and reach no resource.
  - **[testing](skills/engineering/codebase-design/testing.md)** — testing entities through their constructors and interfaces: one behaviour per test, a fake for every port, foreign components through their own test hooks, unit tests by default.
  - **[test-driven-development](skills/engineering/codebase-design/test-driven-development.md)** — driving development through tests: the red → green → refactor loop, one behaviour per cycle.
  - **[project-structure](skills/engineering/codebase-design/project-structure.md)** — where code lives on disk: components, adapters under the port or contract they satisfy, promoted contracts, factories and entrypoints each in their own directory, imports pointing inward, tests mirroring the source tree, legacy code met as a foreign component and moved only along the path of a task.

  Python and TypeScript idioms for every topic live in `python/` and `typescript/` beside them.

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
