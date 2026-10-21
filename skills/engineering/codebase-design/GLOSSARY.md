# Glossary

This file defines the terms in this skill. Use each term only with the meaning in this file.

**Behaviour.** What code does that a test can see: the values it returns, the errors it raises, and the calls it makes to its dependencies.

**Callable.** A value that code can call, for example a function or a bound method. The type of a callable shows the types of its arguments and the type of its return value.

**Component.** Code in `component/` that holds behaviour and that the wiring makes with a constructor. A component can have private code, for example data types and functions that only it uses.

**Configuration.** Data that a component gets from its constructor and that changes its behaviour without a change to its implementation. Configuration is data, not a callable.

**Constructor.** The function that makes a component. It takes the configuration and the dependencies of the component.

**Dependency.** A callable that a component gets from its constructor and calls to do its work. A library or a pure function that a component imports is not a dependency.

**Domain.** The data types in `domain/`. Data that goes between components has a domain type.

**Entrypoint.** A file in `entrypoint/` that a process starts from. The process can start it directly, or through a framework, for example a web server.

**Framework.** Code that the project did not write and that decides when to run code of the project. For example, a web server calls a route handler when a request comes in. One package can be a framework in one use and a library in a different use.

**Implementation.** The body of a function or a class: the code that makes its behaviour. The types of the function or the class are not part of its implementation.

**Library.** Code that the project did not write and that code of the project calls. Code of the project decides when a library runs. The standard library is a library.

**Pure function.** A function that gives the same output for the same input. It does no I/O. It does not read the clock, random values, the environment or global state. It does not change its inputs.

**Refactor.** A change to the structure of code that does not change its behaviour.

**Wiring.** The code in `wiring/`. The wiring makes the components and gives each component its dependencies and configuration.
