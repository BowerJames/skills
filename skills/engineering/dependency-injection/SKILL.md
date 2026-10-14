---
name: dependency-injection
description: Discipline for what a constructor receives. Covers depending on a module's own ports and passing ports and configuration to the constructor. Use when wiring dependencies, deciding whether a dependency belongs in a module's constructor or its interface, or fixing modules that create or fetch what they need. Requires the codebase-design skill.
---

# Prerequisites

- **[codebase-design](../codebase-design/SKILL.md)** — read it first. This skill uses its vocabulary (entity, module, interface, behaviour, implementation, constructor, port, configuration, adapter, factory, resource) exactly.

# Dependency Injection

Dependency injection is passing an entity its dependencies instead of letting it create or fetch them: a module receives **Adapters** for its **Ports** through its **Constructor** rather than reaching for **Resources** itself. The aim is that every resource sits behind a port, and every port is declared in the constructor, so the **Interface** carries only data.

## Rules

### 1. A constructor depends on the module's own ports

Every parameter of a module's constructor that carries behaviour is a port: an interface the module declares itself.

- **A port is an interface, never an implementation.** A constructor never names an adapter's type.
- **A port belongs to the module that declares it.** A constructor never names another module's interface. Two modules that need the same thing each declare their own port. Their adapters can be thin wrappers around the same logic: the adapter decides how each module uses it.
- **A port holds only what the module calls.** It is shaped by the module's needs, not copied from the adapter that will satisfy it.

Data carriers and configuration have no implementation to swap, so they are passed as their concrete types. An object from a library that reaches no resource is configuration, not a port.

An adapter's constructor is not limited to ports: it can take other adapters, or anything else it needs. Like every constructor, it must take everything required to measure the adapter's behaviours.

This improves **flexibility for designers** since a new adapter adds a capability without touching the module.

This improves **locality for maintainers** since a port changes with its module alone, and a change to an adapter or to another module stops at the adapter.

This improves **testability for everyone** since test adapters can be passed to any constructor to observe and drive behaviours.

### 2. Ports and configuration go in the constructor, not the interface

Anything that changes the behaviour of an entity is passed to its constructor: as a port if it is an interface, as configuration if it is a value.

- **The interface takes only data carriers.** A method on the interface never takes a port or a configuration value.
- **A constructor accepts what it needs. It never creates it and never fetches it.** The implementation does not construct an adapter, read an environment variable, read the clock or open a connection. Each of those arrives through the constructor.

This improves **leverage for callers** since the interface for the module stays simpler, increasing the capability per unit of complexity in the interface.

This improves **locality for maintainers** since the behaviour of an entity is decided in one place, where it is constructed.

This improves **testability for everyone** since every behaviour can be measured from the constructor arguments and the interface alone. It also allows tests to be grouped by constructor setup.

## Language idioms

The rules above are language-agnostic. When implementing, read the file for the target language:

- [python.md](python.md) — worked examples of each rule, built on `OrderProcessor`
- [typescript.md](typescript.md) — worked examples of each rule, built on `OrderProcessor`
