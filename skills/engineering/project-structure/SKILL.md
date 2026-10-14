---
name: project-structure
description: Policy for where code lives on disk. Use when you need to create new modules, adapters, factories, entrypoints or tests, deciding where code should belong, or adding code to a codebase that does not follow this structure yet. Requires the codebase-design and testing skills.
---

# Prerequisites

- **[codebase-design](../codebase-design/SKILL.md)** — read it first. This skill uses its vocabulary (entity, module, interface, implementation, constructor, port, adapter, factory, data carrier, resource, behaviour) exactly.
- **[testing](../testing/SKILL.md)** — its vocabulary for the kinds of test and where they point: entity under test, test adapter, unit test, integration test, end-to-end test.

# Project Structure

Project structure is the mapping of entities onto files. The structure is hexagonal: modules sit in the core and know nothing about the resources they use, and everything that reaches a resource sits in the infrastructure around them.

```
src/
  core/
    domain/                          # value types shared by every module
    <module_name>/
      interface                      # the module's interface
      port                           # the module's port, if it has one
      data                           # the data carriers in its interface and port
      errors                         # the errors it determines, if any
      implementation                 # the module's implementation and constructor, in one file or many
  infrastructure/
    adapters/
      <module_name>/
        <adapter_name>/              # an adapter for that module's port
    factories/
      <module_name>/                 # the factories that wire adapters into that module
    entrypoints/                     # code that drives the modules from outside the program
tests/
  unit/                              # mirrors src/
  integration/                       # mirrors src/
  e2e/
```

## Glossary

**Core** — the `core` directory: every module and the domain package. Nothing in it reaches a resource.

**Infrastructure** — the `infrastructure` directory: the adapters, the factories and the entrypoints. Everything that reaches a resource lives here, though not everything here reaches one.

**Package** — the directory that holds the code of one module, one adapter, or the factories of one module. One module, one package; one adapter, one package; one module's factories, one package.

_Avoid_: folder (says nothing about ownership), module (already means a package in the core with an interface, an implementation and a port, not any place on disk).

**Export file** — the one file in a package that other packages import from.

_Avoid_: public API (interface already names what callers must know), barrel (names the mechanism, not the role), entry point (too close to entrypoint).

**Domain package** — the `core/domain` package, holding the value types that belong to no one module and mean the same thing in every module: `Currency`, `CustomerId`. One file per concept, named for the concept. `domain` is a reserved name: no module is called `domain`.

_Avoid_: utils, common, shared, helpers (names that say only that something is shared, so they accept anything).

**Entrypoint** — code that receives input from outside the program and drives a module through its interface: an HTTP handler, a CLI command, a queue consumer. It is not an adapter: it satisfies no port.

_Avoid_: controller, inbound adapter, driving adapter.

**Startup code** — the code that starts the program: it acquires what comes from the environment, calls the factories, and passes the entities to an entrypoint's constructor. It lives beside the entrypoint it starts.

_Avoid_: container, composition root, bootstrap.

**Factory package** — the `infrastructure/factories/<module_name>` package, holding the factories of one module: the code that chooses adapters and passes them to the module's constructor.

_Avoid_: container, composition root, bootstrap.

**Legacy code** — code in the project that does not follow this structure yet.

## Rules

### 1. Imports point inward

- The domain package imports nothing else in the project.
- A module's package imports only from the domain package. It never imports another module's package and never imports from the infrastructure.
- The infrastructure imports from the core. The core never imports from the infrastructure.

This improves **locality for maintainers** since a change to an adapter, an entrypoint or another module cannot reach into a module's package.

This improves **testability for everyone** since a module can be constructed with nothing but its own package, the domain package and a test adapter.

### 2. The core holds modules and data carriers, nothing else

- **The domain package holds data carriers only**, and only the value types that belong to no one module. Anything that carries behaviour is either a module in the core or an adapter in the infrastructure.
- **Every other package in the core is a module.** It has an interface, an implementation and at most one port. The core holds no adapters, no factories and no entrypoints.
- **A module never imports another module.** A module that needs another module adds what it needs to its own port. An adapter in `infrastructure/adapters/<module_name>` satisfies the port by calling the other module through its interface.
- **The core imports a third-party library only if the library reaches no resource internally.** A library that reaches a resource internally is imported only from the infrastructure, behind an adapter.

Whether behaviour that reaches no resource is a module in the core or an adapter in the infrastructure is a design decision. As a module it has its own interface and port, and other modules reach it through their port and an adapter. As an adapter it exists only to satisfy one module's port.

This improves **locality for maintainers** since every behaviour in the core sits behind a module's interface, and every connection between two modules is one adapter.

This improves **testability for everyone** since no module needs another module, or a resource, to be constructed in a test.

### 3. One package per module, with each part in its own file

A module's package is `core/<module_name>`. It holds:

- **interface** — one file with the module's interface, and nothing else.
- **port** — one file with the module's port. A module that needs nothing from outside itself has no port and no port file.
- **data** — one file with the data carriers that appear in the module's interface or port, unless they are in the domain package.
- **errors** — one file with the errors the module's implementation determines, if there are any.
- **implementation** — the module's implementation and its constructor. A package can organise its implementation however it wants: one file, or as many files and directories as it needs.
- **the export file** — exports the interface, the port, the data carriers, the errors and the constructor. Nothing else in the implementation is exported.

Each export has its own importers:

- **The interface, the data carriers and the errors** may be imported by anything in the infrastructure: entrypoints, factories, and adapters that connect another module to this one.
- **The port** is imported by the adapters that satisfy it, and by test adapters.
- **The constructor** is imported only by the module's factory package and by the module's unit tests. An entrypoint or an adapter never imports it.

This improves **leverage for callers** since the interface, data and errors files show everything a caller must know and nothing else.

This improves **flexibility for designers** since the port file shows everything an adapter must provide to extend the module's capability.

### 4. An adapter lives under the module whose port it satisfies

An adapter's package is `infrastructure/adapters/<module_name>/<adapter_name>`, where `<module_name>` matches the package in the core that defines the port.

An adapter satisfies the whole port. It is placed by its role, not by what it reaches: an entity defined to satisfy a module's port lives here whether or not it reaches a resource. It is named for what sets it apart from the other adapters of that port: the resource it reaches (`stripe`, `paypal`, `postgres`), or what it does instead (`in_memory`, `flat_rate`, `retrying`).

An adapter that connects one module to another is named for the module it calls: `infrastructure/adapters/checkout/order_processor`. It translates between the data carriers of the two modules.

- An adapter imports the port it satisfies, and that module's data carriers, from the module's export file, and value types from the domain package.
- An adapter that connects one module to another also imports the other module's interface and data carriers from its export file, and receives the entity through its own constructor.
- An adapter never imports another adapter. An adapter that wraps another adapter of the same port, such as `retrying`, receives it through its constructor, typed as the port. Within `src`, only the factory package of the same module imports an adapter.
- An adapter's package organises its contents however it wants. Whatever the adapter's constructor takes — its own ports, and what satisfies them — and any error it determines is defined inside the adapter's package.
- Two adapters share no code, even when they reach the same resource or differ in one detail. Each holds what it needs.

This improves **locality for maintainers** since the path of an adapter names the module it serves, and everything one adapter knows is in one package.

This improves **flexibility for designers** since adding a capability to a module is adding one package beside the existing adapters.

### 5. Factories live under the module they create

A module's factories live in `infrastructure/factories/<module_name>`, where `<module_name>` matches the module's package in the core. They never live in the core: choosing an adapter means importing it, and the core imports nothing from the infrastructure.

A factory package imports its module's constructor and interface from the module's export file, that module's adapters from their export files, and other modules' factories from their factory packages. What a factory may do is covered by [dependency-injection](../dependency-injection/SKILL.md).

This improves **locality for maintainers** since the choice of adapters for a module is made in one package, and its path names the module.

This improves **testability for everyone** since a test can skip the factories, call the constructor from the core and pass test adapters.

### 6. Entrypoints drive interfaces

An entrypoint lives in `infrastructure/entrypoints`. Its constructor takes the entities it drives, typed as their modules' interfaces: those interfaces are its ports. It calls them, and translates between the outside world and data carriers. It never calls a module's constructor or a factory, and never imports an adapter.

The startup code beside it acquires what comes from the environment, calls the factories, passes the entities to the entrypoint's constructor and runs it. It holds no behaviour.

This improves **locality for maintainers** since a change to how a module is wired never touches an entrypoint.

This improves **testability for everyone** since the entrypoint's translation is a behaviour that a unit test can measure by passing it a test adapter for each interface it drives.

### 7. A data carrier belongs to the module whose interface or port it appears in

A data carrier that appears in a module's interface or port is defined in that module's data file and exported with it. A second module that needs the same data declares its own data carrier in its own data file, and the adapter that connects the two translates between them.

A data carrier goes in the domain package only when it is a value type that belongs to no one module and means the same thing in every module that uses it: `Currency`, `CustomerId`. A data carrier is never moved into the domain package because a second module needs it.

A type used only inside one module's implementation stays in that module's implementation and is not exported.

This improves **locality for maintainers** since a change to a module's data reaches only that module, its callers and its adapters, and never another module.

### 8. Name a package for what it is, not for being shared

There is no `utils`, `common`, `shared` or `helpers` package. A helper used by one package stays in that package. Duplicate a small helper until its second or third use, then decide:

- If it meets one of the reasons in codebase-design's "When to make a module", it is a module: give it an interface, a port if it needs one, and a package named for what it does (`retry`). Other modules reach it through their port and an adapter.
- Otherwise the duplication was the right call.

A helper never goes in the domain package: it carries behaviour, and the domain package holds data carriers only.

This improves **leverage for callers** since a package's name says what it offers.

This improves **locality for maintainers** since nothing accumulates in a place with no owner.

### 9. Tests mirror the source tree

Tests live in `tests`, outside `src`, split at the top by kind of test: `unit`, `integration` and `e2e`.

- **A test mirrors the path of the entity under test.** The unit tests for `src/core/<module_name>` are in `tests/unit/core/<module_name>`. The integration tests for an adapter are at the same mirrored path under `tests/integration`. End-to-end tests, where a project has them, are named for the entrypoint they run.
- **A test adapter lives beside the tests of the entity whose port it satisfies.** It is never under `src`, and never in a shared `fakes` package.
- **A test adapter needed by the tests of a second entity is duplicated, not shared.** Each entity's tests own their test adapters, as each module owns its port.
- **An adapter that is also used in production is not a test adapter.** An in-memory adapter used for local development lives in `infrastructure/adapters/<module_name>/in_memory`, and tests import it from there.

This improves **locality for maintainers** since the tests and test adapters for an entity are found from its path alone.

This improves **testability for everyone** since the unit tests can be run on their own by path, with no environment.

### 10. Legacy code is reached through ports, and moved only when asked

In a codebase that does not follow this structure yet, new code follows it and legacy code stays where it is.

- **A new module reaches legacy code through its port.** An adapter in `infrastructure/adapters/<module_name>/<legacy_name>` satisfies the port by calling the legacy code. The module's unit tests use a test adapter in its place, as for any port.
- **Legacy code reaches a new module the way startup code does.** It calls one of the module's factories and drives the entity through its interface. It never calls the module's constructor.
- **Legacy code is moved into the structure only when asked.** A task that touches legacy code changes it where it is.

This improves **locality for maintainers** since every crossing between legacy code and the structure is an adapter or a factory call, so moving a piece of legacy code later changes one place.

## Language idioms

The rules above are language-agnostic. When implementing, read the file for the target language:

- [python.md](python.md) — packages with `__init__.py` as the export file, a worked `OrderProcessor` module with its adapters, its factory package, an entrypoint and its tests
- [typescript.md](typescript.md) — directories with `index.ts` as the export file, a worked `OrderProcessor` module with its adapters, its factory package, an entrypoint and its tests
