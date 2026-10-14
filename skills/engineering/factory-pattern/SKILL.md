---
name: factory-pattern
description: The code that calls a module's constructor and returns the entity. Use when deciding how a module gets built, what a factory should return or what a factory is allowed to do. Requires the codebase-design and dependency-injection skills.
---

# Prerequisites

- **[codebase-design](../codebase-design/SKILL.md)** — vocabulary used exactly: entity, module, interface, implementation, constructor, port, adapter, factory.
- **[dependency-injection](../dependency-injection/SKILL.md)** — its two rules: a constructor depends on the module's own ports, and ports and configuration go in the constructor, not the interface.

# Factory Pattern

A factory for a module is code that calls the module's constructor and returns the entity. It is never the constructor itself: the constructor accepts the ports, and the factory supplies adapters for them.

Where factories live on disk is covered by [project-structure](../project-structure/SKILL.md).

## Rules

### 1. A factory returns the interface

A factory declares its return type as the interface of the module, never the implementation.

This improves **leverage for callers** since a caller of the factory learns only the interface.

This improves **locality for maintainers** since the implementation can be renamed or replaced without touching any caller of the factory.

### 2. A factory holds no behaviour

A factory chooses adapters, constructs them and calls the module's constructor. It does nothing else. Anything worth testing belongs in a module or an adapter.

This improves **testability for everyone** since every behaviour sits behind a constructor, where a test can reach it with test adapters.

### 3. One constructor, many factories

A module has a single constructor. Each factory is one way of supplying it: one per choice of adapters, or one per environment. A new way of building the module is a new factory, never a second constructor and never a change to the interface.

This improves **flexibility for designers** since a new combination of adapters is added without touching the module.

### 4. A factory gets another module's entity from that module's factory

A factory that needs an entity of another module calls one of that module's factories. It never calls another module's constructor.

This improves **locality for maintainers** since the knowledge of how a module is built stays with that module's factories.

### 5. A factory is where resources are first reached

A factory may reach resources directly: read an environment variable, read a settings file, open a connection. It passes what it reads to constructors, as configuration or as constructed adapters.

A factory may take parameters that select between choices, such as which adapter to use. It never takes a port or an adapter as a parameter: code that takes ports is a constructor.

This improves **locality for maintainers** since the places where the program touches its environment are the factories and the adapters they construct, and nowhere else.

This improves **testability for everyone** since no constructor reaches a resource, so every behaviour can be measured without one.

## Language idioms

The rules above are language-agnostic. When implementing, read the file for the target language:

- [python.md](python.md) — a `create_order_processor` factory with its return type declared wrong, then right, and worked examples of the other rules
- [typescript.md](typescript.md) — a `createOrderProcessor` factory with its return type left inferred, then declared, and worked examples of the other rules
