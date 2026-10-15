---
name: project-structure
description: Policy for where code lives on disk. Use when you need to create new modules, adapters, factories, startup code or tests, deciding where code should belong, or adding code to a codebase that does not follow this structure yet. Requires the codebase-design and testing skills.
---

# Prerequisites

- **[codebase-design](../codebase-design/SKILL.md)** — read it first. This skill uses its vocabulary (entity, module, interface, implementation, constructor, port, adapter, driving and driven adapter, entrypoint, domain interface, factory, data carrier, resource, behaviour) exactly.
- **[testing](../testing/SKILL.md)** — its vocabulary for the kinds of test and where they point: entity under test, test adapter, unit test, integration test, end-to-end test.

# Project Structure

Project structure is the mapping of entities onto files. The structure is hexagonal: modules sit in the core and know nothing about the resources they use or the entrypoints that start them, and everything that touches either sits in the infrastructure around them. On one side, driving adapters are called by an entrypoint and drive the modules; on the other, driven adapters are called by the modules and reach resources.

```
src/
  core/
    domain/                          # data carriers and domain interfaces that belong to no one module
    <module_name>/
      interface                      # the module's interface
      port                           # the module's port, if it has one
      data                           # the data carriers in its interface and port
      errors                         # the errors it determines, if any
      implementation                 # the module's implementation and constructor, in one file or many
  infrastructure/
    adapters/
      <module_name>/
        <adapter_name>/              # a driven adapter for that module's port
      <entrypoint_name>/
        <adapter_name>/              # a driving adapter for that entrypoint's port
      <interface_name>/
        <adapter_name>/              # an adapter of that domain interface
    factories/
      modules/
        <module_name>/               # the module factories of that module
      adapters/
        <module_name>/, <entrypoint_name>/ or <interface_name>/
          <adapter_name>/            # the adapter factories of that adapter
    startup/                         # the startup code, one file per entrypoint
tests/
  unit/                              # mirrors src/
  integration/                       # mirrors src/
  e2e/
```

## Glossary

**Core** — the `core` directory: every module and the domain package. Nothing in it reaches a resource.

**Infrastructure** — the `infrastructure` directory: the adapters, the factories and the startup code. Everything that reaches a resource or is called by an entrypoint lives here, though not everything here does either.

**Package** — the directory that holds the code of one module, one adapter, or the factories of one module or one adapter. One module, one package; one adapter, one package; one entity's factories, one package.

_Avoid_: folder (says nothing about ownership), module (already means a package in the core with an interface, an implementation and a port, not any place on disk).

**Export file** — the one file in a package that other packages import from.

_Avoid_: public API (interface already names what callers must know), barrel (names the mechanism, not the role), entry point (too close to entrypoint).

**Domain package** — the `core/domain` package, holding data carriers and domain interfaces and nothing else: the ones that belong to no one module and mean the same thing in every entity that uses them, such as `Currency`, `CustomerId` and `AgentTool`. Nothing with behaviour goes in it: an interface has none, only a shape. One file per concept, named for the concept. `domain` is a reserved name: no module is called `domain`.

_Avoid_: utils, common, shared, helpers (names that say only that something is shared, so they accept anything).

**Entrypoint name** — the name of the package under `infrastructure/adapters` that holds the driving adapters of one entrypoint, named for the framework or runtime: `fastapi`, `click`, `celery`, or `main` for the process's `main` where there is no framework.

**Interface name** — the name of the package under `infrastructure/adapters` that holds the adapters of one domain interface, named for the interface's file in the domain package: `agent_tool` for `core/domain/agent_tool`.

Module, entrypoint and interface names share the `infrastructure/adapters` directory, so no name is used twice.

**Startup code** — the code that starts the program: it acquires what comes from the environment, calls the factories, registers the driving adapters with the entrypoint, and starts it. It lives in `infrastructure/startup`, one file per entrypoint, and holds no behaviour.

_Avoid_: container, composition root, bootstrap.

**Module factory package** — the `infrastructure/factories/modules/<module_name>` package, holding the module factories of one module: the code that gets adapters from their factories and passes them to the module's constructor.

**Adapter factory package** — the `infrastructure/factories/adapters/<module_name>/<adapter_name>` package, or `<entrypoint_name>/<adapter_name>` for a driving adapter, or `<interface_name>/<adapter_name>` for an adapter of a domain interface, holding the adapter factories of one adapter: the code that builds the adapter's parts and passes them to the adapter's constructor.

_Avoid_: container, composition root, bootstrap.

**Legacy code** — code in the project that does not follow this structure yet.

## Rules

### 1. Imports point inward

- The domain package imports nothing from outside itself: not the rest of the project, not a third-party library. A domain interface may use the domain's data carriers.
- A module's package imports only from the domain package. It never imports another module's package, never imports from the infrastructure, and never imports a third-party library.
- The infrastructure imports from the core. The core never imports from the infrastructure.

This improves **locality for maintainers** since a change to an adapter, the startup code, a framework or another module cannot reach into a module's package.

This improves **testability for everyone** since a module can be constructed with nothing but its own package, the domain package and a test adapter.

### 2. The core holds modules, data carriers and domain interfaces, nothing else

- **The domain package holds data carriers and domain interfaces only**, and only those that belong to no one module. Nothing with behaviour goes in it: anything that carries behaviour is either a module in the core or an adapter in the infrastructure.
- **Every other package in the core is a module.** It has an interface, an implementation and at most one port. The core holds no adapters, no factories and no startup code, and knows nothing of the entrypoints.
- **A module never imports another module.** A module that needs another module adds what it needs to its own port. An adapter in `infrastructure/adapters/<module_name>` satisfies the port by calling the other module through its interface.
- **A module is self-contained.** It depends only on the data carriers and interfaces of the domain package and on its own port. A third-party library is imported only from the infrastructure, and a module reaches what it provides through its port. The language's built-ins and the parts of its standard library that reach no resource are part of the language, not dependencies.

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

- **The interface, the data carriers and the errors** may be imported by anything in the infrastructure: driving adapters, factories, startup code, and adapters that connect another module to this one.
- **The port** is imported by the adapters that satisfy it, and by test adapters.
- **The constructor** is imported only by the module's module factory package and by the module's unit tests. An adapter or startup code never imports it.

This improves **leverage for callers** since the interface, data and errors files show everything a caller must know and nothing else.

This improves **flexibility for designers** since the port file shows everything an adapter must provide to extend the module's capability.

### 4. An adapter lives under the owner of the interface it satisfies

| The interface it satisfies | Its package |
|---|---|
| a module's port | `infrastructure/adapters/<module_name>/<adapter_name>`, where `<module_name>` matches the package in the core that defines the port |
| an entrypoint's port | `infrastructure/adapters/<entrypoint_name>/<adapter_name>`; rule 6 covers what a driving adapter does |
| a domain interface | `infrastructure/adapters/<interface_name>/<adapter_name>`, where `<interface_name>` matches the interface's file in the domain package |
| another adapter's own port | inside that adapter's package, as one of its parts |

An adapter satisfies the whole interface. It is placed by its role, not by what it reaches: an entity defined to satisfy an interface lives under that interface's owner whether or not it reaches a resource. A driven adapter, or an adapter of a domain interface, is named for what sets it apart from the other adapters of that interface: the resource it reaches (`stripe`, `paypal`, `postgres`), or what it does instead (`in_memory`, `flat_rate`, `retrying`, `structured`). A driving adapter is named for the input it handles: `cli`, `orders_router`.

An adapter that connects one module to another is named for the module it calls: `infrastructure/adapters/checkout/order_processor`. It translates between the data carriers of the two modules.

- A driven adapter imports the port it satisfies, and that module's data carriers, from the module's export file, and data carriers from the domain package.
- A driving adapter imports what its framework needs from the framework, and the interfaces and data carriers of the modules it drives from their export files.
- An adapter of a domain interface imports the interface, and any data carriers it uses, from the domain package.
- An adapter that connects one module to another also imports the other module's interface and data carriers from its export file, and receives the entity through its own constructor.
- An adapter never imports another adapter. Adapters are combined at runtime: a factory passes one to another's constructor, typed as the interface it satisfies. An adapter that wraps another adapter of the same port, such as `retrying`, receives it that way, and so does an adapter that offers a module a set of adapters of a domain interface.
- An adapter's export file exports its constructor, its own ports, and the parts that satisfy them in production. Within `src`, only the adapter's own adapter factory package imports them. Its unit tests import its constructor and its own ports.
- An adapter's package organises its contents however it wants. Whatever the adapter's constructor takes — its own ports, and what satisfies them — and any error it determines is defined inside the adapter's package. The exceptions are a module's interface and a domain interface: they are defined where they belong, and what satisfies them is passed in by a factory.
- Two adapters generally hold their own code, even when they reach the same resource or differ in one detail. Code shared between adapters is something to watch for: rule 8 covers what it usually means.

This improves **locality for maintainers** since the path of an adapter names the module, entrypoint or domain interface it serves, and everything one adapter knows is in one package.

This improves **flexibility for designers** since adding a capability to a module is adding one package beside the existing adapters.

### 5. Factories live in the factories directory, under what they build

Every factory lives in `infrastructure/factories`, split at the top by what it builds:

- **Module factories** live in `infrastructure/factories/modules/<module_name>`, where `<module_name>` matches the module's package in the core.
- **Adapter factories** live in `infrastructure/factories/adapters/<module_name>/<adapter_name>`, or `<entrypoint_name>/<adapter_name>` for a driving adapter, or `<interface_name>/<adapter_name>` for an adapter of a domain interface: the same path as the adapter's package under `infrastructure/adapters`. Every adapter with a package of its own has an adapter factory package, driving adapters included. A part, an adapter of another adapter's own port, is built by the factory of the adapter it belongs to.

No factory lives in the core, because building an entity means importing what it is built from, and the core imports nothing from the infrastructure. No factory lives in an adapter's package, so the factories directory is the one place where the program is wired.

- A module factory package imports its module's constructor and interface from the module's export file, adapter factories from adapter factory packages, and other modules' factories from their module factory packages. It never imports from an adapter's package.
- An adapter factory package imports its adapter's constructor and parts from the adapter's export file, and anything else the adapter takes from other factory packages.

What a factory may do is covered by [dependency-injection](../dependency-injection/SKILL.md).

This improves **locality for maintainers** since every factory is found in one directory, and the path of each one names what it builds.

This improves **testability for everyone** since a test can skip the factories, call a constructor and pass test adapters.

### 6. Driving adapters drive interfaces

The entrypoint is not the project's code: it is the framework or runtime that receives input from outside the program and calls into it. What it calls is a driving adapter, in `infrastructure/adapters/<entrypoint_name>/<adapter_name>`.

- **A driving adapter satisfies its entrypoint's port**: it has the shape the framework calls, such as a route or a command.
- **It drives modules through their interfaces**, which its constructor takes as its own ports. It translates between the entrypoint's input and the modules' data carriers.
- **It never calls a module's constructor or a factory, and never imports another adapter.**
- **When the entity it drives must be built per request**, because it holds that request's resources, such as the caller's socket, the driving adapter takes the opening of it as its own port: a callable that returns the module's interface. The startup code satisfies that port with a module factory.

The startup code, in `infrastructure/startup`, acquires what comes from the environment, calls the factories, registers the driving adapters with the entrypoint and starts it. It holds no behaviour.

This improves **locality for maintainers** since a change to how a module is wired never touches a driving adapter, and a change of framework touches only the driving adapters and the startup code.

This improves **testability for everyone** since a driving adapter's translation is a behaviour that a unit test can measure by passing it a test adapter for each interface it drives.

### 7. A data carrier or interface belongs to the module whose interface or port it appears in

A data carrier that appears in a module's interface or port is defined in that module's data file and exported with it. A second module that needs the same data declares its own data carrier in its own data file, and the adapter that connects the two translates between them.

A data carrier goes in the domain package only when it belongs to no one module and means the same thing in every module that uses it: `Currency`, `CustomerId`. A data carrier is never moved into the domain package because a second module needs it.

The same test holds for an interface. A domain interface, such as `AgentTool`, belongs to no one module and means the same thing to every entity that uses it: a module's port may use it, and its implementations are adapters under `infrastructure/adapters/<interface_name>`. An interface that only one entity needs stays where it is declared: a module's port in the module's port file, an adapter's own port in the adapter's package.

A type used only inside one module's implementation stays in that module's implementation and is not exported.

This improves **locality for maintainers** since a change to a module's data reaches only that module, its callers and its adapters, and never another module. A change to a domain data carrier or interface reaches every entity that uses it, which is why only what means the same everywhere goes there.

### 8. Name a package for what it is, not for being shared

There is no `utils`, `common`, `shared` or `helpers` package. A helper used by one package stays in that package. Duplicate a small helper until its second or third use, then decide:

- If it meets one of the reasons in codebase-design's "When to make a module", it is a module: give it an interface, a port if it needs one, and a package named for what it does (`retry`). Other modules reach it through their port and an adapter.
- Otherwise the duplication was the right call.

A helper never goes in the domain package: it carries behaviour, and the domain package holds data carriers and domain interfaces only.

**Watch for code shared between adapters.** It is not forbidden, but it is usually a sign that a capability wants a home of its own, so that it can be passed to constructors instead of copied or imported. Before sharing, ask what the code is:

- **Behaviour, including the translation of a provider's format**, such as a client that builds a provider's requests, parses its responses and maps its errors, is usually a module, with a port for any resource it reaches (`tpfg_proxy`, with an HTTP port its adapter satisfies). Its consumers receive it through their constructors.
- **Glue around a third-party library** usually belongs in the adapter that uses it, or in a generic adapter of a domain interface that others wrap, such as a tool built from a schema.
- **Something too small for either**, a line of formatting or a constant, is often best left as two copies, especially when they are likely to change apart.

When none of these fits, sharing is a judgement call: name the shared package for what it does, and never make one adapter import another.

This improves **leverage for callers** since a package's name says what it offers.

This improves **locality for maintainers** since nothing accumulates in a place with no owner.

### 9. Tests mirror the source tree

Tests live in `tests`, outside `src`, split at the top by kind of test: `unit`, `integration` and `e2e`.

- **A test mirrors the path of the entity under test.** The unit tests for `src/core/<module_name>` are in `tests/unit/core/<module_name>`. The integration tests for an adapter are at the same mirrored path under `tests/integration`. End-to-end tests, where a project has them, are named for the startup code they run.
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

- [python.md](python.md) — packages with `__init__.py` as the export file, a worked `OrderProcessor` module with its adapters, its module and adapter factory packages, a driving adapter, its startup code, a domain interface with its adapters, and the tests
- [typescript.md](typescript.md) — directories with `index.ts` as the export file, a worked `OrderProcessor` module with its adapters, its module and adapter factory packages, a driving adapter, its startup code, a domain interface with its adapters, and the tests
