# Engineering

Skills I use daily for code work.

- **[codebase-design](./codebase-design/SKILL.md)**: Shared vocabulary for designing modules: small interfaces, one port declared in the constructor, errors propagated and never wrapped, testable through the constructor and interface.
- **[dependency-injection](./dependency-injection/SKILL.md)**: How entities get their dependencies: a constructor takes the module's port and configuration, never creating, fetching or reaching a resource; module and adapter factories return the interface, only wire, call one entity's constructor and get everything else from factories, and reach no resource.
- **[testing](./testing/SKILL.md)**: Testing entities through their constructors and interfaces: one behaviour per test, a test adapter for every port, independent expectations, unit tests by default.
- **[test-driven-development](./test-driven-development/SKILL.md)**: Driving development through tests: the red → green → refactor loop, one behaviour per cycle, the constructor, interface and port evolving with the tests.
- **[project-structure](./project-structure/SKILL.md)**: Where code lives on disk: self-contained modules in the core, data carriers and domain interfaces alone in the domain, driving and driven adapters, module and adapter factories and startup code in the infrastructure, imports pointing inward, tests mirroring the source tree, legacy code reached through ports.
