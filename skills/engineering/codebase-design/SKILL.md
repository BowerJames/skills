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

**Interface** — everything a caller must know to use an entity correctly: the type signature, but also invariants, ordering constraints, error modes, required configuration, and performance characteristics.

_Avoid_: API, signature (too narrow — they refer only to the type-level surface).

**Implementation** — The body of code that sits behind an interface.

**Constructor** — A singular thing anything must run to create an entity. Must contain everything required to measure the behaviours of the entity.

**Port** — An interface defined by an entity that must be provided to its constructor.

**Module** — Something with an interface, an implementation and that defines all the interfaces in its constructor (ports).

_Avoid_: unit, component, service.

**Configuration** — Values in the constructor of an entity that alter behaviour without editing implementation. Distinct from ports which are interfaces not values.

**Adapter** — An entity defined to satisfy a port.

**Behaviour** — Something measurable and verifiable by a test through an entity's interface and constructor.

**Factory** — Anything that calls the constructor of an entity and returns the entity.

**Capability** — what an entity can do for a caller through its interface, counting what the entities passed to its constructor provide. One behaviour gives one capability for each adapter that can satisfy a port.

**Data carrier** — a type that holds data and its invariants and reaches no resource: `Order`, `Receipt`, `Money`. It carries data, not behaviour: it has fields, it can refuse to be created with invalid data, and it has no other methods. A computation over a data carrier, however small, belongs in the implementation of a module. It has no implementation behind it, so entities take and return it as its concrete type, not through an interface of its own.

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

A module is the unit that tests are written against, and a connection between two modules costs a port, an adapter, wiring in a factory and a test adapter. So the default is not to make one.

**Default:** new behaviour goes inside an existing module's implementation. An implementation can grow to as many files as it needs without becoming a second module.

Split behaviour out into its own module when one of these holds:

- **It needs its own test surface.** Its behaviours are worth measuring on their own, through their own constructor and interface, and not only through the module that uses it.
- **A second module needs it.** Two modules cannot share implementation code, so behaviour both need has to sit behind an interface of its own.
- **It must be swappable.** The module that uses it should work with more than one version of it. If this is the only reason, it may be an adapter for that module's port and not a module.

## Principles

- **Capability is measured against the interface, not the implementation.** A module can provide a large range of capabilities through the adapters of its ports.
- **An entity's constructor is not part of its interface.** Adding complexity to the constructor of a module does not increase the complexity of the entity's interface and therefore does not reduce the leverage.
- **An entity's interface and constructor are the test surface.** Callers and tests drive through the same interface. Behaviours can be measured through the interface and constructor.
- **Behaviour is not the same as capability.** An entity's capability is measured including the capability of the entities included in its constructor.
- **A port between two modules is a cost paid for independence.** It should buy something: a separate test surface, a second user, or the ability to swap.

## Designing for testability

Good entities make testing natural:

1. **Constructors accept dependencies, they don't create them.**
2. **Return results, don't produce side effects.**
3. **Small surface area.** Fewer methods = fewer tests needed. Fewer params = simpler test setup.

## Relationships

- A **Module** has exactly one **Interface** (the surface it presents to callers and tests).
- **Capability** is a property of an **Entity**, measured against its **Interface**.
- An entity can have many **Factories** but only one **Constructor**.
- An **Adapter**'s interface must satisfy a **Port**.
- A **Port** belongs to the entity that defines it. A module has ports, and so can an adapter: whatever an adapter's constructor takes as an interface is a port of that adapter.
- An **Adapter** is told apart from a **Module** by its role, not its shape: it exists to satisfy a port.

## Rejected framings

- **Capability as a ratio of implementation-lines to interface-lines** (Ousterhout): rewards padding the implementation. We measure capability against the interface instead: the range of capability per unit of interface a caller has to learn.

## Language idioms

The vocabulary above is language-agnostic. When implementing, read the file for the target language:

- [python.md](python.md) — Protocols and Callables as interfaces and ports, a worked `OrderProcessor` module with its port, adapters and factory
- [typescript.md](typescript.md) — no classes: interfaces and function types as interfaces and ports, a worked `OrderProcessor` module with its port, adapters and factory
