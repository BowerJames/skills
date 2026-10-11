---
name: factory-pattern
description: Construction wrapped behind an interface. Use when creation needs to vary (which adapter, which configuration, per-request data), when binding runtime data to dependencies, or when deciding where a module gets built. Requires the codebase-design and dependency-injection skills.
---

# Prerequisites

- **[codebase-design](../codebase-design/SKILL.md)** — vocabulary used exactly: module, interface, seam, adapter, resource.
- **[dependency-injection](../dependency-injection/SKILL.md)** — construction, and seams declared at construction.

# Factory Pattern

A factory for a module is code that creates and returns an entity satisfying the interface of the module

There can be many factories for a module. The factory must declare its return type as the interface of the module and not a concrete implementation of the module.


## Language idioms

The rules above are language-agnostic. When implementing, read the file for the target language:

- [python.md](python.md) — `create_x` factory functions, `Callable`-typed factories bound with `functools.partial` or closures, lambda fakes in tests
- [typescript.md](typescript.md) — `createX` factory functions, function-typed factories bound with arrow closures, arrow fakes in tests
