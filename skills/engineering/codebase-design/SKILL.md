---
name: codebase-design
description: The user's prescribed style for code that a human can find, understand and verify without having written it — immutable domain data, pure functions, self-contained components that take each dependency as one callable, wiring that connects them, and entrypoints at the framework edge. Use when you design, write, change, refactor or test code, decide where code goes, wire dependencies, or add a domain type, pure function, component, wiring or entrypoint.
---

# Codebase Design

## Purpose

Agents write code fast. Humans read code slowly. Humans own the intent of the code. Thus the human must understand code that the human did not write.

This skill makes code easy for a human to read and to keep reading as it changes. These are the goals:

1. **Find.** The reader can find the code for a given behaviour.
2. **Understand.** The reader knows what the code can do and what it cannot do, without reading the implementation.
3. **Estimate.** The reader can estimate the cost of a future change.
4. **Verify.** The reader can verify that a change does what the human intended.
5. **Small code.** The code that a reader must understand is small.
6. **Small change.** The change that a reader must verify is small and local.

The user prescribes the style in this skill. Thus the style in this skill is the style that the user knows. Use the structures, names and idioms in this skill and its files. Use each term with the meaning in the [glossary](GLOSSARY.md). Do not use a different structure or a new term when this skill gives one.

After a change, the code must look as if it was designed for the current intent from the start. Change existing code when the intent requires it. Remove code that the intent no longer requires.

Do refactors and behaviour changes as separate steps. A refactor does not change what the tests check. A behaviour change is small and local.

Get flexibility from structure. Do not write code for a change that nobody has asked for.

When a choice makes code easier to write but harder to read, choose the easier read.

## Files

- **[GLOSSARY](GLOSSARY.md)** — the meaning of each term in this skill. Read it before you use this skill.
- **[python/codebase-design](python/codebase-design.md)** — one way to follow these rules in Python: a worked example of each layer and a test, and the tools to recommend to the user. Read it when you write or change Python code.
- **[typescript/codebase-design](typescript/codebase-design.md)** — one way to follow these rules in TypeScript: a worked example of each layer and a test, and the tools to recommend to the user. Read it when you write or change TypeScript code.

## Layers

The code has five layers. Each layer has one job.

**Find:** The job of the code tells the reader its layer, and thus its folder.

| Layer | Job | Goals |
|---|---|---|
| `domain/` | Holds the immutable data that components use to communicate. | **Understand:** The domain shows all the data that goes between components. Data has no behaviour, so the reader can understand it without reading an implementation. |
| `pure_functions/` | Holds the pure functions that more than one component uses. | **Small code:** Components share logic and do not repeat it. **Verify:** A pure function gives the same output for the same input, so a test can verify it directly. |
| `component/` | Holds the behaviour. Each component is self-contained. | **Understand:** The reader can understand one component without reading other components. **Small change:** A change to one component stays in that component. **Verify:** A test gives each dependency a simple fake. |
| `wiring/` | Builds the components and connects them. Holds all the knowledge of how components work together. | **Find, Understand:** The reader can see the structure of the program in one place. **Small change:** A change to how components work together is a change to the wiring only. |
| `entrypoint/` | Holds the files that a process starts from. Converts between framework types and domain data. | **Small change:** When a framework changes, only the entrypoint changes. **Understand:** Other layers do not use framework types, so the reader does not need to know the framework to read them. |

Imports go in one direction:

`entrypoint/` → `wiring/` → `component/` → `pure_functions/` → `domain/` → libraries and frameworks

A layer can import from the layers to its right. A layer must not import from the layers to its left.

**Estimate:** A change to a layer can affect only the layers to its left. Thus the layer of a change tells the reader its cost. A change to the domain can affect all the code. A change to an entrypoint affects only that entrypoint.

A component must not import a different component.

**Understand:** The reader can read one component alone. **Small change:** A change to one component cannot break a different component, and the wiring can replace one component without a change to the others.

## Domain

The domain holds the data that components use to communicate.

- The domain holds only immutable data. It holds no behaviour.
  **Understand:** The reader can read the domain without reading an implementation. **Verify:** A component cannot change data that a different component gave it. Thus a test of one component stays valid for all callers.

  Exception: A domain type can be another name for a data type of a library, when that data type is not immutable. Code must not change data of that type.
  **Small code:** The domain does not copy the data types of a library.
- Data that goes between components has a domain type.
  **Understand:** The domain shows all the data that can go between components. **Small change:** Components agree only on data. Thus a change to one component does not change a different component.

## Pure functions

- A function in `pure_functions/` must be pure.
  **Verify:** A test can verify a pure function directly, with no fakes. **Small change:** Components can share a pure function, because it holds no state that links them.
- Put a pure function in `pure_functions/` only when more than one component uses it. Until then, keep it private in its component.
  **Find:** The reader finds the function next to the code that uses it. **Small code:** `pure_functions/` holds only shared logic.

## Components

A component holds behaviour. Its constructor takes configuration and dependencies.

- Each dependency is one callable.
  **Understand:** The constructor shows each thing that the component uses, and nothing more. **Verify:** A test gives each dependency a simple fake. **Small change:** The wiring can give any callable of the correct type, for example a different component or a decorator.
- A component must not assume that two of its dependencies share state.
  **Understand:** The reader can understand each dependency alone. **Small change:** The wiring can get each dependency from a different source.
- Configuration changes the behaviour of a component without a change to its implementation. The types of the component must show each behaviour that its configuration can give.
  **Small change:** New behaviour can come from new configuration, not new code. **Understand:** The reader can see all the possible behaviours from the types, without reading the implementation.

## Wiring

The wiring builds the components and connects them. It holds all the knowledge of how components work together.

- The wiring chooses the dependencies and the configuration of each component.
  **Find, Understand:** The reader can see how the program fits together in one place. **Estimate:** The reader can see which components use a given component, and thus what a change to it affects.
- The wiring is a set of functions. To import the wiring does not build components.
  **Verify:** A test can import code without building the real program. **Small change:** Each entrypoint calls the wiring with its own settings.

## Entrypoints

A process starts from an entrypoint. The entrypoint calls the wiring. It can give the result to a framework.

- Only an entrypoint uses the types of a framework. The entrypoint converts these types to domain data, and converts domain data to these types.
  **Understand:** The reader does not need to know the framework to read the other layers. **Small change:** When a framework changes, only the entrypoint changes.
