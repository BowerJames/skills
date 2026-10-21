# Skills

My personal AI coding agent skills, following the [Agent Skills](https://agentskills.io) standard. Layout and conventions inspired by [mattpocock/skills](https://github.com/mattpocock/skills).

## Skills

### Engineering

- **[codebase-design](skills/engineering/codebase-design/SKILL.md)** — the style for code that a human can read and verify without having written it: five layers (domain, pure functions, components, wiring, entrypoints) with minimal rules, each tied to a named goal, written in ASD-STE100.
  - **[GLOSSARY](skills/engineering/codebase-design/GLOSSARY.md)** — the meaning of each term the skill uses.
  - Worked examples in **[Python](skills/engineering/codebase-design/python/codebase-design.md)** and **[TypeScript](skills/engineering/codebase-design/typescript/codebase-design.md)**, with the tools to recommend.

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
