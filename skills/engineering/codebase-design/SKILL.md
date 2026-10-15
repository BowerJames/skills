---
name: codebase-design
description: Shared vocabulary for designing simple, flexible, maintainable codebases. Use when the user wants to design or implement changes in the codebase.
---

# Codebase Design

Design modules that can represent a wide range of capability behind a small interface, fully testable through its interface and constructor. Use this language and these principles wherever code is being designed or restructured.

The aim is to provide:

- leverage for callers
- locality for maintainers
- flexibility for designers
- testability for everyone.

## Glossary

Use these terms exactly — don't substitute "component," "service," "API," or "boundary." Consistent language is the whole point.

**Entity** — A code construct, could be a class, function, type or any other structure that exists within the language syntax.

**Interface** — everything a caller must know to use an entity correctly: the type signature, but also invariants, ordering constraints, error modes and performance characteristics. A module's error modes include the errors its port lets propagate.

_Avoid_: API, signature (too narrow — they refer only to the type-level surface).

**Implementation** — The body of code that sits behind an interface.

**Constructor** — the one function that creates an entity. It takes everything required to measure the entity's behaviours, and it reaches no resource.

**Port** — An interface an entity defines for what it needs from outside itself: passed to its constructor or, for an entrypoint, registered with it. A module defines at most one port, and that port holds everything the module needs from outside itself.

**Module** — a package in the core with an interface, an implementation, a constructor, and at most one port. It is self-contained: it depends only on the data carriers of the domain package and on its own port. Only these packages are modules: code inside an implementation is not a module, whatever its shape.

_Avoid_: unit, component, service.

**Configuration** — Values in the constructor of an entity that alter behaviour without editing implementation. Distinct from a port, which is an interface, not a value.

**Adapter** — An entity defined to satisfy a port. A **driven adapter** satisfies a module's port: the module calls it. A **driving adapter** satisfies an entrypoint's port: the entrypoint calls it, and it drives the modules it needs through their interfaces.

**Entrypoint** — what receives input from outside the program and calls into it: a web framework, a CLI runner, a queue consumer, a scheduler, or the process's `main` where there is no framework. The project does not write it; it configures it and starts it. Its port is the shape of what it calls: a route, a command, a message callback. Driving adapters satisfy it.

A library the program calls is not an entrypoint, even though the project did not define its interface: an HTTP client, a database driver, the environment, the clock. It is how an adapter reaches a resource. What tells them apart is the direction of the call, not who defined the interface.

_Avoid_: controller (names the project's code, not the framework that calls it), entry point.

**Behaviour** — Something measurable and verifiable by a test through an entity's interface and constructor.

**Factory** — Anything that calls the constructor of an entity and returns the entity.

**Capability** — what an entity can do for a caller through its interface, counting what the adapter passed to its constructor provides. The same behaviour gives a different capability for each adapter that can satisfy the port.

**Data carrier** — a type that holds data and reaches no resource: `Order`, `Receipt`, `Currency`. It has fields and no methods, and its only invariants are the ones its types express. Validation beyond the types, and any computation over a data carrier, however small, belongs in the implementation of a module. It has no implementation behind it, so entities take and return it as its concrete type, not through an interface of its own, and it has no tests of its own.

_Avoid_: model (overloaded with ORMs and machine learning), DTO, entity (already means any code construct).

**Resource** — something external from the software's environment or a test's control: the filesystem, the network, the clock, randomness, environment variables, a database. A test cannot decide the outcome of an interaction with a resource, so behaviour that crosses one must be tested by substituting it.

**Leverage** — what callers want from a module: a higher range of capability per unit of interface they learn.

**Locality** — what maintainers want from a module: change, bugs, knowledge, and verification concentrate in one place rather than spreading across callers. Fix once, fixed everywhere.

**Flexibility** — what designers want from a module: the ease with which one can extend or broaden the range of capabilities sitting behind an interface.

## Design Considerations

When designing an interface, ask whether you can maintain the range of capabilities but:

- Reduce the number of methods
- Simplify the parameters
- Hide more complexity behind the interface
- Add sensible defaults

## When to make a module

A module is the unit that tests are written against, and a connection between two modules costs a port method, an adapter, its adapter factory, wiring in a module factory and a test adapter. So the default is not to make one.

**Default:** new behaviour goes inside an existing module's implementation. An implementation can grow to as many files as it needs without becoming a second module, and its internal code is structured however suits it.

Split behaviour out into its own module when one of these holds:

- **It needs its own test surface.** Its behaviours are worth measuring on their own, through their own constructor and interface, and not only through the module that uses it.
- **A second module needs it.** Two modules cannot share implementation code, so behaviour both need has to sit behind an interface of its own.
- **It must be swappable.** The module that uses it should work with more than one version of it. If this is the only reason, it may be an adapter for that module's port and not a module.

## Principles

- **Capability is measured against the interface, not the implementation.** A module can provide a large range of capabilities through the adapters of its port.
- **An entity's constructor is not part of its interface.** Adding complexity to the constructor of a module does not increase the complexity of the entity's interface and therefore does not reduce the leverage.
- **An entity's interface and constructor are the test surface.** Callers and tests drive through the same interface. Behaviours can be measured through the interface and constructor.
- **A module is self-contained.** It depends on nothing but the data carriers of the domain package, its own port, and the language itself: no other module, no adapter, no third-party library. Anything else it needs is a method on its port.
- **A port between two modules is a cost paid for independence.** It should buy something: a separate test surface, a second user, or the ability to swap.

## Errors

- **An error propagates by default.** Code that cannot handle an error lets it pass through unchanged.
- **An error is handled or propagated, never wrapped.** Catching an error to raise a different one in its place hides where it came from. Handling means deciding the outcome: an adapter that turns a card decline into a `false` from its port has handled it.
- **An entity defines an error only when its own implementation determines it, and then it owns it.** A module that refuses an order defines that error in its own package. An adapter that determines an error of its own defines it in the adapter's package. Neither defines errors for failures that belong to something else.

## Designing for testability

Good entities make testing natural:

1. **Constructors accept dependencies, they don't create them.**
2. **Side effects go through the port.** The implementation returns results, and anything else it causes is a call on its port, where a test adapter can observe it.
3. **Small surface area.** Fewer methods = fewer tests needed. Fewer params = simpler test setup.

## Relationships

- A **Module** has exactly one **Interface** (the surface it presents to callers and tests) and at most one **Port**.
- **Capability** is a property of an **Entity**, measured against its **Interface**.
- An entity can have many **Factories** but only one **Constructor**.
- An **Adapter**'s interface must satisfy a **Port**, and it satisfies the whole port.
- Every **Adapter** in the program satisfies the port of a **Module** (a driven adapter) or of an **Entrypoint** (a driving adapter). A test adapter is the exception: it satisfies whichever port of the entity under test a test needs, an adapter's own ports included.
- A **Port** belongs to the entity that defines it. A module has at most one. Any other entity can have its own: whatever an adapter's constructor takes as an interface is a port of it. An entrypoint's port is defined by the framework or runtime that is the entrypoint.
- In the program, what satisfies an adapter's own ports is not an adapter in its own right: a part defined in the adapter's package, an object the entrypoint hands it (a request, a socket), or an entity or factory passed to its constructor. The exception is an adapter wrapping another adapter of the same port, such as `retrying`: the inner one satisfies the same port.
- An **Adapter** is told apart from a **Module** by its role, not its shape: it exists to satisfy a port.

## Rejected framings

- **Capability as a ratio of implementation-lines to interface-lines** (Ousterhout): rewards padding the implementation. We measure capability against the interface instead: the range of capability per unit of interface a caller has to learn.
- **The entrypoint as code the project writes, satisfying no port**: it hides the framework that calls the project's code. A route handler has the shape the framework dictates, which is satisfying a port; calling it anything else leaves its role unnamed. The framework is the entrypoint, and the handler is a driving adapter.
- **The entrypoint as anything whose interface the project did not define**: it would make every library an entrypoint. An HTTP client or a database driver is called by the project, not the other way round, so it is how an adapter reaches a resource.

## Language idioms

The vocabulary above is language-agnostic. When implementing, read the file for the target language:

- [python.md](python.md) — Protocols and Callables as interfaces and ports, a worked `OrderProcessor` module with its port, an error it owns, adapters and their factories
- [typescript.md](typescript.md) — no classes: interfaces and function types as interfaces and ports, a worked `OrderProcessor` module with its port, an error it owns, adapters and their factories
