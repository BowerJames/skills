---
name: project-structure
description: Policy for where code lives on disk. Use when you need to create new modules, adapters, factories or deciding where code should belong.
---

# Prerequisites

- **[codebase-design](../codebase-design/SKILL.md)** — read it first. This skill uses its vocabulary (entity, module, interface, implementation, constructor, port, adapter, factory, resource, behaviour) exactly.

# Project Structure

Project structure is the mapping of entities onto files. The structure is hexagonal: modules sit in the core and know nothing about the resources they use, and everything that reaches a resource sits in the infrastructure around them.

```
src/
  core/
    domain/                          # data carriers
    <module_name>/
      interface                      # the module's interface
      implementation                 # the module's implementation and constructor, in one file or many
      ports                          # the module's ports
  infrastructure/
    adapters/
      <module_name>/
        <adapter_name>/              # an adapter for one of that module's ports
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

_Avoid_: folder (says nothing about ownership), module (already means an interface, an implementation and its ports, not a place on disk).

**Export file** — the one file in a package that other packages import from.

_Avoid_: public API (interface already names what callers must know), barrel (names the mechanism, not the role), entry point (too close to entrypoint).

**Data carrier** — a type that holds data and its invariants and reaches no resource: `Order`, `Receipt`, `Money`. It has no implementation behind it, so modules take and return it as its concrete type, not through an interface of its own.

_Avoid_: model (overloaded with ORMs and machine learning), DTO, entity (already means any code construct).

**Domain package** — the `core/domain` package, holding the data carriers. One file per concept, named for the concept. `domain` is a reserved name: no module is called `domain`.

_Avoid_: utils, common, shared, helpers (names that say only that something is shared, so they accept anything).

**Entrypoint** — code that receives input from outside the program and drives a module through its interface: an HTTP handler, a CLI command, a queue consumer. It is not an adapter: it satisfies no port.

_Avoid_: controller, inbound adapter, driving adapter.

**Factory package** — the `infrastructure/factories/<module_name>` package, holding the factories of one module: the code that chooses adapters and passes them to the module's constructor.

_Avoid_: container, composition root, bootstrap.

## Rules

### 1. Imports point inward

- The domain package imports nothing else in the project.
- A module's package imports only from the domain package. It never imports another module's package and never imports from the infrastructure.
- The infrastructure imports from the core. The core never imports from the infrastructure.

This improves **locality for maintainers** since a change to an adapter, an entrypoint or another module cannot reach into a module's package.

This improves **testability for everyone** since a module can be constructed with nothing but its own package, the domain package and test adapters.

### 2. The core holds modules and data carriers, nothing else

- **The domain package holds data carriers only.** Anything that carries behaviours is not a data carrier. It is either a module in the core or an adapter in the infrastructure.
- **Every other package in the core is a module.** It has an interface, an implementation and ports. The core holds no adapters, no factories and no entrypoints.
- **A module never imports another module.** A module that needs another module declares a port for it. An adapter in `infrastructure/adapters/<module_name>` satisfies that port by calling the other module through its interface.
- **The core imports a third-party library only if the library reaches no resource internally.** A library that reaches a resource internally is imported only from the infrastructure, behind an adapter.

Whether behaviour that reaches no resource is a module in the core or an adapter in the infrastructure is a design decision. As a module it has its own interface and ports, and other modules reach it through a port and an adapter. As an adapter it exists only to satisfy one module's port.

This improves **locality for maintainers** since every behaviour in the core sits behind a module's interface, and every connection between two modules is one adapter.

This improves **testability for everyone** since no module needs another module, or a resource, to be constructed in a test.

### 3. One package per module, with the interface and ports in their own files

A module's package is `core/<module_name>`. It holds:

- **interface** — one file with the module's interface, and nothing else.
- **ports** — one file with every port the module's constructor requires.
- **implementation** — the module's implementation and its constructor. A package can organise its implementation however it wants: one file, or as many files and directories as it needs.
- **the export file** — exports the interface, the ports and the constructor. Nothing else in the implementation is exported.

Callers and entrypoints import the interface, adapters import the ports, and the module's factory package imports the constructor.

This improves **leverage for callers** since the interface file shows everything a caller must know and nothing else.

This improves **flexibility for designers** since the ports file lists every way the module's capability can be extended.

### 4. An adapter lives under the module whose port it satisfies

An adapter's package is `infrastructure/adapters/<module_name>/<adapter_name>`, where `<module_name>` matches the package in the core that defines the port.

An adapter is placed by its role, not by what it reaches: an entity defined to satisfy a module's port lives here whether or not it reaches a resource. It is named for the resource it reaches (`stripe`, `paypal`, `postgres`), or when it reaches none, for what sets it apart from the other adapters of that port (`in_memory`, `flat_rate`, `no_op`).

An adapter that connects one module to another is named for the module it calls: `infrastructure/adapters/checkout/order_processor`.

- An adapter imports the port it satisfies from the module's export file, and data carriers from the domain package.
- An adapter that connects one module to another also imports the other module's interface from its export file, and receives the entity through its own constructor.
- An adapter never imports another adapter. Only the factory package of the same module imports it.

This improves **locality for maintainers** since the path of an adapter names the module it serves, and everything that knows about one resource for one module is in one package.

This improves **flexibility for designers** since adding a capability to a module is adding one package beside the existing adapters.

### 5. Factories live under the module they create

A module's factories live in `infrastructure/factories/<module_name>`, where `<module_name>` matches the module's package in the core. They never live in the core: choosing an adapter means importing it, and the core imports nothing from the infrastructure.

- A factory package imports its module's constructor and interface from the module's export file, and that module's adapters from their export files.
- A factory declares the module's interface as its return type, never the implementation.
- A factory that needs an entity of another module calls a factory from that module's factory package and passes the entity to the adapter that connects the two. It never calls another module's constructor.

This improves **locality for maintainers** since the choice of adapters for a module is made in one package, and its path names the module.

This improves **testability for everyone** since a test can skip the factories, call the constructor from the core and pass test adapters.

### 6. Entrypoints drive interfaces

An entrypoint lives in `infrastructure/entrypoints`. It asks a factory for an entity, calls it through its interface, and translates between the outside world and data carriers. It never calls a constructor and never imports an adapter.

This improves **locality for maintainers** since a change to how a module is wired never touches an entrypoint.

### 7. Data carriers that cross an interface or a port live in the domain package

A data carrier that appears in a module's interface or in one of its ports is used by the module, its callers and its adapters, so it has no single owner. It is defined in the domain package from the start.

A type used only inside one module's implementation stays in that module's package and is not exported.

A module that reads only a slice of a data carrier may declare its own interface with just the fields it reads instead of importing the data carrier.

This improves **locality for maintainers** since each data carrier has exactly one definition, in a place every package is allowed to import from.

### 8. Name a package for what it is, not for being shared

There is no `utils`, `common`, `shared` or `helpers` package. A helper used by one package stays in that package. Duplicate a small helper until its second or third use, then decide:

- If it hides real complexity it is a module: give it an interface, ports and a package named for what it does (`retry`). Other modules reach it through a port and an adapter.
- Otherwise the duplication was the right call.

A helper never goes in the domain package: it carries behaviour, and the domain package holds data carriers only.

This improves **leverage for callers** since a package's name says what it offers.

This improves **locality for maintainers** since nothing accumulates in a place with no owner.

### 9. Tests mirror the source tree

Tests live in `tests`, outside `src`, split at the top by kind of test: `unit`, `integration` and `e2e` (see [testing](../testing/SKILL.md) for what each kind is).

- **A test mirrors the path of the entity under test.** The unit tests for `src/core/<module_name>` are in `tests/unit/core/<module_name>`. The integration tests for an adapter are at the same mirrored path under `tests/integration`. End-to-end tests are named for the entrypoint they run.
- **A test adapter lives beside the tests of the entity whose port it satisfies.** It is never under `src`, and never in a shared `fakes` package.
- **A test adapter needed by the tests of a second entity is duplicated, not shared.** Each entity's tests own their test adapters, as each module owns its ports.
- **An adapter that is also used in production is not a test adapter.** An in-memory adapter used for local development lives in `infrastructure/adapters/<module_name>/in_memory`, and tests import it from there.

This improves **locality for maintainers** since the tests and test adapters for an entity are found from its path alone.

This improves **testability for everyone** since the unit tests can be run on their own by path, with no environment.

## Language idioms

The rules above are language-agnostic. When implementing, read the file for the target language:

- [python.md](python.md) — packages with `__init__.py` as the export file, a worked `OrderProcessor` module with its adapters, its factory package, an entrypoint and its tests
- [typescript.md](typescript.md) — directories with `index.ts` as the export file, a worked `OrderProcessor` module with its adapters, its factory package, an entrypoint and its tests
