---
name: dependency-injection
description: Discipline for wiring modules together. Covers coding against interfaces and declaring seams in the factory. Use when the user wants to design or improve a module's interface, find deepening opportunities, decide where a seam goes, make code more testable or AI-navigable, or when another skill needs the dependency-injection vocabulary.
---

# Prerequisites

- **[codebase-design](../codebase-design/SKILL.md)** — read it first. This skill uses its vocabulary (module, interface, implementation, seam, adapter, resource) exactly.

# Dependency Injection

Dependency injection is passing a module its dependencies instead of letting it create or fetch them: a module receives **Adapters** across its **Seams** rather than reaching for **Resources** itself. The aim is that every resource sits behind a seam, and every seam is declared in the **Interface**.

## Rules

### 1. Code against interfaces, not implementations

Where possible a modules interface and dependencies passed in factories should be interfaces not implementations.

This improves **flexibility for developers** since modules can be deepened through the creation of new adapters.

This improves **locality for maintainers** since modules can be used with a wide range of entities allowing the localistion of behaviour.

This improved **testability for everyone** as swappable mocks can be used in tests to observe and drive behaviours.

### 2. Declare seams in factories

Anything that changes the behaviour of an entity satisfying a modules interface should be passed at the construction point.

This improves **leverage for callers** since the interface for the modules stays simpler increasing the capability per unit of complexity in the interface.

This improves **locality for maintainers** since it reduces the number of location the behaviour of modules changes.

This improves **testability for everyone** since behaviours can be tested consistently based only on the factory setup. It also allows tests to be grouped by factory setup.

## Language idioms

The rules above are language-agnostic. When implementing, read the file for the target language:

- [python.md](python.md) — Protocols and Callables as dependency types, seams declared in `__init__` signatures, pytest fixtures wiring the graph, Python gotchas
- [typescript.md](typescript.md) — interfaces and function types as dependency types, seams declared in factory signatures, object-literal fakes wiring the graph, TypeScript gotchas